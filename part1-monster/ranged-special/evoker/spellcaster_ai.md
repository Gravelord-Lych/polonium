# 施法AI的框架

观察唤魔者和幻术师的施法AI可以发现它们都有共同的父类`SpellcasterUseSpellGoal`。

```java
class EvokerAttackSpellGoal extends SpellcasterIllager.SpellcasterUseSpellGoal { /*...*/ }
class EvokerSummonSpellGoal extends SpellcasterIllager.SpellcasterUseSpellGoal { /*...*/ }
public class EvokerWololoSpellGoal extends SpellcasterIllager.SpellcasterUseSpellGoal { /*...*/ }
class IllusionerBlindnessSpellGoal extends SpellcasterIllager.SpellcasterUseSpellGoal { /*...*/ }
class IllusionerMirrorSpellGoal extends SpellcasterIllager.SpellcasterUseSpellGoal { /*...*/ }
```

在进一步介绍施法AI的框架前，有必要先简单说说`SpellcasterIllager`。`SpellcasterIllager`继承了`AbstractIllager`，是所有原版的施法类灾厄村民类的父类。  

注意如果想要实现一个新的拥有自定义法术的施法类灾厄村民，**不应该直接继承这个类，而是应该继承`AbstractIllager`，然后可以根据这个类的代码自己重新实现**。具体原因本文最后会提到。

```java
// 此EntityDataAccessor代表的值表示目前施放的法术的ID，用于客户端获取目前施放的法术
private static final EntityDataAccessor<Byte> DATA_SPELL_CASTING_ID = SynchedEntityData.defineId(SpellcasterIllager.class, EntityDataSerializers.BYTE);
// 剩余的施法时间（单位：tick），也就是到该灾厄村民放下手并停止放出粒子效果所剩余的时间。若该灾厄村民不在施法则值为0
protected int spellCastingTickCount;
// 目前施放的法术
private SpellcasterIllager.IllagerSpell currentSpell = IllagerSpell.NONE;
```

其中`IllagerSpell`是一个`SpellcasterIllager`内部的枚举类，列出了原版所有灾厄村民会使用的法术及法术的ID与颜色。由于枚举类的特性，模组开发者是无法添加新的`IllagerSpell`的。
```java
protected static enum IllagerSpell {
    // 这是个dummy value，并不是个法术
    NONE(0, 0.0D, 0.0D, 0.0D),
    // 唤魔者召唤恼鬼的法术
    SUMMON_VEX(1, 0.7D, 0.7D, 0.8D),
    // 唤魔者召唤唤魔者尖牙的法术
    FANGS(2, 0.4D, 0.3D, 0.35D),
    // 唤魔者把蓝色绵羊变成红色的法术
    WOLOLO(3, 0.7D, 0.5D, 0.2D),
    // 幻术师隐身并召唤分身的法术
    DISAPPEAR(4, 0.3D, 0.3D, 0.8D),
    // 幻术师使攻击目标失明的法术
    BLINDNESS(5, 0.1D, 0.1D, 0.2D);

    private static final IntFunction<SpellcasterIllager.IllagerSpell> BY_ID = ByIdMap.continuous(illagerSpell -> {
        return illagerSpell.id;
    }, values(), ByIdMap.OutOfBoundsStrategy.ZERO);
    final int id;
    final double[] spellColor;

    private IllagerSpell(int id, double r, double g, double b) {
        this.id = id;
        this.spellColor = new double[]{r, g, b};
    }

    // byId返回id对应的法术值，根据BY_ID这一IntFunction的处理方式（OutOfBoundsStrategy.ZERO），
    // 如果传入的id为负或者数值大于等于法术的种类数（也就是越界），那么该方法的返回值是0
    public static IllagerSpell byId(int id) {
        return BY_ID.apply(id);
    }
}
```

数据保存部分则保存了`spellCastingTickCount`的值。

```java
@Override
public void readAdditionalSaveData(CompoundTag tag) {
    super.readAdditionalSaveData(tag);
    spellCastingTickCount = tag.getInt("SpellTicks");
}

@Override
public void addAdditionalSaveData(CompoundTag tag) {
    super.addAdditionalSaveData(tag);
    tag.putInt("SpellTicks", spellCastingTickCount);
}
```

下面是4个与法术有关的方法。这些方法中主要用于控制灾厄村民的模型与渲染，在施法AI中也有被用到。

