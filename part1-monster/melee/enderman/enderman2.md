# 末影人的AI

接下来到了末影人的AI。因为末影人的行为较简单，所以用`Goal`系统已经足够。因为这一节属于末影人部分，所以本节的内容以分析末影人**特有**的AI为主，其余的通用AI以后找机会再讲吧~  

下面来看`registerGoals`方法：  
```java
@Override
protected void registerGoals() {
//  FloatGoal旧称SwimGoal，游泳的AI。
    goalSelector.addGoal(0, new FloatGoal(this));
    goalSelector.addGoal(1, new EnderMan.EndermanFreezeWhenLookedAt(this));
    goalSelector.addGoal(2, new MeleeAttackGoal(this, 1.0D, false));
    goalSelector.addGoal(7, new WaterAvoidingRandomStrollGoal(this, 1.0D, 0.0F));
    goalSelector.addGoal(8, new LookAtPlayerGoal(this, Player.class, 8.0F));
    goalSelector.addGoal(8, new RandomLookAroundGoal(this));
    goalSelector.addGoal(10, new EnderMan.EndermanLeaveBlockGoal(this));
    goalSelector.addGoal(11, new EnderMan.EndermanTakeBlockGoal(this));
    targetSelector.addGoal(1, new EnderMan.EndermanLookForPlayerGoal(this, this::isAngryAt));
    targetSelector.addGoal(2, new HurtByTargetGoal(this));
    targetSelector.addGoal(3, new NearestAttackableTargetGoal<>(this, Endermite.class, true, false));
    targetSelector.addGoal(4, new ResetUniversalAngerTargetGoal<>(this, false));
}
```
注意以下一部分实例化了静态内部类：
```java
@Override
protected void registerGoals() {
    goalSelector.addGoal(1, new EnderMan.EndermanFreezeWhenLookedAt(this));
    goalSelector.addGoal(10, new EnderMan.EndermanLeaveBlockGoal(this));
    goalSelector.addGoal(11, new EnderMan.EndermanTakeBlockGoal(this));
    targetSelector.addGoal(1, new EnderMan.EndermanLookForPlayerGoal(this, this::isAngryAt));
}
```
我们先看`EndermanFreezeWhenLookedAt`这个类。
```java
static class EndermanFreezeWhenLookedAt extends Goal {
    private final EnderMan enderman;
    @Nullable
    private LivingEntity target;

    public EndermanFreezeWhenLookedAt(EnderMan enderman) {
        this.enderman = enderman;
        setFlags(EnumSet.of(Goal.Flag.JUMP, Goal.Flag.MOVE));
    }

    @Override
    public boolean canUse() {
        target = enderman.getTarget();
        if (!(target instanceof Player)) {
            return false;
        } else {
            double disSqr = target.distanceToSqr(enderman);
            return !(disSqr > 256.0D) && enderman.isLookingAtMe((Player) target);
        }
    }

    @Override
    public void start() {
        enderman.getNavigation().stop();
    }

    @Override
    public void tick() {
        enderman.getLookControl().setLookAt(target.getX(), target.getEyeY(), target.getZ());
    }

//  P.S. canContinueToUse省略了，意味着返回值与canUse相同
}
```
_提示：构造方法内的`setFlags(EnumSet<Goal.Flag>)`很容易遗漏，在设计生物的AI时需特别注意**是否需要添加`Flag`**，以防在禁用或启用`Flag`时（例如在被拴绳牵引时`Flag.MOVE`会被禁用）生物出现意料之外的问题_  

这个类比较简单，目的就是为了让末影人可以在攻击目标在不远处（16格内），且攻击目标正看向自己时看向其攻击目标。另外，这个AI的实时性不高，不需要每tick更新。  

