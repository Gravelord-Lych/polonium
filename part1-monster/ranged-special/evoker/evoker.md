# 唤魔者的基本实现逻辑

唤魔者看上去很复杂，但实际上`Evoker`类（除去里面所有的内部类）内容十分少。这是因为唤魔者的许多行为继承了灾厄村民的行为，而唤魔者的独有行为除去其特有的AI外很少。  

照例先看成员变量和构造方法。
```java
// 正因唤魔者施法而改变颜色的绵羊（唤魔者会施法改变绵羊的颜色）
@Nullable
private Sheep wololoTarget;

public Evoker(EntityType<? extends Evoker> type, Level level) {
    super(type, level);
    this.xpReward = 10; // 唤魔者掉落一般怪物的2倍的经验（10 instead of 5）
}
```

再看属性注册部分。  
```java
public static AttributeSupplier.Builder createAttributes() {
    return Monster.createMonsterAttributes()
        .add(Attributes.MOVEMENT_SPEED, 0.5D)
        .add(Attributes.FOLLOW_RANGE, 12.0D)
        .add(Attributes.MAX_HEALTH, 24.0D);
}
```
可以观察到唤魔者的追踪距离（即`Attributes.FOLLOW_RANGE`的值）与许多灾厄村民一样很短，但是唤魔者的速度达到了惊人的0.5。然而游戏中唤魔者四处闲逛时，移动速度似乎也不是很快，这是为什么呢？  

答案藏在唤魔者的AI中。  
```java
@Override
protected void registerGoals() {
    super.registerGoals();
    goalSelector.addGoal(0, new FloatGoal(this));
    goalSelector.addGoal(1, new Evoker.EvokerCastingSpellGoal());
    goalSelector.addGoal(2, new AvoidEntityGoal<>(this, Player.class, 8.0F, 0.6D, 1.0D));
    // 下面三个AI是唤魔者特有的施法AI，笔者将在1.2.3.2.3中分析它们
    goalSelector.addGoal(4, new Evoker.EvokerSummonSpellGoal());
    goalSelector.addGoal(5, new Evoker.EvokerAttackSpellGoal());
    goalSelector.addGoal(6, new Evoker.EvokerWololoSpellGoal());
    goalSelector.addGoal(8, new RandomStrollGoal(this, 0.6D));
    goalSelector.addGoal(9, new LookAtPlayerGoal(this, Player.class, 3.0F, 1.0F));
    goalSelector.addGoal(10, new LookAtPlayerGoal(this, Mob.class, 8.0F));
    targetSelector.addGoal(1, new HurtByTargetGoal(this, Raider.class).setAlertOthers());
    targetSelector.addGoal(2, new NearestAttackableTargetGoal<>(this, Player.class, true).setUnseenMemoryTicks(300));
    targetSelector.addGoal(3, new NearestAttackableTargetGoal<>(this, AbstractVillager.class, false).setUnseenMemoryTicks(300));
    targetSelector.addGoal(3, new NearestAttackableTargetGoal<>(this, IronGolem.class, false));
}
```
可以发现`AvoidEntityGoal`（这个AI在1.2.2.5.1中提到过）的`walkSpeedModifier`以及`RandomStrollGoal`的`speedModifier`都只有0.6，也就是说唤魔者许多时候只会以60%的移动速度移动，因此唤魔者看上去移动得并不快（尽管唤魔者试图逃离近战的玩家时可以全速逃跑）。

一个需要关注的方法是`isAlledTo`方法。
```java
@Override
public boolean isAlliedTo(Entity entity) {
    if (entity == null) {
        return false;
    } else if (entity == this) {
        return true;
    } else if (super.isAlliedTo(entity)) {
        return true;
    } else if (entity instanceof Vex) {
        return isAlliedTo(((Vex) entity).getOwner());
    } else if (entity instanceof LivingEntity && ((LivingEntity) entity).getMobType() == MobType.ILLAGER) {
        // 当唤魔者与另一灾厄村民都没有所属的队伍时，它们是“一伙的”
        return getTeam() == null && entity.getTeam() == null;
    } else {
        return false;
    }
}
```
由于唤魔者尖牙不应该伤害到与召唤它们的唤魔者属于同一阵营的生物【例如自己召唤的恼鬼、其他灾厄村民（仅在唤魔者没有所属队伍时适用）】，需要在`Evoker`类中重写`isAlliedTo`方法并在`EvokerFangs`类中对攻击到的生物进行`isAlliedTo`的判断。  

`Evoker`类中的其他方法为`wololoTarget`的`getter`和`setter`，决定音效的方法（如`getCelebrateSound`等方法），还有一些笔者认为没有太大意义的方法重写（如`protected void defineSynchedData() { super.defineSynchedData(); }`），此处就不展开了。

下面两节将会介绍唤魔者特有的施法AI，第一节介绍基本框架，第二节介绍具体逻辑。