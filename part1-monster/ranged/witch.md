# 令人头疼的女巫

***注：***

*以后的实体分析中，如果实体较复杂，将只会分析其中重要的内容，一些共通的内容例如实体属性、音效的添加/实体类型的注册由于曾经讲过，将不再赘述。*

---

在1.2.2的上半部分中，我们学习了实现简单的远程攻击的怪物的方式，但是写出来的怪物好像过于单调，缺乏吸引力与对付难度。作为模组开发者，我们自然要为难一下玩家，以提高模组的挑战性。因此我们需要更进一步地，学习拥有多种能力的怪物的实现方式，而玩家们痛恨的女巫便给了我们一个极好的参考。所以本部分我们将分析女巫的实现，来帮助我们更好地理解这种攻击方式复杂多变，令玩家头疼不已的敌对生物。  

在游戏过程中不难发现女巫有如下的3个能力（*此处的“能力”与Forge提供的`Capability`系统无关*）：
1. 向玩家投掷带有负面效果的喷溅药水
2. （在袭击中）向其他袭击者投掷带有正面效果的喷溅药水
3. 喝带有正面效果的药水以治疗自身

这些能力分别是怎样实现的呢？我们来对女巫的能力归类，可以发现能力1、2十分相似，都是“投掷药水”的能力。因此自然想到通过设置女巫的`target`，并让女巫“攻击”玩家和袭击者时使用不同的药水，来使女巫具备这两个能力。

但是如何才能让女巫能有条不紊地在治疗袭击者的同时攻击玩家，避免女巫光顾着治疗不攻击呢？

这个问题值得思考，我们可以想到通过精准地调控2个不同的`TargetGoal`来实现，查阅女巫的源代码，可以找到这样的两个AI。
```java
private NearestHealableRaiderTargetGoal<Raider> healRaidersGoal; // 治疗袭击者的AI
private NearestAttackableWitchTargetGoal<Player> attackPlayersGoal; // 攻击玩家的AI
```

这两个AI是如何发挥作用的呢？先看`registerGoals()`方法。
```java
@Override
protected void registerGoals() {
    super.registerGoals();
    healRaidersGoal = new NearestHealableRaiderTargetGoal<>(this, Raider.class, true, (healTarget) -> {
        return healTarget != null && hasActiveRaid() && healTarget.getType() != EntityType.WITCH;
    });
    attackPlayersGoal = new NearestAttackableWitchTargetGoal<>(this, Player.class, 10, true, false, null); // “10”表示每刻有1/10的概率寻找目标
    goalSelector.addGoal(1, new FloatGoal(this));
    goalSelector.addGoal(2, new RangedAttackGoal(this, 1.0D, 60, 10.0F));
    goalSelector.addGoal(2, new WaterAvoidingRandomStrollGoal(this, 1.0D));
    goalSelector.addGoal(3, new LookAtPlayerGoal(this, Player.class, 8.0F));
    goalSelector.addGoal(3, new RandomLookAroundGoal(this));
    targetSelector.addGoal(1, new HurtByTargetGoal(this, Raider.class));
    targetSelector.addGoal(2, healRaidersGoal);
    targetSelector.addGoal(3, attackPlayersGoal);
}
```
这里除了一些通用的AI外，还分别对`healRaidersGoal`和`attackPlayersGoal`进行了赋值，并把它们添加进了`targetSelector`中。我们也可以发现，`healRaidersGoal`与`attackPlayersGoal`的优先级不同，这使得女巫会优先治疗袭击者。  

那么为什么女巫不会光顾着治疗不攻击呢？在这里似乎不能找到答案，但是我们可以关注`NearestHealableRaiderTargetGoal`与`NearestAttackableWitchTargetGoal`，看看这两个类有没有暗藏玄机。