接下来是两个与方块操作相关的AI，先给出完整的代码。   
```java
static class EndermanLeaveBlockGoal extends Goal {
    private final EnderMan enderman;

    public EndermanLeaveBlockGoal(EnderMan enderman) {
        this.enderman = enderman;
    }

    @Override
    public boolean canUse() {
        if (enderman.getCarriedBlock() == null) {
            return false;
        } else if (!ForgeEventFactory.getMobGriefingEvent(enderman.level(), enderman)) {
            return false;
        } else {
        //  因为这是个每两刻更新的AI，所以必须调用reducedTickDelay（或adjustedTickDelay）方法，下同
        //  执行的概率是1/2000每刻
            return enderman.getRandom().nextInt(reducedTickDelay(2000)) == 0;
        }
    }

    @Override
    public void tick() {
        RandomSource random = enderman.getRandom();
        Level level = enderman.level();

        int x = Mth.floor(enderman.getX() - 1.0D + random.nextDouble() * 2.0D);
        int y = Mth.floor(enderman.getY() + random.nextDouble() * 2.0D);
        int z = Mth.floor(enderman.getZ() - 1.0D + random.nextDouble() * 2.0D);

        BlockPos pos = new BlockPos(x, y, z);
        BlockState currentState = level.getBlockState(pos);
        BlockPos belowPos = pos.below();
        BlockState belowState = level.getBlockState(belowPos);
        BlockState carriedState = enderman.getCarriedBlock();

        if (carriedState != null) {
        //  根据周围方块状态更新自己的方块状态
            carriedState = Block.updateFromNeighbourShapes(carriedState, enderman.level(), pos);
            if (canPlaceBlock(level, pos, carriedState, currentState, belowState, belowPos) && !ForgeEventFactory.onBlockPlace(enderman, BlockSnapshot.create(level.dimension(), level, belowPos), Direction.UP)) {
            //  放置方块
                level.setBlock(pos, carriedState, 3);
                level.gameEvent(GameEvent.BLOCK_PLACE, pos, GameEvent.Context.of(this.enderman, carriedState));
                enderman.setCarriedBlock(null);
            }
        }
    }

    private boolean canPlaceBlock(Level level, BlockPos pos, BlockState carriedBlock, BlockState currentState, BlockState belowState, BlockPos belowPos) {
        return currentState.isAir() // 当前方块状态是空气
                && !belowState.isAir() // 下方方块状态不是空气
                && !belowState.is(Blocks.BEDROCK)  // 下方方块不是基岩
                && !belowState.is(Tags.Blocks.ENDERMAN_PLACE_ON_BLACKLIST) // 下方方块无ENDERMAN_PLACE_ON_BLACKLIST标签
                && belowState.isCollisionShapeFullBlock(level, belowPos) // 下方方块是完整的
                && carriedBlock.canSurvive(level, pos) // 末影人手上拿着的方块可以放在pos处
                && level.getEntities(enderman, AABB.unitCubeFromLowerCorner(Vec3.atLowerCornerOf(pos))).isEmpty(); // 放置方块的位置没有实体
    }
}

static class EndermanTakeBlockGoal extends Goal {
    private final EnderMan enderman;

    public EndermanTakeBlockGoal(EnderMan enderman) {
        this.enderman = enderman;
    }

    @Override
    public boolean canUse() {
        if (enderman.getCarriedBlock() != null) {
            return false;
        } else if (!ForgeEventFactory.getMobGriefingEvent(enderman.level(), enderman)) {
            return false;
        } else {
        //  执行的概率是1/20每刻
            return enderman.getRandom().nextInt(reducedTickDelay(20)) == 0;
        }
    }

    @Override
    public void tick() {
        RandomSource random = enderman.getRandom();
        Level level = enderman.level();
        int x = Mth.floor(enderman.getX() - 2.0D + random.nextDouble() * 4.0D);
        int y = Mth.floor(enderman.getY() + random.nextDouble() * 3.0D);
        int z = Mth.floor(enderman.getZ() - 2.0D + random.nextDouble() * 4.0D);

        BlockPos placeBlockPos = new BlockPos(x, y, z);
        BlockState state = level.getBlockState(placeBlockPos);
        Vec3 myPos = new Vec3(enderman.getBlockX() + 0.5D, y + 0.5D, enderman.getBlockZ() + 0.5D);
        Vec3 placePos = new Vec3(x + 0.5D, y + 0.5D, z + 0.5D);

    //  HitResult很重要，马上会讲
        BlockHitResult res = level.clip(new ClipContext(myPos, placePos, ClipContext.Block.OUTLINE, ClipContext.Fluid.NONE, enderman));
        boolean notBlocked = res.getBlockPos().equals(placeBlockPos);

        if (state.is(BlockTags.ENDERMAN_HOLDABLE) && notBlocked) {
        //  搬起方块
            level.removeBlock(placeBlockPos, false);
            level.gameEvent(GameEvent.BLOCK_DESTROY, placeBlockPos, GameEvent.Context.of(enderman, state));
            enderman.setCarriedBlock(state.getBlock().defaultBlockState());
        }
    }
}
```
_可以发现，`ForgeEventFactory.getMobGriefingEvent(enderman.level(), enderman)`总是在canUse中被调用。这首先是因为**GameRule只是标签**，在操作前一定要确认GameRule是否允许这一行为，其次也因为**Forge提供了EntityMobGriefingEvent这一事件**。如果Forge有相关的事件（例如上面有EntityMobGriefingEvent和EntityPlaceEvent），不要忘记直接或间接post它们。_  

`EndermanLeaveBlockGoal`相对容易理解一些。每两刻如果AI可用，首先会生成一个随机方块坐标，然后检查这个方块坐标是否可以放下手中的方块。如果可以，那么`canContinueToUse`就会返回`false`，这个AI就会`stop`，否则这个AI就会继续运行并寻找方块。  

