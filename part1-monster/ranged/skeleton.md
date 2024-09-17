# 全能的骷髅

骷髅不仅是一种远程攻击的生物，失去弓后还能近战，并且十分聪明。在游戏中，它往往是第一天晚上玩家最害怕的怪物之一。本节将分析骷髅的基本实现方式，从而基本理清这个行为较复杂的怪物的实现逻辑。骷髅和僵尸在底层实现上有很多相似之处，本节也会经常提到1.2.1.1节的内容，因此可以先复习一下1.2.1.1讲过的东西~  

`Skeleton`类的代码似乎不多，但它的父类`AbstractSkeleton`内容丰富。我们先来看`AbstractSkeleton`类。

```java
protected AbstractSkeleton(EntityType<? extends AbstractSkeleton> type, Level level) {
    super(type, level);
    reassessWeaponGoal();
}
```
注意构造方法中调用了`reassessWeaponGoal`方法。顾名思义，这个方法用于根据骷髅自身的状态来决定使用近战的AI还是远程攻击的AI，让我们来看看这个方法。
```java
// 远程攻击的AI
private final RangedBowAttackGoal<AbstractSkeleton> bowGoal = new RangedBowAttackGoal<>(this, 1.0D, 20, 15.0F);
// 近战的AI。注意这个匿名内部类里调用了setAggressive方法，在渲染骷髅时，会根据骷髅是否aggressive来决定骷髅的动作
private final MeleeAttackGoal meleeGoal = new MeleeAttackGoal(this, 1.2D, false) {
    public void stop() {
        super.stop();
        AbstractSkeleton.this.setAggressive(false);
    }

    public void start() {
        super.start();
        AbstractSkeleton.this.setAggressive(true);
    }
};

public void reassessWeaponGoal() {
//  确保下面的逻辑在服务端执行，记住对goalSelector操作前一定要判断是否是服务端
    if (level() != null && !level().isClientSide) {
        // 先移除两个AI，马上再按需添加
        goalSelector.removeGoal(meleeGoal);
        goalSelector.removeGoal(bowGoal);
        // 这一段比较直白，因此不额外解释。注意下面的minAttackInterval为骷髅远程攻击的最短间隔时间
        ItemStack stack = getItemInHand(ProjectileUtil.getWeaponHoldingHand(this, item -> item instanceof BowItem));
        if (stack.is(Items.BOW)) {
            int minAttackInterval = 20;
            if (level().getDifficulty() != Difficulty.HARD) {
                minAttackInterval = 40;
            }
            bowGoal.setMinAttackInterval(minAttackInterval);
            goalSelector.addGoal(4, bowGoal);
        } else {
            goalSelector.addGoal(4, meleeGoal);
        }
    }
}
```
骷髅（以及其他骷髅的变种）实现根据手上武器改变攻击方式的原理便是每当手上武器**可能有变化**时，调用`reassessWeaponGoal`方法来调整AI。  
以下是该方法的其他被调用的位置：
- `finalizeSpawn`方法，即骷髅生成时根据生成时手上的武器判断一次
- `readAdditionalSaveData`方法，即骷髅被重新加载时判断一次，因为**实体的AI不会被保存到NBT标签中**，所以这个判断很有必要
- `setItemSlot`方法，即骷髅手上的武器被改变时判断一次

AI部分，其中前两个`Goal`与骷髅躲避阳光的行为有关，下一节会具体说，剩余的`Goal`比较常规。
```java
@Override
protected void registerGoals() {
    goalSelector.addGoal(2, new RestrictSunGoal(this));
    goalSelector.addGoal(3, new FleeSunGoal(this, 1.0D));
//  下面的AvoidEntityGoal实现了骷髅躲避狼的行为。倒数第三个参数分别表示了最大躲避距离（与狼的最近距离在这个值内就会躲开狼），
//  最后两个参数分别决定了躲避过程中骷髅的行走速度与冲刺速度（与狼的最近距离在7以内会“冲刺”，否则会“行走”）
    goalSelector.addGoal(3, new AvoidEntityGoal<>(this, Wolf.class, 6.0F, 1.0D, 1.2D));
    goalSelector.addGoal(5, new WaterAvoidingRandomStrollGoal(this, 1.0D));
    goalSelector.addGoal(6, new LookAtPlayerGoal(this, Player.class, 8.0F));
    goalSelector.addGoal(6, new RandomLookAroundGoal(this));
    targetSelector.addGoal(1, new HurtByTargetGoal(this));
    targetSelector.addGoal(2, new NearestAttackableTargetGoal<>(this, Player.class, true));
    targetSelector.addGoal(3, new NearestAttackableTargetGoal<>(this, IronGolem.class, true));
//  下面这行与僵尸完全相同，此处不重复解释
    targetSelector.addGoal(3, new NearestAttackableTargetGoal<>(this, Turtle.class, 10, true, false, Turtle.BABY_ON_LAND_SELECTOR));
}
```

