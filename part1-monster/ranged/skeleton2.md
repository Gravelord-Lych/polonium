# 骷髅的“聪明之处”

众所周知，骷髅会在有阳光时走向遮阳的地方，并且在射击时会躲避靠近的玩家，这也是许多玩家认为骷髅“聪明”的地方。那为什么骷髅有这样的行为呢？  

先来讲讲和躲避阳光有关的2个AI。  

首先是`RestrictSunGoal`：
```java
public class RestrictSunGoal extends Goal {
    private final PathfinderMob mob;

    public RestrictSunGoal(PathfinderMob mob) {
        this.mob = mob;
    }

    @Override
    public boolean canUse() {
        return mob.level().isDay() && mob.getItemBySlot(EquipmentSlot.HEAD).isEmpty() && GoalUtils.hasGroundPathNavigation(mob);
    }

    @Override
    public void start() {
        ((GroundPathNavigation) mob.getNavigation()).setAvoidSun(true);
    }

    @Override
    public void stop() {
        if (GoalUtils.hasGroundPathNavigation(mob)) {
            ((GroundPathNavigation) mob.getNavigation()).setAvoidSun(false);
        }
    }
}
```

（`GroundPathNavigation`）
```java
protected void trimPath() {
    super.trimPath();
    if (avoidSun) {
        if (level.canSeeSky(BlockPos.containing(mob.getX(), mob.getY() + 0.5D, mob.getZ()))) {
            return;
        }
        for (int i = 0; i < path.getNodeCount(); ++i) {
            Node node = path.getNode(i);
            if (level.canSeeSky(new BlockPos(node.x, node.y, node.z))) {
                path.truncateNodes(i); // 除去这个有被太阳晒到风险的路径点
                return;
            }
        }
    }
}

public void setAvoidSun(boolean avoidSun) {
    this.avoidSun = avoidSun;
}
```

这个AI让骷髅具有在白天不带头盔时“切短”路径的能力，防止骷髅走入会被太阳晒到的区域。具体是这样的过程：如果`avoidSun`为`true`，那么就遍历骷髅当前的路径点，把会被太阳晒到的一部分从骷髅的路径中除去。  

但单纯只有这个AI也不行，因为这个AI虽然会防止骷髅走进能被太阳晒到的地方，但如果可怜的骷髅就在太阳下怎么办呢？  

这时另一个AI，`FleeSunGoal`，就要发挥作用啦。
```java
public class FleeSunGoal extends Goal {
    protected final PathfinderMob mob;
    // 下面3个变量描述了“安全”位置的坐标
    private double wantedX;
    private double wantedY;
    private double wantedZ;
    //  决定了逃离阳光的速度（逃离阳光的速度 = 基础移速 * speedModifier）
    private final double speedModifier;
    private final Level level;
    
    public FleeSunGoal(PathfinderMob mob, double speedModifier) {
        this.mob = mob;
        this.speedModifier = speedModifier;
        this.level = mob.level();
        setFlags(EnumSet.of(Goal.Flag.MOVE));
    }
    
    @Override
    public boolean canUse() {
        if (mob.getTarget() != null) {
            return false;
        } else if (!level.isDay()) {
            return false;
        } else if (!mob.isOnFire()) {
            return false;
        } else if (!level.canSeeSky(mob.blockPosition())) {
            return false;
        } else {
            // 只当骷髅白天在阳光下燃烧，且不攻击其他目标时可用该AI
            return mob.getItemBySlot(EquipmentSlot.HEAD).isEmpty() && setWantedPos();
        }
    }
    
    protected boolean setWantedPos() {
        Vec3 hidePos = getHidePos();
        if (hidePos == null) {
            return false;
        } else {
            this.wantedX = hidePos.x;
            this.wantedY = hidePos.y;
            this.wantedZ = hidePos.z;
            return true;
        }
    }
    
    @Override
    public boolean canContinueToUse() {
        return !mob.getNavigation().isDone();
    }
    
    @Override
    public void start() {
        mob.getNavigation().moveTo(wantedX, wantedY, wantedZ, speedModifier);
    }

    @Nullable
    protected Vec3 getHidePos() {
        RandomSource random = mob.getRandom();
        BlockPos pos = mob.blockPosition();
        // 这里的hidePos（“安全”位置）也是通过1.2.1.5说过的重复尝试思想来寻找和确定的
        for (int i = 0; i < 10; ++i) {
            BlockPos randomPos = pos.offset(random.nextInt(20) - 10, random.nextInt(6) - 3, random.nextInt(20) - 10);
            // 对骷髅等大多数怪物而言，randomPos的亮度越亮，则getWalkTargetValue返回值越小。对动物则相反
            if (!level.canSeeSky(randomPos) && mob.getWalkTargetValue(randomPos) < 0.0F) {
                return Vec3.atBottomCenterOf(randomPos);
            }
        }
        return null;
    }
}
```
*注：分析源代码可知，对继承了`Monster`类的怪物而言，`getWalkTargetValue(BlockPos)`的返回值为$$\large \frac{8ab-120a-5b+60}{120-6b} $$，式中a为怪物所在维度的环境光照（0<=a<=1），b为传入方块位置的亮度（0<=b<=15）。特别地，当a=0时，该式子可化简为$$\large \frac{5(12-b)}{120-6b} $$。由该式可得，当骷髅位于主世界时，会随机选择不被太阳直射且亮度低于12的位置作为“安全”位置。*  

