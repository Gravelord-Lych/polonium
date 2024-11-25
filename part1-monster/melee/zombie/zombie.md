#僵尸的实现逻辑  

***注：***
1. *在以后的实体分析中，所有较重要的内容会写进正文里，次重要的内容以及对原版代码的补充说明则会放在注释中。*
2. *可能会对引用的原版代码进行包括但不限于以下操作以方便阅读：*
    - *给重写了父类方法的方法添加@Override注解*
    - *重命名形参和局部变量*
    - *更改原版代码的缩进，换行等格式*
    - *省略可以省略的this*
    - *如果可以替换，就把全限定类名换成非限定类名*

---

僵尸是大家最熟悉的近战怪物之一。这节中，我们将结合僵尸的源码分析僵尸的行为和底层实现。  

先来看下面两行代码：
```java
public class Zombie extends Monster {}
public abstract class Monster extends PathfinderMob implements Enemy {}
```
其中`PathFinderMob`在普通的`Mob`的基础上，添加了拴绳相关的行为（重写了`tickLeash()`方法）
而`Enemy`接口，则是在**写任何怪物时都必须直接或间接实现的一个标记接口**。

具体来说，Minecraft对实现`Enemy`接口的实体（以下简称Enemy）规定了一些特殊的性质与行为。例如：
- 铁傀儡会主动攻击Enemy
- Enemy不能被栓绳牵引
- 潮涌核心会对Enemy造成伤害
- ......

相较于`Enemy`接口，`Monster`实际上是可继承可不继承的，因为`Monster`类中只重写了一些方法，比如`isPreventingPlayerRest(Player)`，并更改了一些音效（使用敌对生物的音效）

接下来看下面的常量与变量
```java
// 小僵尸的速度提升的修饰符（AttributeModifier，译名修饰符）的UUID
private static final UUID SPEED_MODIFIER_BABY_UUID = UUID.fromString("B9766B59-9566-4402-BC1F-2EE2A276D836");
// 小僵尸的速度提升的修饰符（基础值变为原来的1.5倍）
private static final AttributeModifier SPEED_MODIFIER_BABY = new AttributeModifier(SPEED_MODIFIER_BABY_UUID, "Baby speed boost", 0.5D, AttributeModifier.Operation.MULTIPLY_BASE);
// 决定了僵尸是否为小僵尸，值为true则为小僵尸
private static final EntityDataAccessor<Boolean> DATA_BABY_ID = SynchedEntityData.defineId(Zombie.class, EntityDataSerializers.BOOLEAN);
// 暂无实际用途
private static final EntityDataAccessor<Integer> DATA_SPECIAL_TYPE_ID = SynchedEntityData.defineId(Zombie.class, EntityDataSerializers.INT);
// 决定了僵尸是否正在转化为溺尸，值为true则正在转化
private static final EntityDataAccessor<Boolean> DATA_DROWNED_CONVERSION_ID = SynchedEntityData.defineId(Zombie.class, EntityDataSerializers.BOOLEAN);
// 当难度为困难时，僵尸可以破门
private static final Predicate<Difficulty> DOOR_BREAKING_PREDICATE = difficulty -> difficulty == Difficulty.HARD;

// 僵尸破门的AI
private final BreakDoorGoal breakDoorGoal = new BreakDoorGoal(this, DOOR_BREAKING_PREDICATE);
// 决定了僵尸是否能破门，值为true则说明可以破门
private boolean canBreakDoors;
// 僵尸泡在水里的时间（无特殊说明单位都为tick）
private int inWaterTime;
// 僵尸转化为溺尸（或尸壳转化为僵尸）剩余的时间
private int conversionTime;
```
大家应该知道`EntityDataAccessor`（原名DataParameter）具有在服务端与客户端之间自动同步数据的功能，不过当数据**无需同步**时，使用`EntityDataAccessor`却是多余的。在上面的例子中，小僵尸和僵尸的**模型不同**，但**僵尸的尺寸却是在服务端决定的**，所以我们需要同步数据，在僵尸转化为溺尸时，客户端**只需要知道是否开始了转化**（正在转化的僵尸会颤抖），**不需要知道僵尸泡在水里的时间和转化剩余的时间**，因此`inWaterTime`和`conversionTime`并没有同步，只同步了`DATA_DROWNED_CONVERSION_ID`。还有，不要忘记在`defineSynchedData`方法或实体的构造方法中定义`EntityDataAccessor`。