`aiStep`方法和`finalizeSpawn`方法，这两个方法与僵尸的这两个方法非常接近，所以不会详细解释
```java
@Override
public void aiStep() {
//  使骷髅在阳光下着火（正如1.2.1.1所说，aiStep方法与僵尸的非常相似……）
    boolean shouldBurn = isSunBurnTick();
    if (shouldBurn) {
        ItemStack helmet = getItemBySlot(EquipmentSlot.HEAD);
        if (!helmet.isEmpty()) {
            if (helmet.isDamageableItem()) {
                helmet.setDamageValue(helmet.getDamageValue() + random.nextInt(2));
                if (helmet.getDamageValue() >= helmet.getMaxDamage()) {
                    broadcastBreakEvent(EquipmentSlot.HEAD);
                    setItemSlot(EquipmentSlot.HEAD, ItemStack.EMPTY);
               }
            }
            shouldBurn = false;
        }
        if (shouldBurn) {
            setSecondsOnFire(8);
        }
    }
    super.aiStep();
}

@Override
@Nullable
public SpawnGroupData finalizeSpawn(ServerLevelAccessor accessor, DifficultyInstance difficulty, MobSpawnType type, @Nullable SpawnGroupData spawnData, @Nullable CompoundTag tag) {
    spawnData = super.finalizeSpawn(accessor, difficulty, type, spawnData, tag);

//  除调用了reassessWeaponGoal方法以外与僵尸相同
    RandomSource source = accessor.getRandom();
    populateDefaultEquipmentSlots(source, difficulty);
    populateDefaultEquipmentEnchantments(source, difficulty);
    reassessWeaponGoal();
    setCanPickUpLoot(source.nextFloat() < 0.55F * difficulty.getSpecialMultiplier());

//  万圣节的彩蛋，与僵尸相同
    if (getItemBySlot(EquipmentSlot.HEAD).isEmpty()) {
        LocalDate date = LocalDate.now();
        int day = date.get(ChronoField.DAY_OF_MONTH);
        int month = date.get(ChronoField.MONTH_OF_YEAR);
        if (month == 10 && day == 31 && source.nextFloat() < 0.25F) {
            setItemSlot(EquipmentSlot.HEAD, new ItemStack(source.nextFloat() < 0.1F ? Blocks.JACK_O_LANTERN : Blocks.CARVED_PUMPKIN));
            armorDropChances[EquipmentSlot.HEAD.getIndex()] = 0.0F;
        }
    }

    return spawnData;
}
```

