# 末影人的底层实现

末影人是比僵尸稍复杂的近战怪物。因为`EnderMan`类中有约40%的代码是继承了`Goal`的非公有静态内部类，也就是末影人独有的AI，内容较多，所以我们现在先不谈论它们。  
本节将以讨论末影人瞬移、搬运方块等能力与特性的实现为主。

先来看Fields：
```java
// 末影人追击时的速度提升的修饰符的UUID
private static final UUID SPEED_MODIFIER_ATTACKING_UUID = UUID.fromString("020E0DFB-87AE-4653-9556-831010E291A0");
// 末影人追击时的速度提升的修饰符
private static final AttributeModifier SPEED_MODIFIER_ATTACKING = new AttributeModifier(SPEED_MODIFIER_ATTACKING_UUID, "Attacking speed boost", (double)0.15F, AttributeModifier.Operation.ADDITION);
// Forge提供的“魔法值”的“翻译”
private static final int DELAY_BETWEEN_CREEPY_STARE_SOUND = 400;
private static final int MIN_DEAGGRESSION_TIME = 600;
// 末影人手持的方块
private static final EntityDataAccessor<Optional<BlockState>> DATA_CARRY_STATE = SynchedEntityData.defineId(EnderMan.class, EntityDataSerializers.OPTIONAL_BLOCK_STATE);
// 决定了末影人是否在“生气”（张嘴和发抖），值为true则说明在“生气”
private static final EntityDataAccessor<Boolean> DATA_CREEPY = SynchedEntityData.defineId(EnderMan.class, EntityDataSerializers.BOOLEAN);
// 决定了末影人是否在盯着实体看，值为true则说明是这样
private static final EntityDataAccessor<Boolean> DATA_STARED_AT = SynchedEntityData.defineId(EnderMan.class, EntityDataSerializers.BOOLEAN);
// 上次播放示威声的时间戳（tick），只在客户端有用
private int lastStareSound = Integer.MIN_VALUE;
// 改变目标的时间戳（tick）
private int targetChangeTime;
// 生气时间的范围
private static final UniformInt PERSISTENT_ANGER_TIME = TimeUtil.rangeOfSeconds(20, 39);
// 这些是上一节的内容（生气的剩余时间和目标UUID）
private int remainingPersistentAngerTime;
@Nullable
private UUID persistentAngerTarget;
```

各Field的用途已经在注释中标注出来了。除了1.2.1.1节讲的修饰符和`EntityDataAccessor`外，你需要注意这里时间戳（timestamp）的运用。时间戳可以减少更新值的次数，因此一般会在`Map<?, Integer>`或`CompoundTag`里存时间戳，而不是每刻数值减少1的“变量”。这里的时间戳的作用可以**近似**地理解为“标识了**这次打开游戏后自实体首次被tick以来经过的tick数**（前提是实体一直在被更新）”。因为实体的`tickCount`不会存到实体的NBT中，所以**在实体NBT中保存基于tickCount的时间戳无意义**。  

时间戳的运用也十分常见，`LivingEntity`里的`lastHurtByPlayerTime`、`lastHurtByMobTimestamp`以及`lastHurtMobTimestamp`等成员变量都是基于`tickCount`的时间戳。  

接下来是构造方法。这个构造方法成分复杂，所以我们来细细研究一下233。
```java
public EnderMan(EntityType<? extends EnderMan> type, Level level) {
    super(type, level);
    setMaxUpStep(1.0F);
    setPathfindingMalus(BlockPathTypes.WATER, -1.0F);
}
```
（`IForgeEntity`）
```java
default float getStepHeight() {
    float vanillaStep = self().maxUpStep();
    if (self() instanceof LivingEntity living) {
//      获取ForgeMod.STEP_HEIGHT_ADDITION.get()属性的值，其中ForgeMod.STEP_HEIGHT_ADDITION是个Attribute的RegistryObject
        AttributeInstance stepHeightAttribute = living.getAttribute(ForgeMod.STEP_HEIGHT_ADDITION.get());
        if (stepHeightAttribute != null) {
            return (float) Math.max(0, vanillaStep + stepHeightAttribute.getValue());
        }
    }
    return vanillaStep;
}
```
先是`setMaxUpStep(1.0F);`。  

