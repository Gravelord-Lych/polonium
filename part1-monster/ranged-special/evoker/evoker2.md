# 唤魔者的三大魔法

唤魔者会使用的法术共分为3种，可以用以下的表格来简单概括：  

|       施法AI类名      |        作用        | 优先级 | 准备时间 | 总用时 | 冷却时间 |
|:---------------------:|:------------------:|:------:|:------------:|:----------:|:--------:|
| `EvokerSummonSpellGoal` |      召唤恼鬼      |    4   |      20      |     40     |    340   |
| `EvokerAttackSpellGoal` |   召唤唤魔者尖牙   |    5   |      20      |     100    |    100   |
| `EvokerWololoSpellGoal` | 把蓝色绵羊变成红色 |    6   |      40      |     60     |    140   |

*注：*
- *上表中所有时间的单位均为tick*
- *优先级：数字越低，优先级越高，当多个AI冷却完毕时，高优先级的AI优先执行*
- *准备时间：即`getCastWarmupTime`返回值，原版中该方法总是返回常数*
- *总用时：即`getCastingTime`返回值，原版中该方法总是返回常数*
- *冷却时间：即`getCastingInterval`返回值，原版中该方法总是返回常数*

上一节中讲过`SpellcasterUseSpellGoal`，其中的`canUse`方法保证施法AI之间是互斥的，因此同一时刻只会执行一个施法AI，且优先级高的施法AI先执行。  

接下来具体分析每一种AI时，为了避免文章冗余，笔者将会只分析除`getCastWarmupTime`、`getCastingTime`与`getCastingInterval`外的，对施法AI的底层实现有重要意义的方法。  

### 召唤恼鬼（`EvokerSummonSpellGoal`）

这是个“很不受玩家喜欢”的法术，但是这也是底层逻辑十分简单的法术~  

唤魔者只会在自身碰撞箱向6个方向扩展16格后形成的长方体内恼鬼数量小于8时召唤恼鬼，且四周的恼鬼越多，使用该法术的概率越小。

```java
@Override
public boolean canUse() {
    if (!super.canUse()) {
        return false;
    } else {
        int i = level().getNearbyEntities(Vex.class, vexCountTargeting, Evoker.this, getBoundingBox().inflate(16.0D)).size();
        return random.nextInt(8) + 1 > i;
    }
}
```

然后是`performSpellCasting`。该方法会在唤魔者四周固定生成3只恼鬼。
```java
@Override
protected void performSpellCasting() {
    ServerLevel serverLevel = (ServerLevel) level();
    for (int i = 0; i < 3; ++i) {
        // 选取自身四周的随机位置，注意召唤出的恼鬼的y坐标永远是唤魔者的y坐标+1
        BlockPos pos = blockPosition().offset(-2 + random.nextInt(5), 1, -2 + random.nextInt(5));
        Vex vex = EntityType.VEX.create(level());
        if (vex != null) { // 这里有个null-check，是因为EntityTyoe的create方法是@Nullable的。不过绝大多数情况下create方法都不会返回null
            vex.moveTo(pos, 0.0F, 0.0F);
            vex.finalizeSpawn(serverLevel, level().getCurrentDifficultyAt(pos), MobSpawnType.MOB_SUMMONED, null, null);
            vex.setOwner(Evoker.this);
            // 限制恼鬼的活动范围
            vex.setBoundOrigin(pos);
            // 使恼鬼在一定时间后不断受到伤害
            vex.setLimitedLife(20 * (30 + random.nextInt(90)));
            serverLevel.addFreshEntityWithPassengers(vex);
        }
    }
}
```

### 把蓝色绵羊变成红色（`EvokerWololoSpellGoal`）

这个法术本身也很简单，只是`canUse`比较复杂而已。

```java
@Override
public boolean canUse() {
    // 显然唤魔者在战斗的时候没有闲心调教绵羊
    if (getTarget() != null) {
        return false;
    } else if (isCastingSpell()) {
        return false;
    } 
    // 这个AI彻底重写了canUse方法，所以要重新判断法术是否在冷却
    else if (tickCount < nextAttackTickCount) {
        return false;
    } 
    // 再次提醒读者不要忘记确认GameRule是否允许实体的行为
    else if (!ForgeEventFactory.getMobGriefingEvent(level(), Evoker.this)) {
        return false;
    } 
    else {
        List<Sheep> list = level().getNearbyEntities(Sheep.class, wololoTargeting, Evoker.this, getBoundingBox().inflate(16.0D, 4.0D, 16.0D));
        if (list.isEmpty()) {
            return false;
        } else {
            // wololoTarget和target是独立的，因此不能替换为setTarget
            setWololoTarget(list.get(random.nextInt(list.size())));
            return true;
        }
    }
}

@Override
public boolean canContinueToUse() {
    return getWololoTarget() != null && attackWarmupDelay > 0;
}
```

`performSpellCasting`中简单地改变了绵羊的颜色。
```java
@Override
protected void performSpellCasting() {
    Sheep sheep = Evoker.this.getWololoTarget();
    if (sheep != null && sheep.isAlive()) {
        sheep.setColor(DyeColor.RED);
    }
}
```