当然骷髅也有自己的独特之处，下面是与骷髅远程攻击有关的内容，也是本节的重难点。
```java
@Override
public void performRangedAttack(LivingEntity target, float power) {
    ItemStack projectile = getProjectile(getItemInHand(ProjectileUtil.getWeaponHoldingHand(this, item -> item instanceof BowItem)));
    AbstractArrow arrow = getArrow(projectile, power);
    if (getMainHandItem().getItem() instanceof BowItem) {
        arrow = ((BowItem) getMainHandItem().getItem()).customArrow(arrow);
    }
    double dx = target.getX() - getX();
    double dy = target.getY(0.3333333333333333D) - arrow.getY();
    double dz = target.getZ() - getZ();
    double distance = Math.sqrt(dx * dx + dz * dz);
    arrow.shoot(dx, dy + distance * (double) 0.2F, dz, 1.6F, (float) (14 - level().getDifficulty().getId() * 4));
    playSound(SoundEvents.SKELETON_SHOOT, 1.0F, 1.0F / (getRandom().nextFloat() * 0.4F + 0.8F));
    level().addFreshEntity(arrow);
}

@Override
protected AbstractArrow getArrow(ItemStack stack, float power) {
    return ProjectileUtil.getMobArrow(this, stack, power);
}

@Override
public boolean canFireProjectileWeapon(ProjectileWeaponItem item) {
    return item == Items.BOW;
}
```
`Monster`类：
```java
@Override
public ItemStack getProjectile(ItemStack stack) {
    if (stack.getItem() instanceof ProjectileWeaponItem) {
//      返回的这个Predicate用于判断自己的武器能否使用手持的弹射物 
        Predicate<ItemStack> predicate = ((ProjectileWeaponItem) stack.getItem()).getSupportedHeldProjectiles();
//      getHeldProjectile方法返回手持的弹射物
        ItemStack heldProjectile = ProjectileWeaponItem.getHeldProjectile(this, predicate);
//      ForgeHook里的getProjectile方法涉及到了LivingGetProjectileEvent事件的发送 
        return ForgeHooks.getProjectile(this, stack, itemstack.isEmpty() ? new ItemStack(Items.ARROW) : heldProjectile);
    } else {
        return ForgeHooks.getProjectile(this, stack, ItemStack.EMPTY);
    }
}
```
`ProjectileUtil`类：
```java
public static InteractionHand getWeaponHoldingHand(LivingEntity livingEntity, Predicate<Item> itemPredicate) {
    return itemPredicate.test(livingEntity.getMainHandItem().getItem()) ? InteractionHand.MAIN_HAND : InteractionHand.OFF_HAND;
}

public static AbstractArrow getMobArrow(LivingEntity livingEntity, ItemStack arrow, float power) {
    ArrowItem arrowItem = (ArrowItem) (arrow.getItem() instanceof ArrowItem ? arrow.getItem() : Items.ARROW);
//  createArrow方法创建了箭的实体，并且将箭具有的状态效果复制到了这个实体上
    AbstractArrow newArrow = arrowItem.createArrow(livingEntity.level(), arrow, livingEntity);
//  根据所持武器的附魔，为箭应用附魔的效果
    newArrow.setEnchantmentEffectsFromEntity(livingEntity, power);
//  如果箭是药箭，那就再一次将箭具有的状态效果复制到这个实体上（可能是为了防止setEnchantmentEffectsFromEntity方法对药箭具有的的状态效果产生影响）
    if (arrow.is(Items.TIPPED_ARROW) && newArrow instanceof Arrow) {
        ((Arrow) newArrow).setEffectsFromItem(arrow);
    }
    return newArrow;
}
```
`BowItem`类：
```java
// 根据弓的类型自定义箭的类型，这应该是Forge加的方法
public AbstractArrow customArrow(AbstractArrow arrow) {
    return arrow;
}
```
拆解一下performRangedAttack方法。  

这是上半部分：
```java
ItemStack projectile = getProjectile(getItemInHand(ProjectileUtil.getWeaponHoldingHand(this, item -> item instanceof BowItem)));
AbstractArrow arrow = getArrow(projectile, power);
if (getMainHandItem().getItem() instanceof BowItem) {
    arrow = ((BowItem) getMainHandItem().getItem()).customArrow(arrow);
}
```
这部分**获取了将要发射出去的弹射物的类型**，**注意所有使用弓的生物都需要这样的处理**，以确保TA们发射出正确的弹射物。里面涉及到的几个方法上面都列出来了并写了注释，如果想弄清楚这个过程是如何实现的，可以参考上面的内容。  

下半部分：
```java
double dx = target.getX() - getX();
// 为了确保箭瞄准目标的身体（偏下部）而非地面，这儿获取目标的y坐标时向上偏移了目标碰撞箱高度的1/3。
double dy = target.getY(0.3333333333333333D) - arrow.getY();
double dz = target.getZ() - getZ();
double distance = Math.sqrt(dx * dx + dz * dz);
// 这里根据难度调整了骷髅射击的精度，困难模式下骷髅射得更准，就是因为最后一个参数的数值小
arrow.shoot(dx, dy + distance * (double) 0.2F, dz, 1.6F, (float) (14 - level().getDifficulty().getId() * 4));
playSound(SoundEvents.SKELETON_SHOOT, 1.0F, 1.0F / (getRandom().nextFloat() * 0.4F + 0.8F));
level().addFreshEntity(arrow);
```
这部分就没有什么非常特别的地方了，如果有不明白的，可以复习1.2.2.2与1.2.2.4的相关内容。

