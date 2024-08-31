# performRangedAttack - 远程攻击の基础

有请`performRangedAttack`登场！

```java
public interface RangedAttackMob {
    void performRangedAttack(LivingEntity target, float power);
}
```
`RangedAttackMob`类很简单，只有这个抽象方法（*可不要把它当函数式接口使用哦*）。这个方法中一共有2个参数，第一个参数是攻击目标，表示应该攻击谁，第二个参数是攻击的“力量”，使用较少，只在弓箭手型生物中被用来确定射出的箭的伤害。  

然后来看`RangedAttackGoal`的构造方法与全局变量：

```java
public class RangedAttackGoal extends Goal {
//  可以远程攻击的生物
    private final Mob mob;
    private final RangedAttackMob rangedAttackMob;
//  当前的攻击目标
    @Nullable
    private LivingEntity target;
//  攻击冷却时间（单位：tick），为0时表示生物正要攻击，为正时表示生物在等待攻击，为负时表示生物当前空闲
    private int attackTime = -1;
//  决定了生物远程攻击时的速度（生物远程攻击时的速度 = 基础移速 * speedModifier）
    private final double speedModifier;
//  表示“能看见攻击目标”的状态持续了多久（单位：tick，例如“连续5tick能无阻挡地看见目标”时该值为5）
    private int seeTime;
//  最短攻击间隔（单位：tick）
    private final int attackIntervalMin;
//  最长攻击间隔（单位：tick）
    private final int attackIntervalMax;
//  攻击半径（攻击范围所在圆的半径）
    private final float attackRadius;
//  攻击半径的平方
    private final float attackRadiusSqr;

//  这个重载的构造方法使用十分广泛，其中attackIntervalMin与attackIntervalMax相等，表示攻击间隔固定
    public RangedAttackGoal(RangedAttackMob mob, double speedModifier, int attackInterval, float attackRadius) {
        this(mob, speedModifier, attackInterval, attackInterval, attackRadius);
    }

    public RangedAttackGoal(RangedAttackMob mob, double speedModifier, int attackIntervalMin, int attackIntervalMax, float attackRadius) {
        if (!(mob instanceof LivingEntity)) {
            throw new IllegalArgumentException("ArrowAttackGoal requires Mob implements RangedAttackMob");
        } else {
            this.rangedAttackMob = mob;
            this.mob = (Mob) mob;
            this.speedModifier = speedModifier;
            this.attackIntervalMin = attackIntervalMin;
            this.attackIntervalMax = attackIntervalMax;
            this.attackRadius = attackRadius;
            this.attackRadiusSqr = attackRadius * attackRadius;
            setFlags(EnumSet.of(Goal.Flag.MOVE, Goal.Flag.LOOK));
        }
    }
}
```

这里为了避免使用泛型或进行强制类型转换，此处用两个被声明为不同类型的引用指向了AI所有者。构造方法里也对传入的RangedAttackMob进行了检查，~~防止你传个lambda进去（）~~。此外，不难发现攻击半径的平方被保存了，这是为了以后比较距离时用，至于为什么用攻击半径的平方，我们马上再说。  

接下来是`canUse`与`canContinueToUse`两件套：
```java
public boolean canUse() {
    LivingEntity currentTarget = mob.getTarget();
    if (currentTarget != null && currentTarget.isAlive()) {
        target = currentTarget;
        return true;
    } else {
        return false;
    }
}

public boolean canContinueToUse() {
    return canUse() || target.isAlive() && !mob.getNavigation().isDone();
}
```
代码很好理解，如果AI所有者有存活的攻击目标，就把`target`引用指向该目标，同时`canUse`返回`true`，否则自然返回`false`。而`canContinueToUse`加了一点点内容：如果已经保存的攻击目标还活着并且AI所有者仍在追击TA，那么`canContinueToUse`也返回`true`。

`requiresUpdateEveryTick`方法如下（这个方法总是返回`true`，因为与狡猾的敌人作战要求生物足够“机灵”）：
```java
public boolean requiresUpdateEveryTick() {
    return true;
}
```

`stop`方法也没什么可说的，无非就是初始化了一些参数而已：
```java
public void stop() {
    target = null;
    seeTime = 0;
    attackTime = -1;
}
```

重点来了！这个AI的核心部分！tick方法！
```java
public void tick() {
    double distSqr = mob.distanceToSqr(target.getX(), target.getY(), target.getZ());
    boolean canSeeTarget = mob.getSensing().hasLineOfSight(target);

//  如果该tick内能看见攻击目标，则seeTime自增一次
    if (canSeeTarget) {
        ++seeTime;
    } else {
        seeTime = 0;
    }

//  如果持续看到攻击目标超过0.25秒，停下，否则追逐目标
    if (!(distSqr > (double) attackRadiusSqr) && seeTime >= 5) {
        mob.getNavigation().stop();
    } else {
        mob.getNavigation().moveTo(target, speedModifier);
    }

//  把头转向攻击目标
    mob.getLookControl().setLookAt(target, 30.0F, 30.0F);

    if (--attackTime == 0) {
        if (!canSeeTarget) {
            return;
        }

//      上文中攻击“力量”由到攻击目标的距离决定
        float power = (float) Math.sqrt(distSqr) / attackRadius;
        float clampedPower = Mth.clamp(power, 0.1F, 1.0F);
//      performRangedAttack的调用位置
        rangedAttackMob.performRangedAttack(target, clampedPower);
        attackTime = Mth.floor(power * (float) (attackIntervalMax - attackIntervalMin) + (float) attackIntervalMin);
    } else if (attackTime < 0) {
//      如果没击中目标，则攻击冷却时间随自己到目标的距离增大而增加
        attackTime = Mth.floor(Mth.lerp(Math.sqrt(distSqr) / (double) attackRadius, (double) attackIntervalMin, (double) attackIntervalMax));
    }
}
```
该AI的核心行为在`tick`方法中被定义，实现的效果比较简单，就是若看不见目标，则追逐，与此同时若攻击冷却时间为0，则攻击。  

值得一提的是如何算“看见”了目标。顺着一路找下去，关键的部分是`Level`类中的`clip`方法，由于该方法极其复杂，要想详细讲述原理需要占用大量篇幅，所以这里只讲一讲它的用处：**计算一条射线在一定范围内与这个世界内方块的交点**。其唯一的`ClipContext`（旧称`RayTraceContext`）参数可以控制诸如“是否考虑与液体方块的交点”一类的属性。  

另外，判断弹射物是否击中实体时也用到了这个方法，这一点以后再讲。  

要使用`RangedAttackGoal`也很简单，只需要实例化这个类即可，就像这样：
```java
goalSelector.addGoal(2, new RangedAttackGoal(this, 1.0D, 60, 10.0F));
```

本节的内容就到此为止了，下一节我们将开始分析雪傀儡的实现。
