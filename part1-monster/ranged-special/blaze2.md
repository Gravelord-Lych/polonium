# 烈焰人小火球的“三连发”

烈焰人每次攻击时，以较短的时间间隔连续发射3个小火球射击目标。这个攻击方式的实现位于烈焰人的一个AI`Blaze.BlazeAttackGoal`中。

这个AI类中定义了这样几个`int`类型的成员变量。

```java
// 攻击阶段（可能取值为0，1，2，3，4，5）
private int attackStep;
// 攻击冷却时间（单位：tick）
private int attackTime;
// 距离上次看到攻击目标过去的时间（单位：tick），如果烈焰人能看到目标则该变量的值为0
private int lastSeen;
```

构造方法中为AI设置了`Flag.MOVE`和`Flag.LOOK`两个`Flag`，一般控制生物攻击的AI都会设置这两个`Flag`。
```java
public BlazeAttackGoal(Blaze blaze) {
    this.blaze = blaze;
    setFlags(EnumSet.of(Goal.Flag.MOVE, Goal.Flag.LOOK));
}
```

然后是AI使用的条件与开始结束时的行为。
```java
@Override
public boolean canUse() {
    LivingEntity entity = blaze.getTarget();
    return entity != null && entity.isAlive() && blaze.canAttack(entity);
}

@Override
public void start() {
    // 重置attackStep，意味着开始新的的一轮攻击
    attackStep = 0;
}

@Override
public void stop() {
    // 让烈焰人身上的火熄灭
    blaze.setCharged(false);
    lastSeen = 0;
}
```

这个AI每个游戏刻都需要更新（包括前面说过的`RangedAttackGoal`在内的几乎所有控制生物攻击的AI也是这样），所以重写`requiresUpdateEveryTick`方法。
```java
@Override
public boolean requiresUpdateEveryTick() {
    return true;
}
```

下面说的`tick`方法是这个AI的关键。
```java
@Override
public void tick() {
    --attackTime;
    LivingEntity target = blaze.getTarget();
    if (target != null) {
        boolean canSeeTarget = blaze.getSensing().hasLineOfSight(target);
        if (canSeeTarget) {
            lastSeen = 0;
        } else {
            ++lastSeen;
        }

        double distSqrToTarget = blaze.distanceToSqr(target);
        if (distSqrToTarget < 4.0D) {
            if (!canSeeTarget) {
                return;
            }
            if (attackTime <= 0) {
                attackTime = 20;
                blaze.doHurtTarget(target);
            }
            blaze.getMoveControl().setWantedPosition(target.getX(), target.getY(), target.getZ(), 1.0D);
        } else if (distSqrToTarget < getFollowDistance() * getFollowDistance() && canSeeTarget) {
            double dx = target.getX() - blaze.getX();
            double dy = target.getY(0.5D) - blaze.getY(0.5D);
            double dz = target.getZ() - blaze.getZ();
        
            if (attackTime <= 0) {
                ++attackStep;
                if (attackStep == 1) {
                    attackTime = 60;
                    blaze.setCharged(true);
                } else if (attackStep <= 4) {
                    attackTime = 6;
                } else {
                    attackTime = 100;
                    attackStep = 0;
                    blaze.setCharged(false);
                }
        
                if (attackStep > 1) {
                    double horizontalOffset = Math.sqrt(Math.sqrt(distSqrToTarget)) * 0.5D;
                    if (!blaze.isSilent()) {
                        blaze.level().levelEvent(null, 1018, blaze.blockPosition(), 0);
                    }
                    for (int i = 0; i < 1; ++i) {
                        SmallFireball fireball = new SmallFireball(blaze.level(),
                                blaze,
                                blaze.getRandom().triangle(dx, 2.297D * horizontalOffset),
                                dy,
                                blaze.getRandom().triangle(dz, 2.297D * horizontalOffset));
                        fireball.setPos(fireball.getX(), blaze.getY(0.5D) + 0.5D, fireball.getZ());
                        blaze.level().addFreshEntity(fireball);
                    }
                }
            }
            blaze.getLookControl().setLookAt(target, 10.0F, 10.0F);
        } else if (lastSeen < 5) {
            blaze.getMoveControl().setWantedPosition(target.getX(), target.getY(), target.getZ(), 1.0D);
        }

        // Goal类的tick方法的方法体是空的，因此下面调用super.tick()其实是非必要的。下文中分段分析时不再提及这个方法。
        super.tick();
    }
}

private double getFollowDistance() {
    return blaze.getAttributeValue(Attributes.FOLLOW_RANGE);
}
```