`NearestHealableRaiderTargetGoal`：
```java
public class NearestHealableRaiderTargetGoal<T extends LivingEntity> extends NearestAttackableTargetGoal<T> {
    private static final int DEFAULT_COOLDOWN = 200;
    private int cooldown = 0;
    
    public NearestHealableRaiderTargetGoal(Raider raider, Class<T> targetType, boolean mustSee, @Nullable Predicate<LivingEntity> targetSelector) {
        super(raider, targetType, 500, mustSee, false, targetSelector); // “500”表示每刻有1/500的概率寻找目标
    }
    
    public int getCooldown() {
        return cooldown;
    }
    
    public void decrementCooldown() {
        --cooldown;
    }
    
    @Override
    public boolean canUse() {
        if (cooldown <= 0 && mob.getRandom().nextBoolean()) {
            if (!((Raider) mob).hasActiveRaid()) {
                return false;
            } else {
                findTarget();
                return target != null;
            }
        } else { // cooldown > 0时即AI在冷却时返回false
            return false;
        }
    }
    
    @Override
    public void start() {
        cooldown = reducedTickDelay(200);
        super.start();
    }
}
```
`NearestAttackableWitchTargetGoal`：
```java
public class NearestAttackableWitchTargetGoal<T extends LivingEntity> extends NearestAttackableTargetGoal<T> {
    private boolean canAttack = true;

    public NearestAttackableWitchTargetGoal(Raider raider, Class<T> targetType, int randomInterval, boolean mustSee, boolean mustReach, @Nullable Predicate<LivingEntity> targetSelector) {
        super(raider, targetType, randomInterval, mustSee, mustReach, targetSelector);
    }

    public void setCanAttack(boolean canAttack) {
        this.canAttack = canAttack;
    }

    @Override
    public boolean canUse() {
        return canAttack && super.canUse();
    }
}
```
具体更新这两个AI的部分则在`aiStep`方法里。
```java
@Override
public void aiStep() {
    if (!level().isClientSide && isAlive()) { // 一定不能忘了只能在女巫存活的条件下在服务端执行这些逻辑
        healRaidersGoal.decrementCooldown();
        if (healRaidersGoal.getCooldown() <= 0) {
            attackPlayersGoal.setCanAttack(true);
        } else {
            attackPlayersGoal.setCanAttack(false);
        }
        // ----------
        // 此处还有代码，主要是上文中能力1的实现，内容比较长，因此暂时省略，等一下再讲
        // ----------
    }
    super.aiStep();
}
```
这下便可以发现女巫的两个AI中一个有冷却，另一个需要手动激活。此外，在`performRangedAttack`方法中，一旦女巫选择了治疗袭击者，就会在执行完选择药水的逻辑后马上重置攻击目标（具体代码下文会提到），这就可以解释女巫为什么不会一直治疗袭击者。同时能发现女巫在治疗完袭击者的短暂时间内，不能再选取玩家作为攻击目标。  

选择好了要攻击的目标，接下来到了攻击目标的环节。这个`performRangedAttack`方法内容有些多，我们把它分解一下。  

首先要确保在不在喝药水的时候进行攻击。  
```java
@Override
public void performRangedAttack(LivingEntity target, float power) {
    if (!isDrinkingPotion()) {
        Vec3 movement = target.getDeltaMovement();
        double x = target.getX() + movement.x - this.getX();
        double y = target.getEyeY() - (double)1.1F - this.getY();
        double z = target.getZ() + movement.z - this.getZ();
        double distance = Math.sqrt(x * x + z * z);
        Potion potion = Potions.HARMING;
        if (target instanceof Raider) {
            if (target.getHealth() <= 4.0F) {
                potion = Potions.HEALING;
            } else {
                potion = Potions.REGENERATION;
            }
            setTarget(null);
        } else if (distance >= 8.0D && !target.hasEffect(MobEffects.MOVEMENT_SLOWDOWN)) {
            potion = Potions.SLOWNESS;
        } else if (target.getHealth() >= 8.0F && !target.hasEffect(MobEffects.POISON)) {
            potion = Potions.POISON;
        } else if (distance <= 3.0D && !target.hasEffect(MobEffects.WEAKNESS) && random.nextFloat() < 0.25F) {
            potion = Potions.WEAKNESS;
        }
        ThrownPotion thrownPotion = new ThrownPotion(level(), this);
        thrownPotion.setItem(PotionUtils.setPotion(new ItemStack(Items.SPLASH_POTION), potion));
        thrownPotion.setXRot(thrownPotion.getXRot() - -20.0F);
        thrownPotion.shoot(x, y + distance * 0.2D, z, 0.75F, 8.0F);
        if (!isSilent()) {
            level().playSound(null, getX(), getY(), getZ(), SoundEvents.WITCH_THROW, getSoundSource(), 1.0F, 0.8F + random.nextFloat() * 0.4F);
        }
        level().addFreshEntity(thrownPotion);
    }
}
```
然后说这个`if`里面的内容，可以被拆成“**计算坐标**”，“**选择药水**”，“**投掷药水**”三个部分。先看“计算坐标”的部分。

```java
Vec3 movement = target.getDeltaMovement();
double x = target.getX() + movement.x - getX();
double y = target.getEyeY() - (double) 1.1F - getY();
double z = target.getZ() + movement.z - getZ();
```
值得注意的是，这里获取了攻击目标的`deltaMovement`，并根据这个`deltaMovement`改变了攻击方向，避免因喷溅药水飞行需要时间而打不中运动的目标。  