```java
public boolean isCastingSpell() {
    // 注意客户端一定要用entityData获取同步过的值
    if (level().isClientSide) {
        return entityData.get(DATA_SPELL_CASTING_ID) > 0;
    } else {
        return spellCastingTickCount > 0;
    }
}

public void setIsCastingSpell(IllagerSpell currentSpell) {
    this.currentSpell = currentSpell;
    entityData.set(DATA_SPELL_CASTING_ID, (byte) currentSpell.id);
}

protected SpellcasterIllager.IllagerSpell getCurrentSpell() {
    return !level().isClientSide ? currentSpell : IllagerSpell.byId(entityData.get(DATA_SPELL_CASTING_ID));
}

protected int getSpellCastingTime() {
    return spellCastingTickCount;
}
```

接下来是实体更新部分。这一部分中重写`customServerAiStep`方法实现了每刻更新一次`spellCastingTickCount`，而重写`tick`方法用于给灾厄村民添加施法的粒子效果。

```java
@Override
protected void customServerAiStep() {
    super.customServerAiStep();
    if (spellCastingTickCount > 0) {
        --spellCastingTickCount;
    }
}

@Override
public void tick() {
    super.tick();
    if (level().isClientSide && isCastingSpell()) {
        IllagerSpell spell = getCurrentSpell();
        double r = spell.spellColor[0];
        double g = spell.spellColor[1];
        double b = spell.spellColor[2];
        float particleSpawningAngle =
                yBodyRot * ((float) Math.PI / 180F)             // 首先进行了角度到弧度的转换，把yBodyRot转化成弧度
                + Mth.cos((float) tickCount * 0.6662F) * 0.25F; // 再给这个值加上一个周期性的小幅偏移
        float particleXOffs = Mth.cos(particleSpawningAngle);
        float particleZOffs = Mth.sin(particleSpawningAngle);
        level().addParticle(ParticleTypes.ENTITY_EFFECT, getX() + (double) particleXOffs * 0.6D, getY() + 1.8D, getZ() + (double) particleZOffs * 0.6D, r, g, b);
        level().addParticle(ParticleTypes.ENTITY_EFFECT, getX() - (double) particleXOffs * 0.6D, getY() + 1.8D, getZ() - (double) particleZOffs * 0.6D, r, g, b);
    }
}
```

`SpellcasterIllager`还重写了`getArmPose`方法，用来给正在施法中的灾厄村民应用正确的施法的手臂动作，同时使TA们在袭击获胜（对于玩家而言即为失败）时有正确的庆祝动作。
```java
@Override
public IllagerArmPose getArmPose() {
    if (isCastingSpell()) {
        return IllagerArmPose.SPELLCASTING;
    } else {
        return isCelebrating() ? IllagerArmPose.CELEBRATING : IllagerArmPose.CROSSED;
    }
}
```

`SpellcasterIllager`有两个非静态的成员内部类，分别是控制施法时灾厄村民运动的AI（`SpellcasterCastingSpellGoal`）及施法的AI（`SpellcasterUseSpellGoal`）。

控制施法时灾厄村民运动的AI很简单，这个AI使灾厄村民施法时停止移动，并且看向攻击目标。
```java
protected class SpellcasterCastingSpellGoal extends Goal {
    public SpellcasterCastingSpellGoal() {
        setFlags(EnumSet.of(Flag.MOVE, Flag.LOOK));
    }
    
    @Override
    public boolean canUse() {
        return getSpellCastingTime() > 0;
    }
    
    @Override
    public void start() {
        super.start();
        navigation.stop();
    }
    
    @Override
    public void stop() {
        super.stop();
        setIsCastingSpell(IllagerSpell.NONE);
    }

    @Override
    public void tick() {
        if (getTarget() != null) {
            getLookControl().setLookAt(getTarget(), (float) getMaxHeadYRot(), (float) getMaxHeadXRot());
        }
    }
}
```

`Evoker`中重写了这个类并做了小的改动，使唤魔者可以看向正因其施法而改变颜色的绵羊。
```java
class EvokerCastingSpellGoal extends SpellcasterIllager.SpellcasterCastingSpellGoal {
    public void tick() {
        if (getTarget() != null) {
            getLookControl().setLookAt(getTarget(), (float) getMaxHeadYRot(), (float) getMaxHeadXRot());
        } else if (getWololoTarget() != null) {
            getLookControl().setLookAt(getWololoTarget(), (float) getMaxHeadYRot(), (float) getMaxHeadXRot());
        }
    }
}
```

接下来说说施法AI吧。  

首先看这两个成员变量，它们各自调控了法术的准备与冷却时间。  

