# 实战2 - 食尸鬼

本节的最后一部分是一个自定义的近战怪物的实例。

### 任务
**制作一种新的怪物——食尸鬼**

---

### 要求
1. 食尸鬼**攻击力是8**，只有**10的最大生命值**，但是其**护甲值是20**，且**盔甲韧性是8**  
2. 食尸鬼**受阳光照射会死亡**，否则每秒恢复1点生命    
3. 食尸鬼死亡后，击杀该食尸鬼的**玩家**会获得**10s的凋零II**状态效果（持续时间和区域难度无关）  
4. 食尸鬼在**50%生命值及以上**时会主动攻击**任何生物（包括其他食尸鬼）**，否则只会主动攻击**玩家**和**铁傀儡**（当然**任何时候都会还击攻击自己的生物**）  
5. 食尸鬼在**50%生命值及以上**时**造成120%的伤害**，在**50%生命值以下**时只**受到80%的伤害**
6. 食尸鬼**会游泳**，空闲时会**四处张望**并**随意走动**
7. 食尸鬼的音效、模型和材质可以任意设置  

---

### 参考步骤
先是`Ghoul`类（此处未展示音效部分）。
```java
public class Ghoul extends Monster {
    public Ghoul(EntityType<? extends Monster> type, Level level) {
        super(type, level);
    }

    public static AttributeSupplier.Builder createAttributes() {
        return createMonsterAttributes()
                .add(Attributes.MAX_HEALTH, 10)
                .add(Attributes.ATTACK_DAMAGE, 8)
                .add(Attributes.ARMOR, 20)
                .add(Attributes.ARMOR_TOUGHNESS, 8);
    }

    @Override
    protected void registerGoals() {
        super.registerGoals();
        goalSelector.addGoal(0, new FloatGoal(this));
        goalSelector.addGoal(2, new MeleeAttackGoal(this, 1, false));
        goalSelector.addGoal(5, new RandomLookAroundGoal(this));
        goalSelector.addGoal(6, new RandomStrollGoal(this, 1));
        targetSelector.addGoal(1, new HurtByTargetGoal(this));
        targetSelector.addGoal(2, new NearestAttackableTargetGoal<>(this, Player.class, true));
        targetSelector.addGoal(3, new NearestAttackableTargetGoal<>(this, IronGolem.class, true));
        targetSelector.addGoal(4, new GhoulTargetAllMobsGoal(this)); // 这个AI优先级较低，因为我们希望食尸鬼会优先攻击玩家，其次是铁傀儡，最后是其他的生物
    }

    @Override
    public void aiStep() {
        super.aiStep();
        if (isAlive()) {
            if (isSunBurnTick()) {
                kill();
            } else if (tickCount % 20 == 0) {
                heal(1);
            }
        }
    }

    public boolean isHighHealth() {
        return getHealth() >= getMaxHealth() * 0.5F;
    }
}
```
`Ghoul`类的内容不多，注意到这里定义了一个`isHighHealth`方法，为达到要求4和5提供便利，同时在`registerGoals`方法内（第17，19，20行）注册了3个AI，用来达到要求6。  

下面来看此处为了达到要求2而重写的`aiStep`方法。
```java
@Override
public void aiStep() {
    super.aiStep();
    if (isAlive()) {
        if (isSunBurnTick()) {
            kill();
        } else if (tickCount % 20 == 0) {
            heal(1);
        }
    }
}
```
此处用表达式`tickCount % 20 == 0`来判断是否需要回血。`tickCount`在实体被更新时会每tick自增一次，所以每20个游戏刻该表达式的值会有1刻为true。

然后是我们新写的AI——`GhoulTargetAllMobsGoal`。内容很简单，只是比原版的`NearestAttackableTargetGoal`多了一个判断。
```java
public class GhoulTargetAllMobsGoal extends NearestAttackableTargetGoal<Mob> {
    public GhoulTargetAllMobsGoal(Ghoul ghoul) {
        super(ghoul, Mob.class, true);
    }

    @Override
    public boolean canUse() {
        Ghoul ghoul = (Ghoul) mob;
        return ghoul.isHighHealth() && super.canUse();
    }
}
```
要求3、5可以用事件监听器来达到。下面是实战1中提到的`EntityEventListener`类里的新增内容。
```java
@SubscribeEvent
public static void onLivingDeath(LivingDeathEvent event) {
    if (event.getEntity() instanceof Ghoul) {
        DamageSource source = event.getSource();
        if (source.getEntity() instanceof Player playerKiller) {
            playerKiller.addEffect(new MobEffectInstance(MobEffects.WITHER, 20 * 10, 1));
        }
    }
}

@SubscribeEvent
public static void onLivingHurt(LivingHurtEvent event) {
    if (event.getSource().getEntity() instanceof Ghoul ghoul && ghoul.isHighHealth()) {
        event.setAmount(event.getAmount() * 1.2f);
    }
    if (event.getEntity() instanceof Ghoul ghoul && !ghoul.isHighHealth()) {
        event.setAmount(event.getAmount() * 0.8f);
    }
}
```
食尸鬼的注册以及模型与渲染就省略不写了（总体与强化僵尸差不多，当然如果你有好的idea，模型与渲染部分可以发挥自己的创意，结合一些渲染方面的教程自己完成~）

到此我们就做完了食尸鬼的全部内容（此处省略效果图）。当然在模组开发中，近战怪物在实现逻辑上都不会很复杂，而真正复杂的实体往往是那些有复杂行为的动物和会“法术”的怪物。

[源代码（`Ghoul`类）](https://github.com/Gravelord-Lych/polonium-ExampleMod/blob/main/src/main/java/lych/polonium/entity/monster/Ghoul.java)  
[源代码（`GhoulTargetAllMobsGoal`类）](https://github.com/Gravelord-Lych/polonium-ExampleMod/blob/main/src/main/java/lych/polonium/entity/monster/ai/GhoulTargetAllMobsGoal.java)

---

### 思考与练习  
- 能否让食尸鬼**不自相残杀**呢？  
- 能否让食尸鬼的生命恢复速度**随生命值减少而增大**？  
- 能否让凋灵**不攻击食尸鬼**？（提示：想想凋灵不攻击什么类型的生物）
- 能否让食尸鬼击杀其他生物时召唤新的食尸鬼？
- 能否使食尸鬼只能被阳光杀死？（也就是说，只要不被阳光照射，无论自己被攻击多少次都不会死亡）  