`setMaxUpStep`方法设置了实体的`maxUpStep`。`maxUpStep`变量的值（默认为0.6）和Forge提供的`ForgeMod.STEP_HEIGHT_ADDITION.get()`属性，一同决定了实体不需要跳跃就能一下走上去的方块的最低高度，例如，当`getStepHeight`的返回值大于等于0.5时，实体能不跳跃走上半砖，返回值大于等于1（如铁傀儡）时就可以一步走上大多数方块。  

然后是`setPathfindingMalus(BlockPathTypes.WATER, -1.0F);`。  

MC中使用了一个**基于“可变堆内元素位置”的二叉堆**（为什么要这样设计而不使用现成的`PriorityQueue`呢？因为`Node`是可变的）的A\*寻路算法。如果你对A\*算法比较陌生，你可以看看[这篇教程](https://zhuanlan.zhihu.com/p/54510444)，或者暂时跳过下面一段内容，因为本节的重点并不是寻路算法。如果你想更深入地了解MC中的寻路系统，[这篇文章](https://www.bilibili.com/read/cv23090029/)或许对你有帮助。

`setPathfindingMalus`方法**间接地影响了NodeEvaluator（路径节点计算器）对符合BlockPathTypes.WATER类型的Node（可以理解为水上的路径节点）计算的costMalus的结果**，`Node`的`costMalus`会影响`Node`的g值。这里简要说一下第二个参数一般的取值方式（以下内容将类名`BlockPathTypes`译为“方块路径类型”）。
- 如果你的Mob**一定需要避免某一类方块**（例如对TA有严重的危险），就把那类方块对应的方块路径类型对应的malus设置为**-1**  
- 如果你的Mob**需要尽量避免某一类方块**（例如对TA有较低的危险），就把那类方块对应的方块路径类型对应的malus设置得**高一些**
- 如果你的Mob**需要尽量在某一类方块上行走**，就把那类方块对应的方块路径类型对应的malus设置得**低一些**（但**一定要非负**）  

其中常用的取值为**-1**，**0**，**8**，**16**（除负数外，值越高表示越需要避免这类方块）。举几个原版使用的例子。  

烈焰人：
```java
public Blaze(EntityType<? extends Blaze> type, Level level) {
    super(type, level);
    setPathfindingMalus(BlockPathTypes.WATER, -1.0F);
    setPathfindingMalus(BlockPathTypes.LAVA, 8.0F);
    setPathfindingMalus(BlockPathTypes.DANGER_FIRE, 0.0F);
    setPathfindingMalus(BlockPathTypes.DAMAGE_FIRE, 0.0F);
    xpReward = 10;
}
```
因为烈焰人不怕火却怕水（`isSensitiveToWater`），所以火焰（`DAMAGE_FIRE`和`DANGER_FIRE`）的malus都被设为了0，就连默认malus为-1，一般的生物都尽量避免的岩浆，`malus`也被设为了8，但水的malus却被设为了-1。  

水生生物：
```java
protected WaterAnimal(EntityType<? extends WaterAnimal> type, Level level) {
    super(type, level);
    setPathfindingMalus(BlockPathTypes.WATER, 0.0F);
}
```
水生生物离不开水，更不可能怕水，所以水的malus被设为了0。  

*提示：新版中BlockPathTypes实现了IExtensibleEnum接口，也就是说你可以实例化属于自己的方块路径类型。建议有需求的Modder重写IForgeBlock的getBlockPathType方法，以给方块自定义的路径类型。*

回到末影人，现在应该能理解第二个参数“-1”的含义了。因为末影人遇水会受到伤害，所以把水的malus设置为了-1。今后在写自己的`Mob`时，也可以通过这个方式调整TA的寻路系统，让TA变得更聪明或更难对付~  

AI部分这节先不讲。

然后是末影人的属性注册。
```java
public static AttributeSupplier.Builder createAttributes() {
    return Monster.createMonsterAttributes()
        .add(Attributes.MAX_HEALTH, 40.0D)
        .add(Attributes.MOVEMENT_SPEED, (double) 0.3F)
        .add(Attributes.ATTACK_DAMAGE, 7.0D)
        .add(Attributes.FOLLOW_RANGE, 64.0D);
}
```
属性注册应该理解起来没有什么难点。再接着是被大改的`setTarget`的方法。
```java
@Override
public void setTarget(@Nullable LivingEntity target) {
    AttributeInstance attr = getAttribute(Attributes.MOVEMENT_SPEED);
    if (target == null) {
         targetChangeTime = 0;
         entityData.set(DATA_CREEPY, false);
         entityData.set(DATA_STARED_AT, false);
         attr.removeModifier(SPEED_MODIFIER_ATTACKING);
    } else {
    //  时间戳的赋值
        targetChangeTime = tickCount;
        entityData.set(DATA_CREEPY, true);
        if (!attr.hasModifier(SPEED_MODIFIER_ATTACKING)) {
            attr.addTransientModifier(SPEED_MODIFIER_ATTACKING);
        }
    }
    //                         Forge的注释，意思就是把super.setTarget移下来可以允许事件监听器更改当前末影人DATA_CREEPY和DATA_STARED_AT的值
    super.setTarget(target); //Forge: Moved down to allow event handlers to write data manager values.
}
```
以及`Mob`类的`setTarget`。
```java
public void setTarget(@Nullable LivingEntity target) {
    LivingChangeTargetEvent changeTargetEvent = ForgeHooks.onLivingChangeTarget(this, target, LivingChangeTargetEvent.LivingTargetType.MOB_TARGET);
    if (!changeTargetEvent.isCanceled()) {
        this.target = changeTargetEvent.getNewTarget();
    }
}
```
首先要感谢Forge的一点是，新版的Forge大幅度优化了`LivingChangeTargetEvent`。以前这个事件既不能被取消，也不能改变target，而且如果使用`Brain`的Mob（例如猪灵）设置攻击的目标，这个事件甚至不会被`post`，可以说是几乎没什么用。现在这几个问题被彻底解决了！  

接着看`EnderMan`类里重写的一部分：如果`target`为null，就**重置时间戳targetChangeTime和末影人的几个状态**，并**移除速度修饰符**。否则**给targetChangeTime赋值当前的tickCount**，并**设置末影人为愤怒状态**，**添加速度修饰符**。这为我们设计生物提供了一个有用的思路：要想让Mob在设置目标时有特殊的行为，可以**重写setTarget方法**或者**监听事件LivingChangeTargetEvent**。  

另外要说的是，在`setTarget`的过程中，不难发现**移除属性修饰符不需要hasModifier的检查**，但**添加属性修饰符一定要检查以前有没有添加过**，不然会抛出`IllegalArgumentException`（*Modifier is already applied on this attribute!*）。

下面是示威声的播放。
```java
@Override
public void onSyncedDataUpdated(EntityDataAccessor<?> accessor) {
    if (DATA_CREEPY.equals(accessor) && hasBeenStaredAt() && level().isClientSide) {
        playStareSound();
    }
    super.onSyncedDataUpdated(accessor);
}

public void playStareSound() {
//  这里是时间戳的应用
    if (tickCount >= this.lastStareSound + 400) {
        this.lastStareSound = tickCount;
        if (!isSilent()) {
            level().playLocalSound(getX(), getEyeY(), getZ(), SoundEvents.ENDERMAN_STARE, getSoundSource(), 2.5F, 1.0F, false);
        }
    }
}
```
因为`lastStareSound`成员变量只在客户端被使用，所以无需数据同步。有一个注意点是，如果你要用除`Entity`的`playSound`方法外的方式（例如上面用`Level`的`playLocalSound`方法）播放**来自你的实体的**声音，**务必要进行**`if (!isSlient())`**的检查**（`playSound`方法内置了检查，所以不需要额外判断一次）。  

任何与播放声音有关的方法往往要涉及到两个`float`参数，一般**前面一个表示音量**（volume），**后面一个表示音调**（pitch）。部分情况下，可能用`random.nextFloat()`等实现音调和音量的随机化，使声音更自然。

接下来是数据保存与加载的部分。
```java
@Override
public void addAdditionalSaveData(CompoundTag tag) {
    super.addAdditionalSaveData(tag);
    BlockState carried = getCarriedBlock();
    if (carried != null) {
        tag.put("carriedBlockState", NbtUtils.writeBlockState(carried));
    }
    addPersistentAngerSaveData(tag);
}

@Override
public void readAdditionalSaveData(CompoundTag tag) {
     super.readAdditionalSaveData(tag);
     BlockState state = null;
  // 10表示CompoundTag（“混合”的NBT标签）
     if (tag.contains("carriedBlockState", 10)) {
        state = NbtUtils.readBlockState(level().holderLookup(Registries.BLOCK), tag.getCompound("carriedBlockState"));
        if (blockstate.isAir()) {
           state = null;
        }
     }
     setCarriedBlock(state);
     readPersistentAngerSaveData(level(), tag);
}
```
这里储存`BlockState`用的是`NbtUtils`里的`writeBlockState`方法，其余应该不难理解。但注意`readBlockState`在未找到NBT里的`BlockState`时，返回`Blocks.AIR.defaultBlockState()`而非null。  

接着来看一个重点，`isLookingAtMe`。
```java
// 这个方法的访问权限就是default
boolean isLookingAtMe(Player player) {
    ItemStack stack = player.getInventory().armor.get(3);
    if (ForgeHooks.shouldSuppressEnderManAnger(this, player, stack)) {
        return false;
    } else {
  //    getViewVector里的float参数与partialTicks与平滑渲染有关（0和1间差了一tick的更新），一般情况下填0或1即可
        Vec3 viewVector = player.getViewVector(1.0F).normalize();
        Vec3 vectorToPlayer = new Vec3(getX() - player.getX(), getEyeY() - player.getEyeY(), getZ() - player.getZ());
        double len = vectorToPlayer.length();
        vectorToPlayer = vectorToPlayer.normalize();
        double dotValue = viewVector.dot(vectorToPlayer);
        return dotValue > 1.0D - 0.025D / len ? player.hasLineOfSight(this) : false;
    }
}
```
这里涉及到了很多的向量运算，用来实现注视末影人的头部激怒末影人的效果。wiki上没有提到这个算法，所以用数学语言大致翻译一下：

设玩家坐标为$$(x_1, y_1, z_1)$$，末影人坐标为$$(x_2, y_2, z_2)$$  
记指向玩家所看向方向的单位向量为$$\vec{p}$$，记向量$$\vec{q}=(x_2-x_1, y_2-y_1, z_2-z_1)$$  
*（因为末影人的跟随范围为64，所以可以认为，总有$$|\vec{q}| \leq 64$$，这个下一节讲AI时再阐述原因）*   

如果满足$$\large \frac{\vec{p}\cdot\vec{q}}{|\vec{q}|} > 1-\frac{0.025}{|\vec{q}|}$$（即$$\large cos\langle\vec{p}, \vec{q}\rangle > 1-\frac{1}{40|\vec{q}|}$$）  

就返回true，否则返回false。  

可见**良好的数学基础在Mod开发中也起着重要作用**。这个方法的调用位置在末影人的AI里，下节再详细讲。  

在进入下一个重点前，先看三个Override。
```java
// 注意这里的dimension还是指尺寸
@Override
protected float getStandingEyeHeight(Pose pose, EntityDimensions dimensions) {
    return 2.55F;
}

@Override
public boolean isSensitiveToWater() {
    return true;
}

@Override
public boolean requiresCustomPersistence() {
    return super.requiresCustomPersistence() || getCarriedBlock() != null;
}
```
简要解释一下这三项：  
1. 因为末影人眼睛的高度偏高，高于默认值_0.85 * 身高2.9 = 2.465_，所以重新指定了眼睛的高度。  
2. `isSensitiveToWater`如果返回`true`，就说明`LivingEntity`遇任何形式的水会受到伤害。  
3. 手持方块的末影人不应该被刷掉，所以`requiresCustomPersistence`返回了`true`（意味着不会刷掉末影人）。  

然后是实体更新。
```java
@Override
public void aiStep() {
    if (level().isClientSide) {
        for (int i = 0; i < 2; ++i) {
//          getRandomX(n)返回：x坐标值 + 碰撞箱宽度 * r * n，其中r为[-1, 1)之间的随机双精度浮点数
//          1.20.1中所有实体类中用到的random均为RandomSource而非Random的实例
            level().addParticle(ParticleTypes.PORTAL, getRandomX(0.5D), getRandomY() - 0.25D, getRandomZ(0.5D), (random.nextDouble() - 0.5D) * 2.0D, -random.nextDouble(), (random.nextDouble() - 0.5D) * 2.0D);
        }
    }
    jumping = false;
    if (!level().isClientSide) {
        updatePersistentAnger((ServerLevel) level(), true);
    }
    super.aiStep();
}

@Override
protected void customServerAiStep() {
    if (level().isDay() && tickCount >= targetChangeTime + 600) {
        float lightLevel = getLightLevelDependentMagicValue();
        if (lightLevel > 0.5F && level().canSeeSky(blockPosition()) && random.nextFloat() * 30.0F < (lightLevel - 0.4F) * 2.0F) {
            setTarget(null);
        //  这个方法会使末影人随机传送，马上就会讲
            teleport();
        }
    }
    super.customServerAiStep();
}
```
`customServerAiStep`与`aiStep`方法的区别在于`customServerAiStep`方法**只会在服务端被调用**，而`aiStep`是**双端被调用**的。因此在`customServerAiStep`中无需`level().isClientSide`（或`level().isClientSide()`）的检查。  

分析代码可以发现，末影人的瞬移频率与亮度值呈正相关，且只有`lightLevelDependentMagicValue`（这个值与实际亮度和维度的环境光照都有关，不过`getLightLevelDependentMagicValue`方法被弃用了）大于0.5时才会尝试随机遗忘攻击目标并瞬移。

接下来讲下一个重点：瞬移。
```java
// 下面几个方法的访问权限很混乱，我也不知道为什么要这样写
protected boolean teleport() {
    if (!level().isClientSide() && isAlive()) {
        double rx = getX() + (random.nextDouble() - 0.5D) * 64.0D; // 水平32格范围随机传送
        double ry = getY() + (double) (random.nextInt(64) - 32); // 竖直方向上随机选取（自身y + n）的y坐标作为传送基准y（n为[-32, 32)内的随机整数）
        double rz = getZ() + (random.nextDouble() - 0.5D) * 64.0D;

        return teleport(rx, ry, rz);
    } else {
        return false;
    }
}

boolean teleportTowards(Entity entity) {
    Vec3 targetVector = new Vec3(getX() - entity.getX(), getY(0.5D) - entity.getEyeY(), getZ() - entity.getZ());
    targetVector = targetVector.normalize();
    double teleportDistance = 16.0D;

    double x = getX() + (random.nextDouble() - 0.5D) * 8.0D - targetVector.x * 16.0D; // 水平方向上，向entity方向传送16格，并加以4格的随机干扰（16.0D -> teleportDistance）
    double y = getY() + (double) (random.nextInt(16) - 8) - targetVector.y * 16.0D; // 竖直方向上随机选取（自身y + n）的y坐标作为传送基准y（n为[-8, 8)内的随机整数）
    double z = getZ() + (random.nextDouble() - 0.5D) * 8.0D - targetVector.z * 16.0D;

    return teleport(x, y, z);
}

private boolean teleport(double x, double y, double z) {
//  运用MutableBLockPos调节y坐标
    BlockPos.MutableBlockPos pos = new BlockPos.MutableBlockPos(x, y, z);
//  只要不是“固体方块”就下移
    while (pos.getY() > level().getMinBuildHeight() && !level().getBlockState(pos).blocksMotion()) { // blocksMotion方法已弃用，实际开发中尽量少用
        pos.move(Direction.DOWN);
    }
    BlockState state = level().getBlockState(pos);
    boolean blocksMotion = state.blocksMotion();
    boolean water = state.getFluidState().is(FluidTags.WATER);

    if (blocksMotion && !water) {
        EntityTeleportEvent.EnderEntity event = ForgeEventFactory.onEnderTeleport(this, x, y, z);
        if (event.isCanceled()) {
            return false;
        }

        Vec3 position = position();
        boolean teleported = randomTeleport(event.getTargetX(), event.getTargetY(), event.getTargetZ(), true);
        if (teleported) {
            level().gameEvent(GameEvent.TELEPORT, position, GameEvent.Context.of(this));
            if (!isSilent()) {
                level().playSound(null, xo, yo, zo, SoundEvents.ENDERMAN_TELEPORT, getSoundSource(), 1.0F, 1.0F);
            //  这里的playSound其实可以移出最深层的if
                playSound(SoundEvents.ENDERMAN_TELEPORT, 1.0F, 1.0F);
            }
        }

        return teleported;
    } else {
        return false;
    }
}
```
实体的瞬移的实现中含有大量的细节需要注意，一般分为2个主要的步骤：
1. 确定**大致**的瞬移位置，主要是**水平位置和基准y坐标**，又可分为以下的3小步：
  1. 找大致方向
  2. 添加水平随机扰动
  3. 确定下一步调整y坐标时使用的y坐标的基准值（简称为基准y坐标）
2. **在y上调整**，直至找到合适的y坐标（这一步主要应用于`LivingEntity`的瞬移，对于弹射物等实体的瞬移可以省略）

前两个方法主要实现了上述步骤的第1步。确定水平位置有两种常用的基本方法：
- 随机生成**x、z坐标**，直接作为水平传送位置（适用于在**矩形**内生成水平位置）
- 随机生成**角度和半径**，利用三角函数算出水平传送位置（适用于在**圆**内生成水平位置）  

如果有更复杂的需求，可以考虑重复尝试生成随机坐标，一成功就`break`。

*瞬移和重复尝试思想可以结合使用。在末影人受伤害瞬移时、及1.2.1.1节说过的盖亚守护者随机传送时，就利用到了这种结合。[暮色森林的巫妖（Lich）](https://github.com/TeamTwilight/twilightforest/blob/1.20.x/src/main/java/twilightforest/entity/boss/Lich.java)（第552~619行）也用到了很多的瞬移技巧，如果有需求或感兴趣，可以去阅读巫妖的源代码。*

第3个方法就实现了上述步骤的第2步。在有基准y坐标的情况下，常用的调整最终y坐标的方式是使用`MutableBlockPos`类，这种情况多见于对实体传送位置或召唤物生成位置的计算。而在没有基准y坐标的情况下，常常借助高度图（`Heightmap`）获取适宜的y坐标，这种情况多见于袭击等事件中对袭击者生成位置的计算。

在第一次用`MutableBlockPos`调整完后，接下来又用`randomTeleport`方法做了第二步调整。  
最后简单看一下`randomTeleport`的实现（不要被方法名带偏了，这个方法中没有生成任何随机坐标）：
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
    //  个人认为在这种情境下这一部分可以省略。可以直接跳到teleportTo(randomX, finalY, randomZ);
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
再看`dropCustomDeathLoot`方法。
```java
@Override
protected void dropCustomDeathLoot(DamageSource source, int lootingLevel, boolean killedByPlayer) {
    super.dropCustomDeathLoot(source, lootingLevel, killedByPlayer);
    BlockState carriedBlock = getCarriedBlock();
    if (carriedBlock != null) {
        ItemStack axe = new ItemStack(Items.DIAMOND_AXE);
        axe.enchant(Enchantments.SILK_TOUCH, 1);
        // 这个强转是安全的，因为这个方法不会在客户端被调用
        LootParams.Builder builder = new LootParams.Builder((ServerLevel) level())
            .withParameter(LootContextParams.ORIGIN, position())
            .withParameter(LootContextParams.TOOL, axe)
            .withOptionalParameter(LootContextParams.THIS_ENTITY, this);
        for (ItemStack drop : carriedBlock.getDrops(builder)) {
        //  spawnLocation方法用于生成携带指定ItemStack的ItemEntity
            spawnAtLocation(drop);
        }
    }
}
```
不难发现，杀死末影人后，如果末影人有手持的方块，会先获取末影人手持的方块**被附魔了精准采集的钻石斧采集时**的战利品表，根据这个战利品表再生成掉落物。
接下来是`hurt`方法。
```java
@Override
public boolean hurt(DamageSource source, float amount) {
    if (isInvulnerableTo(source)) {
        return false;
    } else {
        boolean willBeDamagedByPotion = source.getDirectEntity() instanceof ThrownPotion;
        if (!source.is(DamageTypeTags.IS_PROJECTILE) && !willBeDamagedByPotion) {
            boolean hurt = super.hurt(source, amount);
            if (!level().isClientSide() && !(source.getEntity() instanceof LivingEntity) && random.nextInt(10) != 0) {
                teleport();
            }
            return hurt;
        } else {
        //  source.is(DamageTypeTags.IS_PROJECTILE) || willBeDamagedByPotion
            boolean hurt = willBeDamagedByPotion && hurtWithCleanWater(source, (ThrownPotion) source.getDirectEntity(), amount);
        //  这就是重复尝试瞬移的一个应用
            for (int i = 0; i < 64; ++i) {
                if (teleport()) {
                    return true;
                }
            }
            return hurt;
        }
    }
}

private boolean hurtWithCleanWater(DamageSource source, ThrownPotion potionEntity, float amount) {
    ItemStack potionItem = potionEntity.getItem();
    Potion potion = PotionUtils.getPotion(potionItem);
    List<MobEffectInstance> potionEffects = PotionUtils.getMobEffects(potionItem);
    boolean empty = potion == Potions.WATER && potionEffects.isEmpty();
    // 不能把super.hurt换成hurt，否则会导致无限递归
    return empty ? super.hurt(source, amount) : false;
}
```
这部分内容主要实现了“末影人不会被弹射物伤害（只会被喷溅水瓶伤害）”的特性。注意这里巧妙地调用了`super.hurt`，以避免出现无限递归。还要注意，这里末影人虽然没有成功受到伤害，`hurt`方法也返回了`true`，此时末影人会“变红”，但不会损失生命值。

最后就是音效了。末影人实体类型的注册与僵尸大同小异，没有新的要点，因此此处不再分析。  
```java
@Override
protected SoundEvent getAmbientSound() {
    return isCreepy() ? SoundEvents.ENDERMAN_SCREAM : SoundEvents.ENDERMAN_AMBIENT;
}

@Override
protected SoundEvent getHurtSound(DamageSource source) {
    return SoundEvents.ENDERMAN_HURT;
}

@Override
protected SoundEvent getDeathSound() {
    return SoundEvents.ENDERMAN_DEATH;
}
```
注意不要忽略音效这种细节哦~  

末影人能力与特性的实现便分析到这里了。下一节将会分析末影人的AI。
