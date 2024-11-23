#末影人与“中立”生物

_**注：**由于末影人更多地被认为是敌对的怪物而非友好的中立生物，所以在1.2节而非1.4节中进行分析。_

---

末影人被认为是敌对性的中立生物，这与`EnderMan`（旧称`EndermanEntity`，感觉是把Enderman的字母“M”误打成大写了）类实现的2个接口密切相关。其中的一个接口就是前面说过的Enemy，而另一个接口，**NeutralMob**（旧称`IAngerable`），则不仅仅是一个标记接口，还定义了大量与“生气”相关的方法。
这节笔者主要讲`NeutralMob`这个接口，可能涉及少量实体类对此接口里抽象方法的实现。

---

在开始之前，先要说明一件事情。  
`NeutralMob`接口靠后的一部分抽象方法（即以下方法）在`Mob`或`LivingEntity`中都有实现，而且很容易理解，因此本节不讨论它们。
```java
@Nullable
LivingEntity getLastHurtByMob();

void setLastHurtByMob(@Nullable LivingEntity lastHurtByMob);

void setLastHurtByPlayer(@Nullable Player lastHurtByPlayer);

void setTarget(@Nullable LivingEntity target);

boolean canAttack(LivingEntity entity);

@Nullable
LivingEntity getTarget();
```

**但是要注意**，在你的接口中，**千万不能这样写**，除非你声明的方法不会被重混淆（例如你声明了一个`toString`）。举个例子，假设你写了如下的代码：
```java
public interface ExampleInterface {
    float getHealth();

    default void foo() {
        ExampleMod.LOGGER.info(String.valueOf(getHealth()));
    }
}

public class ExampleEntity extends PathfinderMob implements ExampleInterface {
    public ExampleEntity(EntityType<? extends ExampleEntity> type, Level level) {
        super(type, level);
    }

    @Override
    public void tick() {
        super.tick();
        foo();
    }
}
```
否则编译时正常，开发环境中也正常，而到生产环境中，由于`LivingEntity`类中的`getHealth`方法会被重混淆为SRG名，你的`ExampleInterface`里的`getHealth`却还是`getHealth`，于是产生了一个没有被实现的抽象方法。只要你的实体一`tick`，`AbstractMethodError`就会迎面而来~（本人已亲身体验过一次）  

为了摆脱讨厌的`AbstractMethodError`，你需要这样写：
```java
public interface ExampleInterface {
    private LivingEntity self() {
        return (LivingEntity) this;
    }

    default void foo() {
        ExampleMod.LOGGER.info(String.valueOf(self().getHealth()));
    }
}

public class ExampleEntity extends PathfinderMob implements ExampleInterface {
    public ExampleEntity(EntityType<? extends ExampleEntity> type, Level level) {
        super(type, level);
    }

    @Override
    public void tick() {
        super.tick();
        foo();
    }
}
```
这样就可以有效地赶走`AbstractMethodError`。你不妨去翻一下`IForgeEntity`，`IForgeBlock`等Forge的接口的源代码，里面就有很多这样的写法。

---

回到正题，我们先从前面的2组getter和setter说起。
```java
// 注意单位为tick
int getRemainingPersistentAngerTime();

void setRemainingPersistentAngerTime(int remainingPersistentAngerTime);

@Nullable
UUID getPersistentAngerTarget();

void setPersistentAngerTarget(@Nullable UUID persistentAngerTarget);
```
EnderMan里的实现：
```java
@Override
public int getRemainingPersistentAngerTime() {
    return remainingPersistentAngerTime;
}

@Override
public void setRemainingPersistentAngerTime(int remainingPersistentAngerTime) {
   this.remainingPersistentAngerTime = remainingPersistentAngerTime;
}

@Nullable
@Override
public UUID getPersistentAngerTarget() {
    return persistentAngerTarget;
}

@Override
public void setPersistentAngerTarget(@Nullable UUID persistentAngerTarget) {
    this.persistentAngerTarget = persistentAngerTarget;
}
```
`Bee`里前两个方法的实现。因为蜜蜂生气时与不生气时材质不同，所以这里用了`entityData`。
```java
@Override
public int getRemainingPersistentAngerTime() {
    return entityData.get(DATA_REMAINING_ANGER_TIME);
}

@Override
public void setRemainingPersistentAngerTime(int remainingPersistentAngerTime) {
    entityData.set(DATA_REMAINING_ANGER_TIME, remainingPersistentAngerTime);
}
```
上面一组getter和setter控制了剩余的生气时间（一旦`remainingPersistentAngerTime`为0就不会再生气），下面一组则控制了当前生气的目标（的UUID）（UUID并非只有玩家有，任何实体都有）。  

下一个方法及实现。
```java
void startPersistentAngerTimer();
```
```java
private static final UniformInt PERSISTENT_ANGER_TIME = TimeUtil.rangeOfSeconds(20, 39);

@Override
public void startPersistentAngerTimer() {
    setRemainingPersistentAngerTime(PERSISTENT_ANGER_TIME.sample(random));
}
```
可见这个方法被调用时，`remainingPersistentAngerTime`会被设置为一个随机值（在末影人中，这个值是[400, 780]间的随机整数（静态工厂方法`TimeUtil.rangeOfSeconds`在创建`UniformInt`实例时，给20和39分别乘上了20））。当然如果你的实体实现了这个方法，你也可以做些其他的事，这个方法的调用时机马上会讲到。

