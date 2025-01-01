# 常见实体的召唤

本节将分类说明各种常见的实体的召唤方式。

*注：下文中把实体类中**除以`EntityType`和`Level`为参数的构造方法外的所有构造方法**称为“**特殊构造方法**”。**注意如果重载了任一不含`EntityType`参数的构造方法，或者继承了所有的构造方法都不含`EntityType`参数的实体类，则应该重写`getType`、`getDimensions`、`causeFallDamage`等一切使用到`Entity`类中的成员常量`type`的方法。**这是因为如果重载/继承会导致实体的`type`与实体的真实`EntityType`不一致，需要特殊处理所有用到`type`的地方以防止实体出现意外的问题。*

### 框架

#### 流程

实体的召唤分为**实例化**、**预处理**和**正式添加**3个部分（后两部分顺序可以互换，甚至可以把“正式添加”部分夹在“预处理”中）。

假设有一种实体名叫`Polonium`，其对应的实体类型（`EntityType`）为`POLONIUM`。

##### 实例化
这是实例化部分的代码示例：
```java
Polonium polonium = new Polonium(...);
```
这一部分实例化了一个新的实体类的对象，它代表着未来将要添加到世界中的实体。

##### 预处理
这是预处理部分的代码示例：
```java
// 把实体移动到(10, 64, -25)处
polonium.moveTo(10.0, 64.0, -25.0);
// 为Mob（非玩家生物）的最终生成做最后的调整
ForgeEventFactory.onFinalizeSpawn(polonium, level, level.getCurrentDifficultyAt(polonium.blockPosition()), MobSpawnType.MOB_SUMMONED, null, null);
```
这一部分不是所有实体都需要的，对于许多非生物实体（如弹射物），往往调用它们的特殊构造方法实例化新对象时就进行了预处理，所以无需额外的预处理部分。