与之前分析较复杂AI的方式一样，我们将`if (target != null) {...}`里的内容分为4个部分来分析。  

第一部分用来更新`attackTime`和`lastSeen`。这两个变量的作用刚才提到过，因此此处不再赘述。   
```java
--attackTime;
boolean canSeeTarget = blaze.getSensing().hasLineOfSight(target);
if (canSeeTarget) {
    lastSeen = 0;
} else {
    ++lastSeen;
}
```

第二部分及以后的部分用来根据烈焰人与目标的位置关系调整烈焰人的行为。其中第二部分控制烈焰人在与目标的距离小于2时尝试近战攻击目标，并且每秒可以攻击一次。
```java
double distSqrToTarget = blaze.distanceToSqr(target);
if (distSqrToTarget < 4.0D) {
    // 仅在能看见目标的条件下发动攻击
    if (!canSeeTarget) {
        return;
    }
    if (attackTime <= 0) {
        attackTime = 20;
        // 发动近战攻击
        blaze.doHurtTarget(target);
    }
    // 使烈焰人移向目标
    blaze.getMoveControl().setWantedPosition(target.getX(), target.getY(), target.getZ(), 1.0D);
}
```

如果不满足近战攻击的条件，烈焰人会使用“小火球三连发”。第三部分中则实现了这样子的三连发小火球。
```java
else if (distSqrToTarget < getFollowDistance() * getFollowDistance() && canSeeTarget) {
    double dx = target.getX() - blaze.getX();
    double dy = target.getY(0.5D) - blaze.getY(0.5D);
    double dz = target.getZ() - blaze.getZ();
        
    if (attackTime <= 0) {
        ++attackStep;
        // attackStep为1，准备远程攻击
        if (attackStep == 1) {
            attackTime = 60;
            blaze.setCharged(true);
        } else if (attackStep <= 4) {
            attackTime = 6; // 每6游戏刻（0.3s）攻击一次
        } else {
            // 等待5s再发动下一轮攻击
            attackTime = 100;
            // 重置attackStep
            attackStep = 0;
            blaze.setCharged(false);
        }
        
        if (attackStep > 1) {
            // 为火球设置一个偏移量，这个偏移量的最大值与烈焰人到攻击目标距离的算术平方根成正比
            double horizontalOffset = Math.sqrt(Math.sqrt(distSqrToTarget)) * 0.5D;
            if (!blaze.isSilent()) {
                // 播放烈焰人发射火球的声音
                blaze.level().levelEvent(null, 1018, blaze.blockPosition(), 0);
            }
            // 其实我也很好奇为什么这里要用到for循环（我认为这里完全不需要循环）
            for (int i = 0; i < 1; ++i) {
                SmallFireball fireball = new SmallFireball(blaze.level(),
                        blaze,
                        blaze.getRandom().triangle(dx, 2.297D * horizontalOffset),
                        dy,
                        blaze.getRandom().triangle(dz, 2.297D * horizontalOffset));
                // 把小火球的y坐标移动到烈焰人中心的y坐标上方0.5格
                fireball.setPos(fireball.getX(), blaze.getY(0.5D) + 0.5D, fireball.getZ());
                blaze.level().addFreshEntity(fireball);
            }
        }
    }

    // 让烈焰人看向目标，注意由于“烈焰人的头不太灵活，不能大角度旋转”，所以yRot和xRot旋转量的最大值都只设为了10（单位为角度）
    //
    // 说到yRot和xRot（zRot不常用，只有鱿鱼等少数生物需要不断调整自身的zRot），就顺带提一下在控制实体逻辑的类（如Entity）中这几个rot
    // 一般用角度表示，而在控制实体模型的类（如ModelPart）中则用弧度表示。因此在开发过程中需要注意角度与弧度间的转换
    blaze.getLookControl().setLookAt(target, 10.0F, 10.0F);
}
```
先说一下`attackStep`的具体作用。  