下面就是一些默认方法了，这些方法**通常不需要被重写**。首先是数据保存与加载。
```java
default void addPersistentAngerSaveData(CompoundTag tag) {
    tag.putInt("AngerTime", getRemainingPersistentAngerTime());
    if (getPersistentAngerTarget() != null) {
        tag.putUUID("AngryAt", getPersistentAngerTarget());
    }
}

default void readPersistentAngerSaveData(Level level, CompoundTag tag) {
    setRemainingPersistentAngerTime(tag.getInt("AngerTime"));
    if (level instanceof ServerLevel) {
        if (!tag.hasUUID("AngryAt")) {
            setPersistentAngerTarget(null);
        } else {
            UUID angryAtUUID = tag.getUUID("AngryAt");
            setPersistentAngerTarget(angryAtUUID);
            Entity angryAt = ((ServerLevel) level).getEntity(angryAtUUID);
            if (angryAt != null) {
                if (angryAt instanceof Mob) {
                    setLastHurtByMob((Mob) angryAt);
                }
                if (entity.getType() == EntityType.PLAYER) {
                    setLastHurtByPlayer((Player) angryAt);
                }
            }
        }
    }
}
```
对于这里的数据与加载，需要注意如下几点：
- `getPersistentAngerTarget`可能返回`null`，在保存时需要一个null-check
- 在从NBT中读取数据时，因为可能没有保存`persistentAngerTarget`，因此要用到`hasUUID`的检查（这个之前提到过）
- 通过UUID获取实体一定要在服务端进行  

接下来是Anger的更新。
```java
default void updatePersistentAnger(ServerLevel level, boolean alwaysAngryIfTargetsPlayer) {
    LivingEntity target = getTarget();
    UUID angryAt = getPersistentAngerTarget();
    if ((target == null || target.isDeadOrDying()) && angryAt != null && level.getEntity(angryAt) instanceof Mob) {
        stopBeingAngry();
    } else {
        if (target != null && !Objects.equals(angryAt, target.getUUID())) {
            setPersistentAngerTarget(target.getUUID());
      //    这里就调用了startPersistentAngerTimer，下面还会有一次调用
            startPersistentAngerTimer();
        }
        if (getRemainingPersistentAngerTime() > 0 && (target == null || target.getType() != EntityType.PLAYER || !alwaysAngryIfTargetsPlayer)) {
            setRemainingPersistentAngerTime(getRemainingPersistentAngerTime() - 1);
            if (getRemainingPersistentAngerTime() == 0) {
                stopBeingAngry();
            }
        }
    }
}
```
这个`updatePersistentAnger`方法应该**在实体更新时在服务端手动调用**。如果给方法的第二个参数传入了`false`（在原版中，只有蜜蜂用了`false`），那么只要`remainingPersistentAngerTime`大于0，这个变量的值就会每游戏刻减少1，也就是说，你的生物会在追击你时慢慢原谅你。

接下来的几个方法用于获取该`NeutralMob`是否生气。
```java
default boolean isAngryAt(LivingEntity entity) {
    if (!canAttack(entity)) {
        return false;
    } else {
        return entity.getType() == EntityType.PLAYER && isAngryAtAllPlayers(entity.level()) ? true : entity.getUUID().equals(getPersistentAngerTarget());
    }
}

default boolean isAngryAtAllPlayers(Level level) {
    return level.getGameRules().getBoolean(GameRules.RULE_UNIVERSAL_ANGER) && isAngry() && getPersistentAngerTarget() == null;
}

default boolean isAngry() {
    return getRemainingPersistentAngerTime() > 0;
}
```
在这些方法中，`isAngryAt`方法常作为方法引用使用在`NearestAttackableTargetGoal`中的最后一个参数中，来控制`NeutralMob`只攻击惹怒自己的玩家。`isAngryAtAllPlayers`只在`isAngryAt`方法中被调用。而`isAngry`方法使用得就比较广泛，常用于判断`NeutralMob`是否在生气中，并用于下一步的行为或材质控制。  

最后是与移除生气状态相关的方法。
```java
default void playerDied(Player player) {
    if (player.level().getGameRules().getBoolean(GameRules.RULE_FORGIVE_DEAD_PLAYERS)) {
        if (player.getUUID().equals(getPersistentAngerTarget())) {
            stopBeingAngry();
        }
    }
}

default void forgetCurrentTargetAndRefreshUniversalAnger() {
    stopBeingAngry();
    startPersistentAngerTimer();
}

default void stopBeingAngry() {
    setLastHurtByMob(null);
    setPersistentAngerTarget(null);
    setTarget(null);
    setRemainingPersistentAngerTime(0);
}
```
当玩家死亡后，`playerDied`方法将会在**服务端**被自动调用，也就是说，原版中`player`参数总是传入一个`ServerPlayer`的实例。`forgetCurrentTargetAndRefreshUniversalAnger`方法，则在`NeutralMob`的`ResetUniversalAngerTargetGoal`里的`start`方法中被调用，用来重置Anger。`stopBeingAngry`方法用于直接移除`NeutralMob`的生气状态，由于`NeutralMob`类中已经写好了部分方法，所以一般不需要直接调用它（唯一的例外是在蜜蜂成功伤害实体后，这个方法会被直接调用）。

这节的内容就是这么多啦，下一节将正式开始分析末影人~