同时我们还要留意到以下关于僵尸体型设置的细节：
```java
public void setBaby(boolean baby) {
    getEntityData().set(DATA_BABY_ID, baby);
    if (level() != null && !level().isClientSide) {
        AttributeInstance instance = getAttribute(Attributes.MOVEMENT_SPEED);
        instance.removeModifier(SPEED_MODIFIER_BABY);
        if (baby) {
            instance.addTransientModifier(SPEED_MODIFIER_BABY);
        }
    }
}

@Override // @Override为手动添加，下同
public void onSyncedDataUpdated(EntityDataAccessor<?> accessor) {
    if (DATA_BABY_ID.equals(accessor)) {
//      注意这里的dimension指尺寸而不是维度
        refreshDimensions();
    }
    super.onSyncedDataUpdated(accessor);
}

@Override
public int getExperienceReward() {
    if (isBaby()) {
//      小僵尸掉落的xp是普通僵尸的2.5倍
        xpReward = (int) ((double) xpReward * 2.5D);
    }
    return super.getExperienceReward();
}
```
首先要留意到`setBaby`方法不仅仅是设置了`DATA_BABY_ID`的值，而是在这之后还进行了这样一步：移除僵尸身上的`SPEED_MODIFIER_BABY`，如果僵尸是小僵尸，就给该僵尸**临时**添加这个修饰符。`AttributeInstance`类中还有`addPermanentModifier`方法（这个方法添加的修饰符将会保存到实体的NBT中），但因为`setBaby`方法还会在`readAdditionalSaveData`方法中被调用，因此不需要添加到实体的永久修饰符中。

其次还要注意`onSyncedDataUpdated`方法，在`DATA_BABY_ID`改变后，实体的尺寸也要在客户端随之改变，因此要调用`refreshDimensions`方法。