```java
protected abstract class SpellcasterUseSpellGoal extends Goal {
    // 施法所剩余的准备时间，也就是到该灾厄村民真正施放法术所剩余的时间（单位：tick）
    // 而所有法术都是在attackWarmupDelay为0的“一瞬间”才真正被施放的
    protected int attackWarmupDelay;
    // 这是个时间戳，当灾厄村民的tickCount大于等于该值时灾厄村民就可以施放该法术（tickCount每游戏刻都会自增）
    protected int nextAttackTickCount;
    
    // 不难发现施法AI是没有Flag的，也就是说它们不会与SpellcasterCastingSpellGoal等其他AI冲突
    // 该AI的剩余部分暂时省略
}
```

注意区分`attackWarmupDelay`与`spellCastingTickCount`，前者指到**放下手并停止放出粒子效果**所剩余的时间，后者指到**真正施放法术**所剩余的时间，因此`attackWarmupDelay`应该总是小于等于`spellCastingTickCount`。

然后是“可用性”。当攻击目标存在且存活，同时法术不处于冷却状态时，施法AI可以被使用。而当攻击目标存在且存活，同时施法未结束时，施法AI可以被继续使用。

```java
@Override
public boolean canUse() {
    LivingEntity target = getTarget();
    if (target != null && target.isAlive()) {
        if (isCastingSpell()) {
            return false;
        } else {
            return tickCount >= nextAttackTickCount;
        }
    } else {
        return false;
    }
}
    
@Override
public boolean canContinueToUse() {
    LivingEntity target = getTarget();
    return target != null && target.isAlive() && attackWarmupDelay > 0;
}
```

`start`方法主要初始化了部分数值，并播放了准备施法的音效。其中出现的抽象方法马上会一起说。

```java
@Override
public void start() {
    // 这个AI并不requiresUpdateEveryTick，因此要adjustedTickDelay
    attackWarmupDelay = adjustedTickDelay(getCastWarmupTime());
    spellCastingTickCount = getCastingTime();
    nextAttackTickCount = tickCount + getCastingInterval();
    SoundEvent spellPrepareSound = getSpellPrepareSound();
    if (spellPrepareSound != null) {
        playSound(spellPrepareSound, 1.0F, 1.0F);
    }
    setIsCastingSpell(getSpell());
}
```

AI的更新比较简单。当施法所剩余的准备时间为0时会播放施法音效并应用法术效果。  
```java
@Override
public void tick() {
    --attackWarmupDelay;
    if (attackWarmupDelay == 0) {
        performSpellCasting();
        playSound(getCastingSoundEvent(), 1.0F, 1.0F);
    }
}
```

最后是一些抽象方法。
```java
// 决定施法的效果，也就是灾厄村民施法时会干什么。这个方法是施法AI中最核心的部分
protected abstract void performSpellCasting();

// 返回施法所需要的准备时间（单位：tick），也就是灾厄村民举起手并开始放出粒子效果，到真正施放法术所需的时间
protected int getCastWarmupTime() {
    return 20;
}

// 返回施法的（总）时间（单位：tick），也就是灾厄村民举起手并开始放出粒子效果，到放下手并停止放出粒子效果所需的时间
// 注意要让getCastWarmupTime小于等于getCastingTime
protected abstract int getCastingTime();
    
// 返回施法的冷却时间（单位：tick），即两次施放同一法术的最短间隔时间
protected abstract int getCastingInterval();

// 返回准备施法的音效，返回null则说明无准备音效
@Nullable
protected abstract SoundEvent getSpellPrepareSound();

// 返回该AI所代表的法术，也就是这是哪一种法术的AI
protected abstract IllagerSpell getSpell();
```

看到这里，读者也应该大致知道为什么实现一个新的施法类灾厄村民不应该直接继承`SpellcasterIllager`了吧。这是因为`IllagerSpell`是枚举类，我们无法添加新的`IllagerSpell`，而`IllagerSpell`在继承了`SpellcasterIllager`的施法类灾厄村民中是非常重要的，比如它决定了施法时的粒子效果的颜色，我们也必须重写`SpellcasterUseSpellGoal`中的`getSpell`方法并使其返回一个非空的值。因此，直接继承`SpellcasterIllager`无法实现新的拥有自定义法术的施法类灾厄村民，我们“从头开始”的时候也要自己写一个功能与`SpellcasterUseSpellGoal`类似的施法AI。

本节的内容就是这么多了，下一节我们将会介绍唤魔者的施法AI的具体实现逻辑。