`EndermanTakeBlockGoal`也用了类似的机制，但是该AI被“触发”的概率更大（因为搬起方块更难找到合适的，有可搬运方块的坐标）。  

提一下`EndermanTakeBlockGoal`用到的`HitResult`（旧称`RayTraceResult`），不论是用于描述指向的方块的`BlockHitResult`，还是用于描述指向的实体的`EntityHitResult`，使用频率都很高。在这个AI中的作用，则是判断末影人坐标到尝试搬起方块的坐标间是否有障碍。  
下表对比了`BlockHitResult`与`EntityHitResult`的一些区别：  

|      类别      | BlockHitResult | EntityHitResult |
|:--------------:|:------------------:|:-------------------:|
|    主要作用    |         描述指向的**方块**              |          描述指向的**实体**                     |
|    用途举例    |         方块的放置（如在地上放置草方块）  |         与实体的交互（如用骨头喂狼）             |
| 一般的获取方式 |        Level类的clip实例方法            |         ProjectileUtil类中的一系列静态方法        |

最后是代码最长的AI，`EndermanLookForPlayerGoal`。Wiki上这样写：_“玩家在64格距离内注视末影人的头部达到5游戏刻（0.25秒）也会激怒它们。”_，这是怎么实现的呢？
```java
static class EndermanLookForPlayerGoal extends NearestAttackableTargetGoal<Player> {
    private final EnderMan enderman;
    @Nullable
    private Player pendingTarget;
    private int aggroTime;
    private int teleportTime;
    private final TargetingConditions startAggroTargetConditions;
    private final TargetingConditions continueAggroTargetConditions = TargetingConditions.forCombat().ignoreLineOfSight();
    private final Predicate<LivingEntity> isAngerInducing;

    public EndermanLookForPlayerGoal(EnderMan enderman, @Nullable Predicate<LivingEntity> targetConditions) {
        super(enderman, Player.class, 10, false, false, targetConditions);
        this.enderman = enderman;
        this.isAngerInducing = entity -> (enderman.isLookingAtMe((Player) entity) || enderman.isAngryAt(entity)) && !enderman.hasIndirectPassenger(entity);
        // TargetingConditions里forCombat和forNonCombat的区别在于forCombat会检查是否是和平模式以及target是否为队友，而forNonCombat不会
        this.startAggroTargetConditions = TargetingConditions.forCombat().range(getFollowDistance()).selector(isAngerInducing);
    }

    @Override
    public boolean canUse() {
    //  pendingTarget，指待定的攻击目标
        pendingTarget = enderman.level().getNearestPlayer(startAggroTargetConditions, enderman);
        return pendingTarget != null;
    }

    @Override
    public void start() {
        aggroTime = adjustedTickDelay(5);
        teleportTime = 0;
        enderman.setBeingStaredAt();
    }

    @Override
    public void stop() {
        pendingTarget = null;
        super.stop();
    }

    @Override
    public boolean canContinueToUse() {
        if (pendingTarget != null) {
            if (!isAngerInducing.test(pendingTarget)) {
                return false;
            } else {
            //  lookAt的后两个float参数分别表示最大的yRot，最大的xRot
                enderman.lookAt(pendingTarget, 10.0F, 10.0F);
                return true;
            }
        } else {
            if (target != null) {
                if (enderman.hasIndirectPassenger(target)) {
                    return false;
                }
                if (continueAggroTargetConditions.test(enderman, target)) {
                    return true;
                }
            }
            return super.canContinueToUse();
        }
    }

    @Override
    public void tick() {
        if (enderman.getTarget() == null) {
            super.setTarget(null);
        }
        if (pendingTarget != null) {
            if (--aggroTime <= 0) {
          //    达到5刻，切换目标
                target = pendingTarget;
                pendingTarget = null;
                super.start();
            }
        } else {
        //  注视末影人后末影人的行为
            if (target != null && !enderman.isPassenger()) {
                if (enderman.isLookingAtMe((Player) target)) {
                    if (target.distanceToSqr(enderman) < 16.0D) {
                        enderman.teleport();
                    }
                    teleportTime = 0;
                } else if (target.distanceToSqr(enderman) > 256.0D && teleportTime++ >= adjustedTickDelay(30) && enderman.teleportTowards(target)) {
                    teleportTime = 0;
                }
            }
            super.tick();
        }
    }
}
```
`EndermanLookForPlayerGoal`中**先用pendingTarget临时记录了待定的攻击目标**，等到注视5游戏刻后，再将其设置为真正的攻击目标。注意`canUse`被重写了，也就是说只要附近有玩家，这个AI就会“start”。

本节的内容就到此为止了，下一节将会简单讲一下末影人的模型与渲染~