再来看“选择药水”的部分。
```java
double distance = Math.sqrt(x * x + z * z);
// 默认使用瞬间伤害药水
Potion potion = Potions.HARMING;
// 需要注意以下使用过长的if-else语句不是一种好的编程习惯，会导致代码可读性差并且难以维护。下面将解释这一大堆if-else的含义~
if (target instanceof Raider) {
    // （1）如果攻击目标是袭击者，则一定选择治疗类型的药水，并在治疗完目标后重置攻击目标，避免反复治疗
    if (target.getHealth() <= 4.0F) {
        // 如果袭击者生命值低，则使用瞬间治疗药水
        potion = Potions.HEALING;
    } else {
        // 否则使用生命恢复药水
        potion = Potions.REGENERATION;
    }
    setTarget(null); // 重置攻击目标
} else if (distance >= 8.0D && !target.hasEffect(MobEffects.MOVEMENT_SLOWDOWN)) {
    // （2）如果不满足（1）所述的条件，并且与攻击目标的距离大于8，同时目标没有缓慢效果，则一定选择迟缓药水
    potion = Potions.SLOWNESS;
} else if (target.getHealth() >= 8.0F && !target.hasEffect(MobEffects.POISON)) {
    // （3）如果不满足（2）所述的条件，并且攻击目标的生命值大于8，同时目标没有中毒效果，则一定选择剧毒药水
    potion = Potions.POISON;
} else if (distance <= 3.0D && !target.hasEffect(MobEffects.WEAKNESS) && random.nextFloat() < 0.25F) {
    // （4）如果不满足（3）所述的条件，并且与攻击目标的距离小于3，同时目标没有虚弱效果，则有25%的概率选择虚弱药水
    potion = Potions.WEAKNESS;
}
```
这里使用了复杂的条件判断语句来决定应该投掷出的药水种类，当然用过长的if-else语句未必是最合适的方式。  

最后看“投掷药水”的部分。
```java
ThrownPotion thrownPotion = new ThrownPotion(level(), this);
thrownPotion.setItem(PotionUtils.setPotion(new ItemStack(Items.SPLASH_POTION), potion));
thrownPotion.setXRot(thrownPotion.getXRot() - -20.0F);
thrownPotion.shoot(x, y + distance * 0.2D, z, 0.75F, 8.0F);
if (!isSilent()) {
    level().playSound(null, getX(), getY(), getZ(), SoundEvents.WITCH_THROW, getSoundSource(), 1.0F, 0.8F + random.nextFloat() * 0.4F);
}
level().addFreshEntity(thrownPotion);
```
与投掷雪球不同的是，投掷药水时额外指定了喷溅药水的`xRot`，以确保投掷出的药水方向朝前。还要记得将`Potion`（决定了药水的成分）绑定到投掷物上。  