| attackStep的值 | 对应的烈焰人的行为 |
|:--------------:|:------------------:|
|        0       |   非远程攻击状态   |
|        1       |    准备远程攻击    |
|       2~4      |     发射小火球     |
|        5       |    结束远程攻击    |

当烈焰人处于可以远程攻击的状态时，每当`attackTime`为0时都会自增`attackStep`。在`attackStep`被设为1时，烈焰人会使自身着火，并在60tick（3s）之后开始发射小火球。每次发射小火球后6tick（0.3s）再次发射小火球，共重复3次这个过程。最后当`attackStep`增大到5时，烈焰人的所有攻击会进入100tick（5s）的冷却。  

另外注意如果远程攻击时攻击目标距离自己过近（距离<2），烈焰人会暂时“暂停”远程攻击流程而去近战攻击该攻击目标。  

下面来说让生物发射小火球等继承了`AbstractHurtingProjectile`的弹射物的具体方式。  

计算发射方向的一步与发射雪球等继承了`ThrowableProjectile`的弹射物以及各种箭是一样的。与之前提到过的雪傀儡不同，烈焰人发射火球前计算y坐标的差值用的是攻击目标碰撞箱中心的y坐标减去烈焰人碰撞箱中心的y坐标，这是为了使小火球尽量击中目标碰撞箱中部。

```java
double dx = target.getX() - blaze.getX();
double dy = target.getY(0.5D) - blaze.getY(0.5D);
double dz = target.getZ() - blaze.getZ();
```

`Entity`：
```java
public double getY(double scale) {
    return position.y + (double) getBbHeight() * scale;
}

// getX(double), getZ(double)两个方法是类似的，只不过后面的getBbHeight()被替换为了getBbWidth()。这儿一并给出吧~
public double getX(double scale) {
    return position.x + (double) getBbWidth() * scale;
}

public double getZ(double scale) {
    return position.z + (double) getBbWidth() * scale;
}
```

下面则是实例化弹射物。这里我们在计算发射方向后才实例化了小火球，是因为我们要向小火球的构造方法中传入发射小火球的方向（对于下面调用的这个构造方法而言，小火球被实例化后会移动到烈焰人的坐标处，并以向量(`blaze.getRandom().triangle(dx, 2.297D * horizontalOffset)`, `dy`, `blaze.getRandom().triangle(dz, 2.297D * horizontalOffset)`)为其弹道的方向向量）。还有小火球等继承了`AbstractHurtingProjectile`的弹射物都是不受重力影响的。
```java
SmallFireball fireball = new SmallFireball(blaze.level(),
                        blaze,
                        blaze.getRandom().triangle(dx, 2.297D * horizontalOffset),
                        dy,
                        blaze.getRandom().triangle(dz, 2.297D * horizontalOffset));
// 把小火球的y坐标移动到烈焰人中心的y坐标上方0.5格
fireball.setPos(fireball.getX(), blaze.getY(0.5D) + 0.5D, fireball.getZ());
blaze.level().addFreshEntity(fireball);
```
但是我们把小火球直接移动到烈焰人的“脚”（指烈焰人碰撞箱左下角~~，烈焰人哪有脚？~~）上显然不合适，所以要把火球移上去（增大其y坐标）一点。  

最后添加小火球到世界中，这样就能看到烈焰人的小火球射向攻击目标啦！  

回到烈焰人的AI。`tick`方法的第四部分用于在近、远程攻击的条件都不满足，但是5tick（0.25s）内曾看到过攻击目标的条件下，让烈焰人以1倍速移向攻击目标。
```java
else if (lastSeen < 5) {
    blaze.getMoveControl().setWantedPosition(target.getX(), target.getY(), target.getZ(), 1.0D);
}
```

这样烈焰人的AI就差不多讲完了。  

你一定很好奇烈焰人的模型是怎样制作出来的，以及烈焰人为什么在完全黑暗时身体也是亮着的吧233，下一节我们就来研究这些内容。