在`FleeSunGoal`中，我们通过在骷髅被太阳直射时，生成一个“安全”位置，并尝试移动到该位置来避免被灼伤。`RestrictSunGoal`与`FleeSunGoal`巧妙配合，前者防患于未然，后者“亡羊补牢”，及时止损，共同确保了骷髅的生命安全。  

然后是骷髅射击时躲避靠近的玩家的原理。这个我们可以在`RangedBowAttackGoal`里找到答案。`RangedBowAttackGoal`结构基本与`RangedAttackGoal`相同，但是`tick`方法处二者有一定区别。  

*注：后文中扫射（strafe）指骷髅等弓箭手远程攻击时以玩家为中心持续侧移绕圈以尝试规避攻击的行为*

下面展示一下有区别的`tick`方法。
```java
// 两个boolean表示扫射方向，其中strafingClockwise为左/右（顺时针/逆时针）方向，strafingBackwards为后/前方向
private boolean strafingClockwise;
private boolean strafingBackwards;
// 扫射时间，与扫射方向的调节有关，为-1时表示不在扫射状态
private int strafingTime = -1;

@Override
public void tick() {
    LivingEntity target = mob.getTarget();
    if (target != null) {
        double distSqr = mob.distanceToSqr(target.getX(), target.getY(), target.getZ());
        boolean hasLineOfSight = mob.getSensing().hasLineOfSight(target);
        boolean hasSawTarget = seeTime > 0;
        if (hasLineOfSight != hasSawTarget) {
            seeTime = 0;
        }
        if (hasLineOfSight) {
            ++seeTime;
        } else {
            --seeTime;
        }
        if (!(distSqr > (double) attackRadiusSqr) && seeTime >= 20) {
            mob.getNavigation().stop();
            ++strafingTime;
        } else {
            mob.getNavigation().moveTo(target, speedModifier);
            strafingTime = -1;
        }
        if (strafingTime >= 20) {
            if ((double) mob.getRandom().nextFloat() < 0.3D) {
                strafingClockwise = !strafingClockwise;
            }
            if ((double) mob.getRandom().nextFloat() < 0.3D) {
                strafingBackwards = !strafingBackwards;
            }
            strafingTime = 0;
        }
        if (strafingTime > -1) {
            if (distSqr > (double) (attackRadiusSqr * 0.75F)) {
                strafingBackwards = false;
            } else if (distSqr < (double) (attackRadiusSqr * 0.25F)) {
                strafingBackwards = true;
            }
            mob.getMoveControl().strafe(strafingBackwards ? -0.5F : 0.5F, strafingClockwise ? 0.5F : -0.5F);
            Entity vehicle = mob.getControlledVehicle();
            if (vehicle instanceof Mob) {
                Mob mobVehicle = (Mob) vehicle;
                mobVehicle.lookAt(target, 30.0F, 30.0F);
            }
            mob.lookAt(target, 30.0F, 30.0F);
        } else {
            mob.getLookControl().setLookAt(target, 30.0F, 30.0F);
        }
        if (mob.isUsingItem()) {
            if (!hasLineOfSight && seeTime < -60) {
                mob.stopUsingItem();
            } else if (hasLineOfSight) {
                int ticksUsingItem = mob.getTicksUsingItem();
                if (ticksUsingItem >= 20) {
                    mob.stopUsingItem();
                    mob.performRangedAttack(target, BowItem.getPowerForTime(ticksUsingItem));
                    attackTime = attackIntervalMin;
                }
            }
        } else if (--attackTime <= 0 && seeTime >= -60) {
            mob.startUsingItem(ProjectileUtil.getWeaponHoldingHand(mob, item -> item instanceof BowItem));
        }
    }
}
```
我们将`if (target != null) {...}`里的内容拆解为4部分。  