##### 正式添加
这是正式添加部分的代码示例：
```java
level.addFreshEntity(polonium);
```
用`addFreshEntity`把实体添加到世界中时，需要注意实体UUID的唯一性（在分析雪傀儡投掷雪球的方式时提到过这一点，具体的代码举例可以看[1.2.2.2.1的相关内容](https://lych.top/polonium/part1-monster/ranged/snow_golem/snow_golem.html)）。

#### `setPowRaw`、`setPos`、`moveTo`、`teleportTo`和`randomTeleport`的区别
在召唤实体的过程中，几乎所有情况下都需要改变召唤的实体的位置。小标题中的5个方法都是更改实体位置的方法，那么它们之间有什么区别呢？

##### `setPowRaw`
`setPosRaw`是最底层的，用于改变且仅改变实体的坐标。在实体的坐标中，`position`表示实体碰撞箱**底面中心**的坐标，而`blockPosition`和`chunkPosition`表示`position`所在的方块坐标和区块坐标。

```java
public final void setPosRaw(double x, double y, double z) {
    // 更新position, blockPosition和chunkPosition
    // position改变是blockPosition改变的必要不充分条件，而blockPosition改变是chunkPosition改变的必要不充分条件，因此从position开始依次更新坐标。
    // 这几层逻辑关系也很容易理解，因为从position到chunkPosition是由“精确”位置到“粗略”位置的变化。
    if (position.x != x || position.y != y || position.z != z) {
        position = new Vec3(x, y, z);
        int x1 = Mth.floor(x);
        int y1 = Mth.floor(y);
        int z1 = Mth.floor(z);
        if (x1 != blockPosition.getX() || y1 != blockPosition.getY() || z1 != blockPosition.getZ()) {
            blockPosition = new BlockPos(x1, y1, z1);
            feetBlockState = null;
            if (SectionPos.blockToSectionCoord(x1) != chunkPosition.x || SectionPos.blockToSectionCoord(z1) != chunkPosition.z) {
                chunkPosition = new ChunkPos(blockPosition);
            }
        }
        // 这里用于更新实体的section（section是世界中16x16x16的区域）
        levelCallback.onMove();
    }
    if (isAddedToWorld() && !level.isClientSide && !isRemoved()) {
        level.getChunk((int) Math.floor(x) >> 4, (int) Math.floor(z) >> 4); // Forge - ensure target chunk is loaded.
    }
}
```
但实际应用中，往往还需要改变实体的碰撞箱、朝向等，因此有了后面的4个方法。

##### `setPos`
`setPos`除了调用了`setPosRaw`外，`setPos`还为实体重新设置了碰撞箱。

```java
public void setPos(double x, double y, double z) {
    setPosRaw(x, y, z);
    setBoundingBox(makeBoundingBox());
}
```

##### `moveTo`
`moveTo`在`setPos`的基础上还更新了实体的朝向和一系列旧坐标值（把旧坐标值改为与新坐标值一致）。更改旧坐标值是为了防止旧坐标值与新坐标值差异过大造成渲染问题。
```java
public void moveTo(double x, double y, double z, float yRot, float xRot) {
    setPosRaw(x, y, z);
    setYRot(yRot);
    setXRot(xRot);
    setOldPosAndRot(); // 更新实体的旧坐标和旧的旋转角度，旧坐标和旧的旋转角度常用于平滑实体的渲染
    reapplyPosition();
}

protected void reapplyPosition() {
    setPos(position.x, position.y, position.z);
}
```
`moveTo`还提供了许多方便的重载方法，以更方便地使用。

##### `teleportTo`
`teleportTo`仅在服务端执行，调用时会一并对实体的骑乘者（passenger/rider）执行`moveTo`的操作。
```java
public void teleportTo(double x, double y, double z) {
    if (level() instanceof ServerLevel) {
        moveTo(x, y, z, getYRot(), getXRot());
        teleportPassengers();
    }
}

private void teleportPassengers() {
    getSelfAndPassengers().forEach(entity -> {
        for (Entity entity : entity.passengers) {
            entity.positionRider(entity, Entity::moveTo);
        }
    });
}
```

##### `randomTeleport`
`randomTeleport`是`LivingEntity`中的方法，与`teleportTo`的区别在于对传送点的y坐标进行了调整，以确保生物落在固体方块上。注意**这个方法中没有任何随机生成坐标的过程**，因此传送点需要自己提前随机生成好。  

`randomTeleport`有返回值，返回`true`则说明找到了合适的目标位置。  

*注：[1.2.1.3.2](https://lych.top/polonium/part1-monster/melee/enderman/enderman.html)中也提到了`randomTeleport`，其中说到笔者认为在末影人瞬移的情境下调整y坐标的地方可以省略，但是如果单独调用此方法而不像末影人一样在`randomTeleport`前就确定好了调整过的传送点，那么该方法内调整y坐标的地方则不可缺少*
```java
public boolean randomTeleport(double randomX, double randomY, double randomZ, boolean showParticles) {
    double x = getX();
    double y = getY();
    double z = getZ();
    double finalY = randomY;

    boolean success = false;
    BlockPos targetPos = BlockPos.containing(randomX, randomY, randomZ);
    Level level = level();

    if (level.hasChunkAt(targetPos)) {
        boolean foundSolid = false;
        while (!foundSolid && targetPos.getY() > level.getMinBuildHeight()) {
            BlockPos below = targetPos.below();
            BlockState belowBlockState = level.getBlockState(below);

            if (belowBlockState.blocksMotion()) {
                foundSolid = true;
            } else {
                --finalY;
                targetPos = below;
            }
        }
        if (foundSolid) {
            teleportTo(randomX, finalY, randomZ);

            if (level.noCollision(this) && !level.containsAnyLiquid(getBoundingBox())) {
                success = true;
            }
        }
    }

    if (!success) {
        teleportTo(x, y, z);
        return false;
    } else {
        if (showParticles) {
//          广播46号实体事件会生成大量传送粒子效果
            level.broadcastEntityEvent(this, (byte) 46);
        }
        if (this instanceof PathfinderMob) {
            ((PathfinderMob) this).getNavigation().stop();
        }
        return true;
    }
}
```

##### 总结
一般召唤实体的过程中，对生物用`moveTo`，而对非生物用`setPos`调整位置就行，当然有特殊需求的话也可以采取别的方式。还有一些其他的更改实体位置的方法用处相对较少，这儿就暂时略去不说了。

### 非生物的召唤

非生物实体的召唤相对简单，许多非生物实体都有特殊构造方法，以方便我们实例化该实体。  

#### `ThrowableProjectile`与`AbstractArrow`（大多数受重力影响的弹射物）

`ThrowableProjectile`的实现类通常有两个特殊构造方法。其中一个构造方法含有1个`LivingEntity`参数，另一个含有3个`double`参数。  

`ThrowableProjectile`类：
```java
protected ThrowableProjectile(EntityType<? extends ThrowableProjectile> type, double x, double y, double z, Level level) {
    this(type, level);
    setPos(x, y, z);
}

protected ThrowableProjectile(EntityType<? extends ThrowableProjectile> type, LivingEntity owner, Level level) {
    this(type, owner.getX(), owner.getEyeY() - (double) 0.1F, owner.getZ(), level);
    setOwner(owner);
}
```

对于含有`LivingEntity`参数的构造方法，调用后会将弹射物的所有者设置为传入的生物，并把弹射物移动到此生物**眼睛坐标下0.1格**处。
```java
ThrowableProjectile throwableProjectile = new ThrowableProjectileImpl(level, owner);
throwableProjectile.shoot(dx, dy, dz, scale, deviation);
level.addFreshEntity(throwableProjectile);
```

而对于含有3个`double`参数的构造方法，调用后会把弹射物移动到这3个`double`参数组成的坐标上。
```java
ThrowableProjectile throwableProjectile = new ThrowableProjectileImpl(level, x, y, z);
throwableProjectile.shoot(dx, dy, dz, scale, deviation);
level.addFreshEntity(throwableProjectile);
```
一般`ThrowableProjectile`由某一特定生物射出时用含有`LivingEntity`参数的构造方法来实例化（`LivingEntity`参数传入射出弹射物的生物），而不由生物射出的弹射物则用含有3个`double`参数的构造方法来实例化。  

所有的箭（`AbstractArrow`）虽然都没有继承`ThrowableProjectile`，但是箭的召唤方式与`ThrowableProjectile`几乎完全一样。不过箭有很多变种，因此还需要注意往往要对某些生物发射出的箭进行特殊处理。  

举骷髅射箭为例（下文中删去了坐标计算、音效播放部分的代码。另见[1.2.2.5.1](https://lych.top/polonium/part1-monster/ranged/skeleton/skeleton.html)）。其中`getProjectile`、`getArrow`和`customArrow`就是这样的特殊处理。
```java
ItemStack projectile = getProjectile(getItemInHand(ProjectileUtil.getWeaponHoldingHand(this, item -> item instanceof BowItem)));
AbstractArrow arrow = getArrow(projectile, power);
if (getMainHandItem().getItem() instanceof BowItem) {
    arrow = ((BowItem) getMainHandItem().getItem()).customArrow(arrow);
}
arrow.shoot(dx, dy + distance * (double) 0.2F, dz, 1.6F, (float) (14 - level().getDifficulty().getId() * 4));
level().addFreshEntity(arrow);
```

#### `AbstractHurtingProjectile`（大多数不受重力影响的弹射物）

`AbstractHurtingProjectile`的实现类通常也有两个特殊构造方法。其中一个构造方法含有1个`LivingEntity`参数和3个`double`参数，另一个含有6个`double`参数。  

`AbstractHurtingProjectile`类：
```java
public AbstractHurtingProjectile(EntityType<? extends AbstractHurtingProjectile> type, double x, double y, double z, double targetX, double targetY, double targetZ, Level level) {
    this(type, level);
    moveTo(x, y, z, getYRot(), getXRot());
    reapplyPosition();
    double targetDistance = Math.sqrt(targetX * targetX + targetY * targetY + targetZ * targetZ);
    if (targetDistance != 0.0D) {
        this.xPower = targetX / targetDistance * 0.1D;
        this.yPower = targetY / targetDistance * 0.1D;
        this.zPower = targetZ / targetDistance * 0.1D;
    }
}

public AbstractHurtingProjectile(EntityType<? extends AbstractHurtingProjectile> type, LivingEntity owner, double targetX, double targetY, double targetZ, Level level) {
    this(type, owner.getX(), owner.getY(), owner.getZ(), targetX, targetY, targetZ, level);
    setOwner(owner);
    setRot(owner.getYRot(), owner.getXRot());
}
```

对于含有`LivingEntity`参数和`double`参数的构造方法，调用后会将弹射物的所有者设置为传入的生物，把弹射物移动到此生物的**底面中心**处，并以传入的3个`double`参数组成的向量为弹射物的发射方向。
```java
AbstractHurtingProjectile hurtingProjectile = new AbstractHurtingProjectileImpl(level, owner, dx, dy, dz, level);
// 此处可以直接发射弹射物，也可以用setPos等方法调整弹射物的初始坐标
level.addFreshEntity(hurtingProjectile);
```

但由于通常生物不会从自身的底面中心处射出弹射物，所以一般会调整弹射物初始坐标。例如凋灵会将凋灵之首调整到正确的头部上再发射凋灵之首。  
```java
WitherSkull skull = new WitherSkull(level(), this, dx, dy, dz);
skull.setOwner(this);
if (dangerous) {
    skull.setDangerous(true);
}
// 此处使用了setPosRaw，而前面说过setPosRaw不会更新碰撞箱，因此会导致凋灵之首的坐标与碰撞箱位置不一致
// 不过由于弹射物每刻更新时都会通过setPos改变自身坐标，所以不会造成大的问题
// 个人认为这里可以改成setPos，而且改成setPos可能更好
skull.setPosRaw(headX, headY, headZ);
level().addFreshEntity(skull);
```

而对于含有6个`double`参数的构造方法，调用后会把弹射物移动到前3个`double`参数组成的坐标上，并以后3个`double`参数组成的向量为弹射物的发射方向。
```java
AbstractHurtingProjectile hurtingProjectile = new AbstractHurtingProjectileImpl(level, x, y, z, dx, dy, dz, level);
level.addFreshEntity(hurtingProjectile);
```

### 生物的召唤

召唤生物（`LivingEntity`）往往需要更复杂的预处理，同时大部分生物都没有特殊构造方法。

（WIP）