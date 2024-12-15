# 烈焰人的实现逻辑

读者可以先想一想烈焰人和雪傀儡、女巫、骷髅等攻击方式的区别。  

不难发现，烈焰人的小火球可以“三连发”，但后面三者每次攻击只能发射1个弹射物，这是由于`performRangeAttack`方法只能在远程攻击的那一游戏刻被调用一次的特性决定的。因此如果要实现烈焰人的“三连发”必须另起炉灶，自己写一个远程攻击的AI。  

```java
// 允许的最大自己与攻击目标间的高度差，当目标比自己高的距离大于这个值时烈焰人（绝大多数情况下）会向上运动
private float allowedHeightOffset = 0.5F;
// 剩余的重置allowedHeightOffset的时间（单位：tick），当这个值为0时会重置allowedHeightOffset的值
private int nextHeightOffsetChangeTick;
// 决定了烈焰人是否在准备攻击，如果这个值是奇数则表示烈焰人正准备攻击
private static final EntityDataAccessor<Byte> DATA_FLAGS_ID = SynchedEntityData.defineId(Blaze.class, EntityDataSerializers.BYTE);

public Blaze(EntityType<? extends Blaze> type, Level level) {
    super(type, level);
    // 这几行在1.2.1.3.2中讲过，所以本部分不再赘述，感兴趣的读者可以翻回1.2.1.3.2中看看
    setPathfindingMalus(BlockPathTypes.WATER, -1.0F);
    setPathfindingMalus(BlockPathTypes.LAVA, 8.0F);
    setPathfindingMalus(BlockPathTypes.DANGER_FIRE, 0.0F);
    setPathfindingMalus(BlockPathTypes.DAMAGE_FIRE, 0.0F);
    // 烈焰人被杀死后掉落10经验值
    xpReward = 10;
}
```
还有AI和属性。
```java
@Override
protected void registerGoals() {
    goalSelector.addGoal(4, new Blaze.BlazeAttackGoal(this));
    goalSelector.addGoal(5, new MoveTowardsRestrictionGoal(this, 1.0D));
    goalSelector.addGoal(7, new WaterAvoidingRandomStrollGoal(this, 1.0D, 0.0F));
    goalSelector.addGoal(8, new LookAtPlayerGoal(this, Player.class, 8.0F));
    goalSelector.addGoal(8, new RandomLookAroundGoal(this));
    // HurtByTargetGoal设置为setAlertOthers，说明烈焰人受攻击时会警告附近的所有烈焰人来攻击攻击自己的玩家
    targetSelector.addGoal(1, new HurtByTargetGoal(this).setAlertOthers());
    targetSelector.addGoal(2, new NearestAttackableTargetGoal<>(this, Player.class, true));
}

public static AttributeSupplier.Builder createAttributes() {
    return Monster.createMonsterAttributes()
            .add(Attributes.ATTACK_DAMAGE, 6.0D)
            .add(Attributes.MOVEMENT_SPEED, (double) 0.23F)
            // 烈焰人的攻击范围明显大于许多怪物
            .add(Attributes.FOLLOW_RANGE, 48.0D);
}
```
这里的`BlazeAttackGoal`就是烈焰人实现“三连发”的关键所在，下节我们来专门分析这个AI。  

烈焰人重写了`getLightLevelDependentMagicValue`方法（这个已被弃用的方法的返回值决定了实体的“亮度”，默认返回实体眼睛处所在方块的亮度，且返回值与`isSunBurnTick`的返回值有关，当这个方法的返回值小于等于0.5F时`isSunBurnTick`将总会返回`false`）和`isSensitiveToWater`方法。
```java
@Override
public float getLightLevelDependentMagicValue() {
    return 1.0F;
}
```

下面是`aiStep`方法和`customServerAiStep`方法，与`aiStep`不同的是`customServerAiStep`方法只会在服务端执行。这些方法实现了烈焰人的行为，具体的一些细节在注释里给出了。
```java
@Override
public void aiStep() {
    // 当烈焰人正在下降时，使其运动速度在y轴上的分速度降低以向下减速，
    if (!onGround() && getDeltaMovement().y < 0.0D) {
        setDeltaMovement(getDeltaMovement().multiply(1.0D, 0.6D, 1.0D));
    }
    if (level().isClientSide) {
        // 随机播放烈焰人噼啪作响的声音
        if (random.nextInt(24) == 0 && !isSilent()) {
            level().playLocalSound(getX() + 0.5D, getY() + 0.5D, getZ() + 0.5D, SoundEvents.BLAZE_BURN, getSoundSource(), 1.0F + random.nextFloat(), random.nextFloat() * 0.7F + 0.3F, false);
        }
        // 在四周生成一些黑烟
        for (int i = 0; i < 2; ++i) {
            level().addParticle(ParticleTypes.LARGE_SMOKE, getRandomX(0.5D), getRandomY(), getRandomZ(0.5D), 0.0D, 0.0D, 0.0D);
        }
    }
    super.aiStep();
}

@Override
protected void customServerAiStep() {
    --nextHeightOffsetChangeTick;
    if (nextHeightOffsetChangeTick <= 0) {
        // 重置nextHeightOffsetChangeTick的值
        nextHeightOffsetChangeTick = 100;
        // 下面的random.triangle方法返回一个取值在(0.5 - 6.891, 0.5 + 6.891)区间内的随机双精度浮点数，且取接近0.5的值的概率更大
        allowedHeightOffset = (float) random.triangle(0.5D, 6.891D);
    }
    LivingEntity target = getTarget();
    if (target != null && target.getEyeY() > getEyeY() + (double) allowedHeightOffset && canAttack(target)) {
        Vec3 deltaMovement = getDeltaMovement();
        // 大多数情况下给自己一个向上的加速度，如果烈焰人向上运动得足够快则会给它向下的加速度
        setDeltaMovement(getDeltaMovement().add(0.0D, ((double) 0.3F - deltaMovement.y) * (double) 0.3F, 0.0D));
        // 将hasImpulse赋值为true，以通知客户端更新实体的运动，注意只在服务端用setDeltaMovement方法使实体发生了运动后往往都要给hasImpluse赋值true
        hasImpulse = true;
    }
    super.customServerAiStep();
}
```
下面是有关烈焰人准备攻击（charged）的状态的一些逻辑。如果烈焰人在准备攻击，那么烈焰人会着火。
```java
@Override
protected void defineSynchedData() {
    super.defineSynchedData();
    entityData.define(DATA_FLAGS_ID, (byte) 0);
}

@Override
public boolean isOnFire() {
    return isCharged();
}

private boolean isCharged() {
    return (entityData.get(DATA_FLAGS_ID) & 1) != 0;
}

void setCharged(boolean charged) {
    byte flags = entityData.get(DATA_FLAGS_ID);
    if (charged) {
        flags = (byte) (flags | 1); // 给flags的最后一位赋值1
    } else {
        flags = (byte) (flags & -2); // 给flags的最后一位赋值0
    }
    entityData.set(DATA_FLAGS_ID, flags);
}
```

还有音效等杂项。
```java
// 烈焰人在水/雨中会受到伤害
@Override
public boolean isSensitiveToWater() {
     return true;
}

@Override
protected SoundEvent getAmbientSound() {
    return SoundEvents.BLAZE_AMBIENT;
}

@Override
protected SoundEvent getHurtSound(DamageSource source) {
    return SoundEvents.BLAZE_HURT;
}

@Override
protected SoundEvent getDeathSound() {
    return SoundEvents.BLAZE_DEATH;
}
```
本节的内容就是这些了，下一节将介绍`BlazeAttackGoal`的实现。