第一部分用来更新`seeTime`。
```java
double distSqr = mob.distanceToSqr(target.getX(), target.getY(), target.getZ());
boolean hasLineOfSight = mob.getSensing().hasLineOfSight(target);
boolean hasSawTarget = seeTime > 0;
if (hasLineOfSight != hasSawTarget) {
    seeTime = 0;
}
if (hasLineOfSight) {
    ++seeTime;
} else {
    --seeTime;
}
```
这部分与`RangedAttackGoal`是相似的，但是如果`seeTime`到0后仍未看到目标，`seeTime`会继续下降至负值（负值的绝对值表示没有看到目标的时间）。  

第二部分用来更新3个与扫射有关的变量。
```java
if (!(distSqr > (double) attackRadiusSqr) && seeTime >= 20) {
    // 如果看到目标超过1秒，并且目标在攻击范围内，则准备进行扫射
    mob.getNavigation().stop();
    ++strafingTime;
} else {
    // 否则移向目标，并把strafingTime重置为-1
    mob.getNavigation().moveTo(target, speedModifier);
    strafingTime = -1;
}
// 当扫射持续1秒未变向时，每刻都有0.3的概率改变扫射方向（左右方向/前后方向）
if (strafingTime >= 20) {
    if ((double) mob.getRandom().nextFloat() < 0.3D) {
        strafingClockwise = !strafingClockwise;
    }
    if ((double) mob.getRandom().nextFloat() < 0.3D) {
        strafingBackwards = !strafingBackwards;
    }
    // 把strafingTime重置为0，注意这里不是重置为-1，因为此时正在扫射目标，而-1表示不处于扫射状态
    strafingTime = 0;
}
```
注意到这部分随机化了扫射方向，防止一直向同一方向扫射。

第三部分根据之前更新过的扫射变量更新了射击者自身的移动方向，还调整了射击者及其坐骑的头部朝向，使他/她/它们看向目标。
```java
if (strafingTime > -1) {
    // 对扫射的前后方向做一些必要的调整，以防射击者走出其射程
    if (distSqr > (double) (attackRadiusSqr * 0.75F)) {
        strafingBackwards = false;
    } else if (distSqr < (double) (attackRadiusSqr * 0.25F)) {
        strafingBackwards = true;
    }
    // 两个参数的绝对值大小与速度大小有关，正负与速度方向有关（第一个参数正为向前，负为向后；第二个参数正为向右，负为向左）
    mob.getMoveControl().strafe(strafingBackwards ? -0.5F : 0.5F, strafingClockwise ? 0.5F : -0.5F);
    Entity vehicle = mob.getControlledVehicle();
    if (vehicle instanceof Mob) {
        Mob mobVehicle = (Mob) vehicle;
        mobVehicle.lookAt(target, 30.0F, 30.0F);
    }
    mob.lookAt(target, 30.0F, 30.0F);
} else {
    mob.getLookControl().setLookAt(target, 30.0F, 30.0F);
}
```

第四部分则准备进行攻击。
```java
if (mob.isUsingItem()) {
    // 没看到目标3秒以上，则放下手中的弓
    if (!hasLineOfSight && seeTime < -60) {
        mob.stopUsingItem();
    } else if (hasLineOfSight) {
        int ticksUsingItem = mob.getTicksUsingItem();
        // 使用弓的时长达到1秒以上，则发起攻击
        if (ticksUsingItem >= 20) {
            mob.stopUsingItem();
            mob.performRangedAttack(target, BowItem.getPowerForTime(ticksUsingItem));
            attackTime = attackIntervalMin;
        }
    }
// 没看到目标的时间不足3秒，且经过了攻击间隔而可以攻击，则开始张弓
} else if (--attackTime <= 0 && seeTime >= -60) {
    mob.startUsingItem(ProjectileUtil.getWeaponHoldingHand(mob, item -> item instanceof BowItem));
}
```

除`tick`方法外`RangedBowAttackGoal`中的内容基本上都是`RangedAttackGoal`中出现过的，此处省略不再次讲述。  

下一节是骷髅的模型与渲染哦，我们下次再见~