最后不要忘了重置`wololoTarget`。
```java
@Override
public void stop() {
    super.stop();
    Evoker.this.setWololoTarget((Sheep)null);
}
```

### 召唤唤魔者尖牙（`EvokerAttackSpellGoal`）

这看上去是唤魔者使用的最朴素的法术，可其实这个法术在“代码意义”上才是最复杂的法术。复杂就复杂在尖牙的位置的确定。  

先考虑单个水平位置（x与z）已被确定的尖牙。尖牙的y坐标是如何确定的呢？读者可能会认为，使用地面的y坐标值不就行了吗？但问题是哪里才是所谓的“地面”呢？  

回顾1.2.1.3.2中提到的找y坐标的方式：**用`MutableBlockPos`调整**或**通过`Heightmap`获取**。如果借助高度图（`Heightmap`）获取，显然因为这种方式因为通过获取水平位置上固体方块的最大y坐标值来确定坐标，所以在室内环境等环境下并不适用。  

所以只能采用前一种方式，不断检查尖牙生成位置并以此调整y坐标。原版的实现中没有使用`MutableBlockPos`，但是思想是一样的。原版对一定y坐标范围内的方块进行了检查，如果检测到合适的可生成尖牙的位置就生成尖牙。  
```java
// warmupDelayTicks是尖牙出现的额外的延迟时间，算上施法本身的准备时间，在tps为20的情况下，一个warmupDelayTicks为5的尖牙会在从唤魔者举起手开始1.25s后出现
private void createSpellEntity(double x, double z, double minY, double maxY, float rotation, int warmupDelayTicks) {
    BlockPos pos = BlockPos.containing(x, maxY, z);
    boolean foundSuitablePos = false;
    double dy = 0.0D;
    
    do {
        BlockPos belowPos = pos.below();
        BlockState belowState = level().getBlockState(belowPos);
        // 检查该位置下方的方块是否能够为该位置的方块提供SupportType.FULL，用于检查该位置方块是否“不透明”。
        if (belowState.isFaceSturdy(level(), belowPos, Direction.UP)) {
            if (!level().isEmptyBlock(pos)) {
                BlockState state = level().getBlockState(pos);
                VoxelShape shape = state.getCollisionShape(level(), pos);
                if (!shape.isEmpty()) {
                    dy = shape.max(Direction.Axis.Y);
                }
            }
            foundSuitablePos = true;
            break;
        }
        pos = pos.below();
    } while (pos.getY() >= Mth.floor(minY) - 1);
    
    if (foundSuitablePos) {
        level().addFreshEntity(new EvokerFangs(level(), x, (double) pos.getY() + dy, z, rotation, warmupDelayTicks, Evoker.this));
    }
}
```
*附上Wiki的描述：尖牙只会生成在不低于位置最低的目标的脚、不高于位置最高的目标的脚上方1格的位置。尖牙会尝试在这一区域的不透明方块上生成，但当这个方块被固体方块挡住时会生成失败。这意味着召唤的尖牙无法生成在深坑里或高墙顶，但如果唤魔者在深坑的底部而目标在上方，它会试图走到顶部以发动攻击，反之亦然。*

然后考虑尖牙水平位置的确定。水平位置的确定则比较简单，只要进行一些三角函数的运算就可以了。  

下图展示了唤魔者尖牙的水平位置的大致确定方式。
![evoker_fang_position](../images/evoker_fang_position.png)

以及相关的代码。
```java
@Override
protected void performSpellCasting() {
    LivingEntity target = getTarget();
    double minY = Math.min(target.getY(), getY());
    double maxY = Math.max(target.getY(), getY()) + 1.0D;
    float facingAngle = (float) Mth.atan2(target.getZ() - getZ(), target.getX() - getX());
    // 对距离较近的目标：召唤两圈尖牙
    if (distanceToSqr(target) < 9.0D) {
        for (int i = 0; i < 5; ++i) {
            float rotation = facingAngle + (float) i * (float) Math.PI * 0.4F;
            createSpellEntity(getX() + (double) Mth.cos(rotation) * 1.5D, getZ() + (double) Mth.sin(rotation) * 1.5D, minY, maxY, rotation, 0);
        }
        for (int k = 0; k < 8; ++k) {
            // 注：1.2566371F = Math.PI * 0.4F
            float rotation = facingAngle + (float) k * (float) Math.PI * 2.0F / 8.0F + 1.2566371F;
            createSpellEntity(getX() + (double) Mth.cos(rotation) * 2.5D, getZ() + (double) Mth.sin(rotation) * 2.5D, minY, maxY, rotation, 3);
        }
    } 
    // 对距离较远的目标：召唤一排尖牙
    else {
        for (int l = 0; l < 16; ++l) {
            double distanceToSelf = 1.25D * (double) (l + 1);
            int warmupDelayTicks = 1 * l;
            createSpellEntity(getX() + (double) Mth.cos(facingAngle) * distanceToSelf, getZ() + (double) Mth.sin(facingAngle) * distanceToSelf, minY, maxY, facingAngle, warmupDelayTicks);
        }
    }
}
```

那么唤魔者的逻辑部分到这里就结束了，接下来则是唤魔者的渲染部分。