下面是僵尸的AI与属性注册：
```java
@Override
protected void registerGoals() {
    goalSelector.addGoal(4, new Zombie.ZombieAttackTurtleEggGoal(this, 1.0D, 3));
    goalSelector.addGoal(8, new LookAtPlayerGoal(this, Player.class, 8.0F));
    goalSelector.addGoal(8, new RandomLookAroundGoal(this));
    addBehaviourGoals();
}

protected void addBehaviourGoals() {
    goalSelector.addGoal(2, new ZombieAttackGoal(this, 1.0D, false));
    goalSelector.addGoal(6, new MoveThroughVillageGoal(this, 1.0D, true, 4, this::canBreakDoors));
    goalSelector.addGoal(7, new WaterAvoidingRandomStrollGoal(this, 1.0D));
    targetSelector.addGoal(1, (new HurtByTargetGoal(this)).setAlertOthers(ZombifiedPiglin.class));
    targetSelector.addGoal(2, new NearestAttackableTargetGoal<>(this, Player.class, true));
    targetSelector.addGoal(3, new NearestAttackableTargetGoal<>(this, AbstractVillager.class, false));
    targetSelector.addGoal(3, new NearestAttackableTargetGoal<>(this, IronGolem.class, true));
//  Turtle.BABY_ON_LAND_SELECTOR是用来选择可攻击的海龟的Predicate（也就是说僵尸只会攻击岸上的小（baby）海龟）
    targetSelector.addGoal(5, new NearestAttackableTargetGoal<>(this, Turtle.class, 10, true, false, Turtle.BABY_ON_LAND_SELECTOR));
}

public static AttributeSupplier.Builder createAttributes() {
//  createMonsterAttributes方法中包含对Attributes.MAX_HEALTH的注册（事实上createLivingAttributes方法就注册了）
//  而僵尸的最大生命值就是20，因此无需重复注册
    return Monster.createMonsterAttributes()
        .add(Attributes.FOLLOW_RANGE, 35.0D)
        .add(Attributes.MOVEMENT_SPEED, (double) 0.23F)
        .add(Attributes.ATTACK_DAMAGE, 3.0D)
        .add(Attributes.ARMOR, 2.0D)
        // 不传入double参数则会使用一个属性的默认值
        .add(Attributes.SPAWN_REINFORCEMENTS_CHANCE);
}

public boolean canBreakDoors() {
    return canBreakDoors;
}
```
如果你对Mob的AI不是很熟悉，推荐阅读[这篇教程](https://izzel.io/2021/12/19/living-things/)（当然本文中不会涉及到`Brain`）。该文章中的后记也很好地解释了为什么大多数Boss都不会使用`Goal`和`Brain`。  

不难发现僵尸在注册AI的`registerGoals`方法中调用了`addBehaviourGoals`方法，这是一种多态（在`Husk`等`Zombie`的子类中将会重写这个方法），在`Zombie`类中大多数被定义为`protected`的方法都用到了多态的思想。
注意这里**没有注册breakDoorGoal**，我们马上会讲到它。  

接下来是`setCanBreakDoors`方法。`setCanBreakDoors`方法将在僵尸的`finalizeSpawn`中被调用，而后者是在`Mob`即将生成完毕时会被调用的方法，这个我们后面再讲。
```java
public void setCanBreakDoors(boolean canBreakDoors) {
//  如果Mob的一个实例mob使用的PathNavigation是GroundPathNavigation的实例（instanceof GroundPathNavigation），GoalUtils.hasGroundPathNavigation(mob) 就会返回true
    if (supportsBreakDoorGoal() && GoalUtils.hasGroundPathNavigation(this)) {
        if (this.canBreakDoors != canBreakDoors) {
            this.canBreakDoors = canBreakDoors;
            ((GroundPathNavigation) getNavigation()).setCanOpenDoors(canBreakDoors);
            if (canBreakDoors) {
                goalSelector.addGoal(1, breakDoorGoal);
            } else {
                goalSelector.removeGoal(breakDoorGoal);
            }
        }
    } else if (this.canBreakDoors) {
        goalSelector.removeGoal(breakDoorGoal);
        this.canBreakDoors = false;
    }
}

// 同样是多态（溺尸就不可能破门）
protected boolean supportsBreakDoorGoal() {
    return true;
}
```
我们发现，实际注册与更新了`breakDoorGoal`的位置就是在`setCanBreakDoors`这个方法里。这说明了**Mob的AI的注册与更新不只局限在registerGoals中**（但注意`isClientSide`的判断，不要在客户端注册与更新实体AI）。  

接下来一部分是实体的更新，**实体每tick的更新极其重要**，不论是`Mob`的AI、寻路系统，还是弹射物的飞行，都在实体每tick的更新中完成。
```java
@Override
public void tick() {
    if (!level().isClientSide && isAlive() && !isNoAi()) {
    //  如果转化正在发生...
        if (isUnderWaterConverting()) {
            --conversionTime;
            // ForgeEventFactory.canLivingConvert的第三个参数是Consumer<Integer>
            if (conversionTime < 0 && ForgeEventFactory.canLivingConvert(this, EntityType.DROWNED, timer -> conversionTime = timer)) {
                doUnderWaterConversion();
            }
        }
    //  如果转化可以发生但还没发生...
        else if (convertsInWater()) {
            if (isEyeInFluid(FluidTags.WATER)) {
                ++inWaterTime;
                if (inWaterTime >= 600) {
                //  开始吧！
                    startUnderWaterConversion(300);
                }
            } else {
                inWaterTime = -1;
            }
        }
    }
    super.tick();
}

protected boolean convertsInWater() {
    return true;
}  

private void startUnderWaterConversion(int conversionTime) {
    this.conversionTime = conversionTime;
    getEntityData().set(DATA_DROWNED_CONVERSION_ID, true);
}

protected void doUnderWaterConversion() {
    convertToZombieType(EntityType.DROWNED);
    if (!isSilent()) {
//      广播1040号事件会播放SoundEvents.ZOMBIE_CONVERTED_TO_DROWNED（1041号事件则是播放SoundEvents.HUSK_CONVERTED_TO_ZOMBIE）
        level().levelEvent(null, 1040, blockPosition(), 0);
    }
}

protected void convertToZombieType(EntityType<? extends Zombie> zombieType) {
    Zombie zombie = convertTo(zombieType, true);
    if (zombie != null) {
        zombie.handleAttributes(zombie.level().getCurrentDifficultyAt(zombie.blockPosition()).getSpecialMultiplier());
        zombie.setCanBreakDoors(zombie.supportsBreakDoorGoal() && canBreakDoors());
        ForgeEventFactory.onLivingConvert(this, zombie);
    }
}

@Override
public void aiStep() {
    if (isAlive()) {
        boolean shouldBurn = isSunSensitive() && isSunBurnTick();
        if (shouldBurn) {
            ItemStack helmet = getItemBySlot(EquipmentSlot.HEAD);
        //  僵尸只要有了头盔就可以抵抗阳光~
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
    }
    super.aiStep();
}
```
如果一个`LivingEntity`未被移除，那么这个实体的`aiStep`方法会在`LivingEntity`的`tick`方法中被调用（即每tick调用1次），在调用完`aiStep`后，将会更新实体的旋转角度。  
这里重写的`tick`方法中，主要更新了僵尸的转化（尸壳->僵尸->溺尸）；而这里重写的`aiStep`方法，使僵尸在阳光下着火（同样的逻辑在`AbstractSkeleton`里，以几乎一样的代码，又出现了一次...）。  

然后就到了分别与受击和攻击有关的`hurt`和`doHurtTarget`方法，这两个方法在复杂实体的开发中也非常常用。
```java
@Override
public boolean hurt(DamageSource source, float amount) {
    if (!super.hurt(source, amount)) {
        return false;
    } else if (!(level() instanceof ServerLevel)) {
        return false;
    } else {
        ServerLevel level = (ServerLevel) level();
        LivingEntity target = getTarget();
        if (target == null && source.getEntity() instanceof LivingEntity) {
            target = (LivingEntity) source.getEntity();
        }

        int x = Mth.floor(getX());
        int y = Mth.floor(getY());
        int z = Mth.floor(getZ());
        ZombieEvent.SummonAidEvent event = ForgeEventFactory.fireZombieSummonAid(this, level(), x, y, z, target, getAttribute(Attributes.SPAWN_REINFORCEMENTS_CHANCE).getValue());
        if (event.getResult() == Event.Result.DENY) {
            return true;
        }

//      大致解释一下这个超长条件：
//      如果将事件SummonAidEvent的结果设置为ALLOW，则僵尸一定会呼叫增援
//      否则执行原版逻辑：若游戏难度是困难、游戏规则doMobSpawning为true并且被打时攻击目标或伤害自己者存在，则生成一个[0, 1)的随机浮点数（记为n），
//                      如果n小于Attributes.SPAWN_REINFORCEMENTS_CHANCE就呼叫增援
        if (event.getResult() == Event.Result.ALLOW  || target != null
                && level().getDifficulty() == Difficulty.HARD
                && (double) random.nextFloat() < getAttribute(Attributes.SPAWN_REINFORCEMENTS_CHANCE).getValue()
                && level().getGameRules().getBoolean(GameRules.RULE_DOMOBSPAWNING)) {
             Zombie zombie = event.getCustomSummonedAid() != null && event.getResult() == Event.Result.ALLOW ? event.getCustomSummonedAid() : EntityType.ZOMBIE.create(level());

//           尝试50次
             for (int i = 0; i < 50; ++i) {
                 int randomX = x + Mth.nextInt(random, 7, 40) * Mth.nextInt(random, -1, 1);
                 int randomY = y + Mth.nextInt(random, 7, 40) * Mth.nextInt(random, -1, 1);
                 int randomZ = z + Mth.nextInt(random, 7, 40) * Mth.nextInt(random, -1, 1);
                 BlockPos spawnPos = new BlockPos(randomX, randomY, randomZ);
                 EntityType<?> zombieType = zombie.getType();
                 SpawnPlacements.Type placementType = SpawnPlacements.getPlacementType(zombieType);
                 if (NaturalSpawner.isSpawnPositionOk(placementType, level(), spawnPos, zombieType) && SpawnPlacements.checkSpawnRules(zombieType, level, MobSpawnType.REINFORCEMENT, spawnPos, level().random)) {
                     zombie.setPos(randomX, randomY, randomZ);
//                   又一个长条件，大致解释一下：
//                   如果僵尸的生成位置7格之内没有（活的）玩家，同时生成的僵尸的碰撞箱内既没有障碍物，也没有液体，就允许生成支援的僵尸
                     if (!level().hasNearbyAlivePlayer(randomX, randomY, randomZ, 7.0D)
                             && level().isUnobstructed(zombie)
                             && level().noCollision(zombie)
                             && !level().containsAnyLiquid(zombie.getBoundingBox())) {
                         if (target != null) {
                             zombie.setTarget(target);
                         }
                         zombie.finalizeSpawn(level, level().getCurrentDifficultyAt(zombie.blockPosition()), MobSpawnType.REINFORCEMENT, null, null);
                         level.addFreshEntityWithPassengers(zombie);
                    //   降低新生成的僵尸的召唤援助概率
                         getAttribute(Attributes.SPAWN_REINFORCEMENTS_CHANCE).addPermanentModifier(new AttributeModifier("Zombie reinforcement caller charge", -0.05F, AttributeModifier.Operation.ADDITION));
                         zombie.getAttribute(Attributes.SPAWN_REINFORCEMENTS_CHANCE).addPermanentModifier(new AttributeModifier("Zombie reinforcement callee charge", -0.05F, AttributeModifier.Operation.ADDITION));
                         break;
                     }
                 }
             }
         }
         return true;
    }
}

@Override
public boolean doHurtTarget(Entity target) {
    boolean success = super.doHurtTarget(target);
//  只有成功造成了伤害才会传火
    if (success) {
        float difficulty = level().getCurrentDifficultyAt(blockPosition()).getEffectiveDifficulty();
        if (getMainHandItem().isEmpty() && isOnFire() && random.nextFloat() < difficulty * 0.3F) {
        //  传火~
            target.setSecondsOnFire(2 * (int) difficulty);
        }
    }
    return success;
}
```
重写`hurt`方法让僵尸有了呼叫增援的能力，简单说一下如何实现僵尸的呼叫增援，这个思路也是不少召唤型Boss所采用的。

1. 获取攻击的目标
2. 判断是否满足呼叫增援的条件
3. 重复尝试50次，如果成功（随机选择的位置符合要求）立即break  

其中**重复尝试的思想是一个重要的思想**，我们在1.2.1.3.2节还会去讲。往往当你苦于如何生成满足要求的随机坐标时，它会派上大用场。植物魔法中[盖亚守护者的随机传送位置的选定](https://github.com/VazkiiMods/Botania/blob/1.20.x/Xplat/src/main/java/vazkii/botania/common/entity/GaiaGuardianEntity.java)（第917行），就用到了这种思想。  

重写`doHurtTarget`方法主要目的是为了让僵尸能在一定难度下传火给僵尸攻击的目标。这里的代码不难理解，关于区域难度的计算不是本教程的重点，如果你感兴趣，可以阅读`DifficultyInstance`类的源代码。

`doHurtTarget`和`hurt`方法都有返回值，如果成功造成了伤害（`doHurtTarget`）或受到了伤害（`hurt`），就应该返回true，否则一般返回false

_还有一点需要注意，不管是上文所述的代码高度重复，还是这部分出现的if语句中使用长条件，都是不好的开发习惯，需要**尽量避免**。毕竟开发Mod不是参加OI（这种算法竞赛中只要你能AC，你全用单字母变量名与函数名都没人管你），要保证代码的可读性。_

接着是`killedEntity`方法，这个方法虽不经常被重写，但对于僵尸依然重要。
```java
@Override
public boolean killedEntity(ServerLevel level, LivingEntity entity) {
    boolean killed = super.killedEntity(level, entity);
    if ((level.getDifficulty() == Difficulty.NORMAL || level.getDifficulty() == Difficulty.HARD)
            && entity instanceof Villager villager
            && ForgeEventFactory.canLivingConvert(entity, EntityType.ZOMBIE_VILLAGER, timer -> {})) {
 //     即普通难度下50%，困难难度下100%召唤僵尸村民
        if (level.getDifficulty() != Difficulty.HARD && random.nextBoolean()) {
            return killed;
        }
        ZombieVillager zombieVillager = villager.convertTo(EntityType.ZOMBIE_VILLAGER, false);
        if (zombieVillager != null) {
 //         复制村民的部分数据到僵尸村民
            zombieVillager.finalizeSpawn(level, level.getCurrentDifficultyAt(zombieVillager.blockPosition()), MobSpawnType.CONVERSION, new Zombie.ZombieGroupData(false, true), null);
            zombieVillager.setVillagerData(villager.getVillagerData());
            zombieVillager.setGossips(villager.getGossips().store(NbtOps.INSTANCE));
            zombieVillager.setTradeOffers(villager.getOffers().createTag());
            zombieVillager.setVillagerXp(villager.getVillagerXp());
            ForgeEventFactory.onLivingConvert(entity, zombieVillager);
            if (!isSilent()) {
//              广播1026号事件会播放SoundEvents.ZOMBIE_INFECT
                level.levelEvent(null, 1026, blockPosition(), 0);
            }
            killed = false;
        }
    }
    return killed;
}
```
Minecraft中并没有很好的复制实体数据到另一个实体的方法，因此上面的代码中出现了许多`a.set(b.get())`的操作。  

注意这个方法也有返回值，如果返回了`false`（MC里还没这样干过），那么`GameEvent.ENTITY_DIE`（`GameEvent`与新版本的“声音”有关）就不会被广播，实体也不会有任何掉落物（包括凋零玫瑰）。

然后是数据保存与加载。
```java
@Override
public void addAdditionalSaveData(CompoundTag tag) {
    super.addAdditionalSaveData(tag);
    tag.putBoolean("IsBaby", isBaby());
    tag.putBoolean("CanBreakDoors", canBreakDoors());
    tag.putInt("InWaterTime", isInWater() ? inWaterTime : -1);
    tag.putInt("DrownedConversionTime", isUnderWaterConverting() ? conversionTime : -1);
}

@Override
public void readAdditionalSaveData(CompoundTag tag) {
    super.readAdditionalSaveData(tag);
    setBaby(tag.getBoolean("IsBaby"));
    setCanBreakDoors(tag.getBoolean("CanBreakDoors"));
    inWaterTime = tag.getInt("InWaterTime");
    // 99表示的是任意数字型NBT标签
    if (tag.contains("DrownedConversionTime", 99) && tag.getInt("DrownedConversionTime") > -1) {
        startUnderWaterConversion(tag.getInt("DrownedConversionTime"));
    }
}
```
数据保存的部分在较基础的教程中都有涉及，因此不做过多的赘述，等到后面如果讲到较复杂的数据结构（如`List`，`Map`）的保存时，再讲保存方式。  
需要注意的是，如果一个成员变量的初始值不是默认的初始值（`0`，`false`，`null`）或者该成员变量在`addAdditionalSaveData`保存时使用了条件（eg.`if (a != null) tag.put(a);`），那么在`readAdditionalSaveData`中就**必须进行tag.contains（UUID可以用tag.hasUUID）的检查**（否则一调用完这个方法就会给你换成0，false或null，甚至给你抛一个NPE）。

最后一个重点了！`finalizeSpawn`方法。
```java
@Nullable
@Override
public SpawnGroupData finalizeSpawn(ServerLevelAccessor accessor, DifficultyInstance difficulty, MobSpawnType spawnType, @Nullable SpawnGroupData spawnData, @Nullable CompoundTag spawnTag) {
    RandomSource random = accessor.getRandom();

//  注意你在写的时候，要把这里换成ForgeEventFactory.onFinalizeSpawn(this, accessor, difficulty, spawnType, spawnData, spawnTag);
    spawnData = super.finalizeSpawn(accessor, difficulty, spawnType, spawnData, spawnTag);

    float specialMultiplier = difficulty.getSpecialMultiplier();
    setCanPickUpLoot(random.nextFloat() < 0.55F * specialMultiplier);
    if (spawnData == null) {
        spawnData = new Zombie.ZombieGroupData(getSpawnAsBabyOdds(random), true);
    }

    if (spawnData instanceof Zombie.ZombieGroupData data) {
        if (data.isBaby) {
            setBaby(true);

            if (data.canSpawnJockey) {
                if ((double) random.nextFloat() < 0.05D) {
              //    尝试骑一只没有被骑的鸡
                    List<Chicken> chickens = accessor.getEntitiesOfClass(Chicken.class, getBoundingBox().inflate(5.0D, 3.0D, 5.0D), EntitySelector.ENTITY_NOT_BEING_RIDDEN);
                    if (!chickens.isEmpty()) {
                        Chicken chicken = chickens.get(0);
                        chicken.setChickenJockey(true);
                        startRiding(chicken);
                    }
                } else if ((double) random.nextFloat() < 0.05D) {
              //    尝试自己生成一只鸡骑
                    Chicken chicken = EntityType.CHICKEN.create(level());
                    if (chicken != null) {
                        chicken.moveTo(getX(), getY(), getZ(), getYRot(), 0.0F);
                        chicken.finalizeSpawn(accessor, difficulty, MobSpawnType.JOCKEY, null, null);
                        chicken.setChickenJockey(true);
                        startRiding(chicken);
                        accessor.addFreshEntity(chicken);
                    }
                }
            }
        }

        setCanBreakDoors(supportsBreakDoorGoal() && random.nextFloat() < specialMultiplier * 0.1F);
        populateDefaultEquipmentSlots(random, difficulty);
        populateDefaultEquipmentEnchantments(random, difficulty);
    }

//  万圣节的彩蛋
    if (getItemBySlot(EquipmentSlot.HEAD).isEmpty()) {
        LocalDate date = LocalDate.now();
        int day = date.get(ChronoField.DAY_OF_MONTH);
        int month = date.get(ChronoField.MONTH_OF_YEAR);

        if (month == 10 && day == 31 && random.nextFloat() < 0.25F) {
            setItemSlot(EquipmentSlot.HEAD, new ItemStack(random.nextFloat() < 0.1F ? Blocks.JACK_O_LANTERN : Blocks.CARVED_PUMPKIN));
        //  这时南瓜头盔不可能掉落
            armorDropChances[EquipmentSlot.HEAD.getIndex()] = 0.0F;
        }
    }

    handleAttributes(specialMultiplier);
    return spawnData;
}

public static boolean getSpawnAsBabyOdds(RandomSource random) {
    return random.nextFloat() < ForgeConfig.SERVER.zombieBabyChance.get();
}

protected void handleAttributes(float specialMultiplier) {
    randomizeReinforcementsChance();
    getAttribute(Attributes.KNOCKBACK_RESISTANCE).addPermanentModifier(new AttributeModifier("Random spawn bonus", random.nextDouble() * (double) 0.05F, AttributeModifier.Operation.ADDITION));

    double bonusMultiplier = random.nextDouble() * 1.5D * (double) specialMultiplier;
    if (bonusMultiplier > 1) {
        getAttribute(Attributes.FOLLOW_RANGE).addPermanentModifier(new AttributeModifier("Random zombie-spawn bonus", bonusMultiplier, AttributeModifier.Operation.MULTIPLY_TOTAL));
    }

//  强化“领头”僵尸
    if (random.nextFloat() < specialMultiplier * 0.05F) {
        getAttribute(Attributes.SPAWN_REINFORCEMENTS_CHANCE)
            .addPermanentModifier(new AttributeModifier("Leader zombie bonus", random.nextDouble() * 0.25D + 0.5D, AttributeModifier.Operation.ADDITION));
        getAttribute(Attributes.MAX_HEALTH)
            .addPermanentModifier(new AttributeModifier("Leader zombie bonus", random.nextDouble() * 3.0D + 1.0D, AttributeModifier.Operation.MULTIPLY_TOTAL));
        setCanBreakDoors(supportsBreakDoorGoal());
    }
}

//  Attributes.SPAWN_REINFORCEMENTS_CHANCE的值是在这里被设置的，而不是在createAttributes中（Attributes.SPAWN_REINFORCEMENTS_CHANCE的默认值是0）
//  createAttributes只是声明了这个属性
protected void randomizeReinforcementsChance() {
    getAttribute(Attributes.SPAWN_REINFORCEMENTS_CHANCE).setBaseValue(random.nextDouble() * ForgeConfig.SERVER.zombieBaseSummonChance.get());
}

@Override
protected void populateDefaultEquipmentSlots(RandomSource random, DifficultyInstance difficulty) {
    super.populateDefaultEquipmentSlots(random, difficulty);
    if (random.nextFloat() < (level().getDifficulty() == Difficulty.HARD ? 0.05F : 0.01F)) {
        int i = random.nextInt(3);
        if (i == 0) {
            setItemSlot(EquipmentSlot.MAINHAND, new ItemStack(Items.IRON_SWORD));
        } else {
            setItemSlot(EquipmentSlot.MAINHAND, new ItemStack(Items.IRON_SHOVEL));
        }
    }
}
```
`finalizeSpawn`方法为`Mob`的最终生成做了最后的调整。`Zombie`类重写了这个方法，使僵尸在生成时获得了加强。其中特别容易遗忘的一点是，`populateDefaultEquipmentSlots`（一般用来给予Mob生成时的装备）和`populateDefaultEquipmentEnchantments`（一般用来给Mob生成时的装备附魔）两个方法，虽然在`Mob`类中就声明了，但是**必须在finalizeSpawn方法手动调用**。举一个有关`finalizeSpawn`方法用途的例子：蜘蛛生成时所携带的药水效果，便是在这个方法中添加的。

_注意事项：forge明确说明：在目前的forge版本中，这个方法**只能被重写**，**直接调用finalizeSpawn方法会导致StackOverflowError**！因此**一定要使用ForgeEventFactory.onFinalizeSpawn**！_

然后是`dropCustomDeathLoot`，本节的内容也接近尾声了。
```java
@Override
protected void dropCustomDeathLoot(DamageSource source, int lootingLevel, boolean killedByPlayer) {
    super.dropCustomDeathLoot(source, lootingLevel, killedByPlayer);
    Entity entity = source.getEntity();
    if (entity instanceof Creeper creeper) {
        if (creeper.canDropMobsSkull()) {
            ItemStack skull = getSkull();
            if (!skull.isEmpty()) {
                creeper.increaseDroppedSkulls();
                spawnAtLocation(skull);
            }
        }
    }
}

protected ItemStack getSkull() {
    return new ItemStack(Items.ZOMBIE_HEAD);
}
```
`dropCustomDeathLoot`主要让`LivingEntity`可以掉落**较复杂的，常规战利品表难以实现的**掉落物（比如被特殊的（高压且没炸掉过头的）苦力怕炸死时会掉落头颅），当然能用战利品表就用战利品表，不要掉什么都用`dropCustomDeathLoot`来实现。

最后是一些杂项。
```java
@Override
protected SoundEvent getAmbientSound() {
    return SoundEvents.ZOMBIE_AMBIENT;
}

@Override
protected SoundEvent getHurtSound(DamageSource source) {
    return SoundEvents.ZOMBIE_HURT;
}

@Override
protected SoundEvent getDeathSound() {
    return SoundEvents.ZOMBIE_DEATH;
}

protected SoundEvent getStepSound() {
    return SoundEvents.ZOMBIE_STEP;
}

@Override
protected void playStepSound(BlockPos pos, BlockState state) {
    playSound(getStepSound(), 0.15F, 1.0F);
}

@Override
public MobType getMobType() {
    return MobType.UNDEAD;
}

@Override
protected float getStandingEyeHeight(Pose pose, EntityDimensions dimensions) {
    return isBaby() ? 0.93F : 1.74F;
}

@Override
public boolean canHoldItem(ItemStack stack) {
    return stack.is(Items.EGG) && isBaby() && isPassenger() ? false : super.canHoldItem(stack);
}

@Override
public boolean wantsToPickUp(ItemStack stack) {
    return stack.is(Items.GLOW_INK_SAC) ? false : super.wantsToPickUp(stack);
}

@Override
public double getMyRidingOffset() {
    return isBaby() ? 0.0D : -0.45D;
}
```
简单提及一下僵尸实体类型（`EntityType`）的注册。  
```java
public static final EntityType<Zombie> ZOMBIE = register("zombie", EntityType.Builder.<Zombie>of(Zombie::new, MobCategory.MONSTER).sized(0.6F, 1.95F).clientTrackingRange(8));
```
这部分理解难度不大，并且在基础的教程中也提到了一部分，不过尤其要注意一点：**千万不要忽视这些细节**！许多优秀的Mod，便优秀在对细节的重视。  
顺便说一下`clientTrackingRange`（单位为区块），**当一个实体在这个追踪距离内时，这个实体将会被更新**。一般“战场”面积越大的实体，`clientTrackingRange`越大（比如末影水晶是16）

僵尸的行为和底层实现便分析到这里了。虽然僵尸看上去很容易实现（也好打），可是与僵尸相关的实现细节却不少，需要一段时间才能理清楚。  

下一节将会分析僵尸的模型及渲染。23333333333~~