说完了能力1、2，能力3又是怎样实现的呢？可以从刚才省略的代码中找到答案。
```java
// 对应着前面提到的aiStep方法中省略的部分
if (isDrinkingPotion()) {
    // 这里成员变量usingTime代表着喝药水的剩余时间
    if (usingTime-- <= 0) {
        // 当“使用药水”的时间减少到0（即喝完药水）时应用药水的效果
        setUsingItem(false);
        ItemStack mainHandItem = getMainHandItem();
        setItemSlot(EquipmentSlot.MAINHAND, ItemStack.EMPTY);
        if (mainHandItem.is(Items.POTION)) {
            List<MobEffectInstance> effects = PotionUtils.getMobEffects(mainHandItem);
            if (effects != null) {
                // 遍历并应用药水中效果
                for (MobEffectInstance effect : effects) {
                    addEffect(new MobEffectInstance(effect));
                }
            }
        }
        // 因为喝完药水了，所以要移除之前添加的降低移速的Modifier
        getAttribute(Attributes.MOVEMENT_SPEED).removeModifier(SPEED_MODIFIER_DRINKING);
    }
} else {
    Potion potion = null;
    if (random.nextFloat() < 0.15F && isEyeInFluid(FluidTags.WATER) && !hasEffect(MobEffects.WATER_BREATHING)) {
        // （1）如果眼睛在水中且自身没有水下呼吸效果，则每刻有15%的可能选择水肺药水
        potion = Potions.WATER_BREATHING;
    } else if (random.nextFloat() < 0.15F && (isOnFire() || getLastDamageSource() != null && getLastDamageSource().is(DamageTypeTags.IS_FIRE)) && !hasEffect(MobEffects.FIRE_RESISTANCE)) {
        // （2）如果不满足（1）所述的条件，并且身处火焰上/上次受到的伤害是火焰伤害，同时自身没有防火效果，则每刻有15%的可能选择抗火药水
        potion = Potions.FIRE_RESISTANCE;
    } else if (random.nextFloat() < 0.05F && getHealth() < getMaxHealth()) {
        // （3）如果不满足（2）所述的条件，并且生命值不满，则每刻有5%的可能选择瞬间治疗药水
        potion = Potions.HEALING;
    } else if (random.nextFloat() < 0.5F && getTarget() != null && !hasEffect(MobEffects.MOVEMENT_SPEED) && getTarget().distanceToSqr(this) > 121.0D) {
        // （4）如果不满足（3）所述的条件，并且与攻击目标的距离大于11，同时自身没有速度效果，则每刻有50%的可能选择迅捷药水
        potion = Potions.SWIFTNESS;
    }
    if (potion != null) {
        // 如果选择了药水，则准备喝该药水
        setItemSlot(EquipmentSlot.MAINHAND, PotionUtils.setPotion(new ItemStack(Items.POTION), potion));
        usingTime = getMainHandItem().getUseDuration();
        setUsingItem(true);
        if (!isSilent()) {
            level().playSound(null, getX(), getY(), getZ(), SoundEvents.WITCH_DRINK, getSoundSource(), 1.0F, 0.8F + random.nextFloat() * 0.4F);
        }
        // 添加喝药水时降低移速的Modifier
        AttributeInstance speedAttribute = getAttribute(Attributes.MOVEMENT_SPEED);
        speedAttribute.removeModifier(SPEED_MODIFIER_DRINKING);
        speedAttribute.addTransientModifier(SPEED_MODIFIER_DRINKING);
    }
}
if (random.nextFloat() < 7.5E-4F) {
    // 从下方补充的handleEntityEvent方法可以得知，15号实体事件代表着生成女巫头上的魔法粒子效果
    level().broadcastEntityEvent(this, (byte) 15); 
}

// 补充这个handleEntityEvent方法
@Override
public void handleEntityEvent(byte id) {
    if (id == 15) {
        for (int i = 0; i < this.random.nextInt(35) + 10; ++i) {
            level().addParticle(ParticleTypes.WITCH, getX() + random.nextGaussian() * (double) 0.13F, getBoundingBox().maxY + 0.5D +  random.nextGaussian() * (double) 0.13F, getZ() + random.nextGaussian() * (double) 0.13F, 0.0D, 0.0D, 0.0D);
        }
    } else {
        super.handleEntityEvent(id);
    }
}
```
这里又一次使用了复杂的条件判断语句，来决定应该喝下的药水种类。

另外，此处通过`broadcastEntityEvent`与`handleEntityEvent`来实现服务端与客户端的数据同步，原版里这样的同步方式很常见，但是在**写模组时需要避免用这种方式同步数据**。首先我们有强大的`SimpleChannel`，其次这样做容易出现事件id冲突。  

女巫重写了`getDamageAfterMagicAbsorb`方法，来实现避免受到来自自身的伤害及对有`DamageTypeTags.WITCH_RESISTANT_TO`标签的（主要是魔法类）伤害减免85%。
```java
@Override
protected float getDamageAfterMagicAbsorb(DamageSource source, float amount) {
    amount = super.getDamageAfterMagicAbsorb(source, amount);
    if (source.getEntity() == this) {
        amount = 0.0F;
    }
    if (source.is(DamageTypeTags.WITCH_RESISTANT_TO)) {
        amount *= 0.15F;
    }
    return amount;
}
```

最后是与袭击相关的内容。
```java
// 该方法中的布尔值在该方法被调用时总是传入false，原版代码中也从未用到过这个值，因此暂不明确其作用
@Override
public void applyRaidBuffs(int nextWave, boolean alwaysFalse) {}

@Override
public boolean canBeLeader() {
   return false;
}

@Override
public SoundEvent getCelebrateSound() {
    return SoundEvents.WITCH_CELEBRATE;
}
```
女巫不能成为袭击的领导者，在参与袭击时不会获得任何buff，且在袭击失败（此处说的失败是指“玩家失败”）时会播放特有的庆祝声。  

还有个细节：女巫的眼睛的高度被单独指定为1.62。  
```java
@Override
protected float getStandingEyeHeight(Pose pose, EntityDimensions dimensions) {
    return 1.62F;
}
```
女巫的逻辑就分析到这里了哦。女巫的模型有个注意点，我们下次再说。