最后`AbstractSkeleton`类中的杂项。注意重写了`rideTick`方法使骷髅骑手的行为正常。
```java
public static AttributeSupplier.Builder createAttributes() {
    return Monster.createMonsterAttributes().add(Attributes.MOVEMENT_SPEED, 0.25D);
}

@Override
protected void playStepSound(BlockPos pos, BlockState state) {
    playSound(getStepSound(), 0.15F, 1.0F);
}

@Override
protected abstract SoundEvent getStepSound();

@Override
public MobType getMobType() {
    return MobType.UNDEAD;
}

@Override
public void rideTick() {
    super.rideTick();
    Entity vehicle = this.getControlledVehicle();
    if (vehicle instanceof PathfinderMob pathfinderMob) {
        yBodyRot = pathfinderMob.yBodyRot;
    }
}

@Override
public void readAdditionalSaveData(CompoundTag tag) {
    super.readAdditionalSaveData(tag);
    reassessWeaponGoal();
}

@Override
public void setItemSlot(EquipmentSlot slot, ItemStack stack) {
    super.setItemSlot(slot, stack);
    if (!level().isClientSide) {
        reassessWeaponGoal();
    }
}

@Override
protected float getStandingEyeHeight(Pose pose, EntityDimensions dimensions) {
    return 1.74F;
}

@Override
public double getMyRidingOffset() {
    return -0.6D;
}

@Override
public boolean isShaking() {
    return isFullyFrozen();
}
```

再来看子类`Skeleton`类的内容，子类实现了骷髅陷入细雪后的转化以及骷髅头颅的掉落。先来看与骷髅转化为流浪者的过程有关的代码。
```java
private static final int TOTAL_CONVERSION_TIME = 300;
// 决定了骷髅是否正在转化为流浪者，值为true则正在转化
private static final EntityDataAccessor<Boolean> DATA_STRAY_CONVERSION_ID = SynchedEntityData.defineId(Skeleton.class, EntityDataSerializers.BOOLEAN);
public static final String CONVERSION_TAG = "StrayConversionTime";
// 陷入细雪的时间（tick）
private int inPowderSnowTime;
// 开始转化的时间（tick）
private int conversionTime;

public boolean isFreezeConverting() {
    return getEntityData().get(DATA_STRAY_CONVERSION_ID);
}

public void setFreezeConverting(boolean converting) {
    entityData.set(DATA_STRAY_CONVERSION_ID, converting);
}

// 正在转化过程中的骷髅身体会抖动
@Override
public boolean isShaking() {
    return isFreezeConverting();
}

@Override
public void tick() {
    if (!level().isClientSide && isAlive() && !isNoAi()) {
        if (isInPowderSnow) {
            if (isFreezeConverting()) {
//              如果正在转化过程中，每刻减少转换时间
                --conversionTime;
                if (conversionTime < 0) {
//                  转化为流浪者
                    doFreezeConversion();
                }
            } else {
//              如果在细雪中，增加在细雪中的时间
                ++inPowderSnowTime;
//              如果在细雪中的时间大于等于140刻，开始300刻的转化过程
                if (inPowderSnowTime >= 140) {
                    startFreezeConversion(300);
                }
            }
        } else {
//          重置转化时间，并设置为不在转化过程中
            inPowderSnowTime = -1;
            setFreezeConverting(false);
        }
    }
    super.tick();
}

private void startFreezeConversion(int conversionTime) {
    this.conversionTime = conversionTime;
    setFreezeConverting(true);
}

protected void doFreezeConversion() {
    convertTo(EntityType.STRAY, true);
    if (!isSilent()) {
        level().levelEvent(null, 1048, blockPosition(), 0);
    }
}

@Override
public boolean canFreeze() {
//  骷髅不会被冻伤
    return false;
}
```
掉落头颅的部分与僵尸基本相同，只不过掉的是骷髅的头。
```java
@Override
protected void dropCustomDeathLoot(DamageSource source, int lootingLevel, boolean killedByPlayer) {
    super.dropCustomDeathLoot(source, lootingLevel, killedByPlayer);
    Entity entity = source.getEntity();
    if (entity instanceof Creeper creeper) {
        if (creeper.canDropMobsSkull()) {
            creeper.increaseDroppedSkulls();
            spawnAtLocation(Items.SKELETON_SKULL);
        }
    }
}
```
本节的内容就到此为止了，下一节将分析上面说过的骷髅的两个特殊的AI以及骷髅远程攻击时所特有的行为。