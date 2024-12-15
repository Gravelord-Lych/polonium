# 雪傀儡的实现逻辑

为什么从雪傀儡讲起呢？这是因为虽然骷髅一类的生物是我们更为熟悉的远程攻击生物，但是骷髅的行为较为复杂，例如骷髅在玩家靠近时会尝试走离玩家，而且骷髅不只可以使用弓，还可以近战攻击，从骷髅入手会大大增加教程的复杂程度，引入较多与主题无关的内容且不利于读者理解。雪傀儡的行为则相对简单很多，单个雪傀儡所使用的`Goal`也只有5个，从雪傀儡讲起可以简化教程并使文章与本章节主题的关联性更强。  

下面是一些之前曾经讲解过的、各种生物共有的或者易理解的部分，此处不再做详细解释。雪傀儡只有一个`EntityDataAccessor`，用于判断雪傀儡是否有南瓜头。  

```java
// 如果值为16则说明雪傀儡有南瓜头，尚不明确此处为什么不用Boolean。
private static final EntityDataAccessor<Byte> DATA_PUMPKIN_ID = SynchedEntityData.defineId(SnowGolem.class, EntityDataSerializers.BYTE);

public static AttributeSupplier.Builder createAttributes() {
     return Mob.createMobAttributes()
         .add(Attributes.MAX_HEALTH, 4.0D)
         .add(Attributes.MOVEMENT_SPEED, (double) 0.2F);
}

protected void defineSynchedData() {
    super.defineSynchedData();
    entityData.define(DATA_PUMPKIN_ID, (byte) 16);
}

public void addAdditionalSaveData(CompoundTag tag) {
    super.addAdditionalSaveData(tag);
    tag.putBoolean("Pumpkin", hasPumpkin());
}

public void readAdditionalSaveData(CompoundTag tag) {
    super.readAdditionalSaveData(tag);
    if (tag.contains("Pumpkin")) {
        setPumpkin(tag.getBoolean("Pumpkin"));
    }
}

public boolean isSensitiveToWater() {
    return true;
}

@Nullable
protected SoundEvent getAmbientSound() {
    return SoundEvents.SNOW_GOLEM_AMBIENT;   
}

@Nullable
protected SoundEvent getHurtSound(DamageSource source) {
    return SoundEvents.SNOW_GOLEM_HURT;
}

@Nullable
protected SoundEvent getDeathSound() {
    return SoundEvents.SNOW_GOLEM_DEATH;
}

// 下面的部分使用了位运算，用于表示与修改雪傀儡的状态，如有不理解的可以去阅读相关内容
public boolean hasPumpkin() {
     return (entityData.get(DATA_PUMPKIN_ID) & 16) != 0;
}

public void setPumpkin(boolean pumpkin) {
    byte value = entityData.get(DATA_PUMPKIN_ID);
    if (pumpkin) {
        entityData.set(DATA_PUMPKIN_ID, (byte) (value | 16));
    } else {
        entityData.set(DATA_PUMPKIN_ID, (byte) (value & -17));
    }
}
```  

再来看AI。
```java
protected void registerGoals() {
    goalSelector.addGoal(1, new RangedAttackGoal(this, 1.25D, 20, 10.0F));
    goalSelector.addGoal(2, new WaterAvoidingRandomStrollGoal(this, 1.0D, 1.0000001E-5F)); // 1.0000001E-5F是散步的可能性，暂时不清楚为什么不直接使用1E-5F
    goalSelector.addGoal(3, new LookAtPlayerGoal(this, Player.class, 6.0F));
    goalSelector.addGoal(4, new RandomLookAroundGoal(this));
    targetSelector.addGoal(1, new NearestAttackableTargetGoal<>(this, Mob.class, 10, true, false, target -> {
       return target instanceof Enemy;
    }));
}
```
注意一下RangedAttackGoal的参数。第二个参数`speedModifier`传入了1.25，意味着雪傀儡在攻击时的速度是基础移速的125%。其余参数的含义见1.2.2.1章节。

接下来是雪傀儡实现远程攻击的核心部分——`performRangedAttack`，以下的内容**至关重要**，几乎所有远程攻击的底层实现都是下面代码的变体。  

```java
public void performRangedAttack(LivingEntity target, float power) {
    Snowball snowball = new Snowball(level(), this);
    double targetY = target.getEyeY() - (double) 1.1F;
    double dx = target.getX() - this.getX();
    double dy = targetY - snowball.getY();
    double dz = target.getZ() - this.getZ();
//  编写发射火球一类弹射物的代码时，这下面的部分会稍有差别，我们以后再讲
    double yModifier = Math.sqrt(dx * dx + dz * dz) * (double) 0.2F;
    snowball.shoot(dx, yModifier + dy, dz, 1.6F, 12.0F);
    playSound(SoundEvents.SNOW_GOLEM_SHOOT, 1, 0.4F / (getRandom().nextFloat() * 0.4F + 0.8F));
    level().addFreshEntity(snowball);
}
```
这段代码首先实例化了一个`Snowball`，代表了将要投掷出的雪球实体。  

下面一段十分关键：
```java
double dx = target.getX() - this.getX();
double dy = targetY - snowball.getY();
double dz = target.getZ() - this.getZ();
```
设玩家坐标为$$(x_1, y_1, z_1)$$，雪傀儡坐标为$$(x_2, y_2, z_2)$$，以下部分计算出了向量$$(x_2-x_1, y_2-y_1, z_2-z_1)$$的x、y及z坐标，这个向量与弹射物的飞行轨迹关系密切。对于雪球等继承了`ThrowableProjectile`的弹射物以及各种箭而言，这个向量**决定了弹射物被射出时的方向与高度**；对于各种火球（包括末影龙火球）、凋灵之首等继承了`AbstractHurtingProjectile`的弹射物而言，这个向量则**直接决定了弹射物的轨迹所在的直线**。此处我们先分析前一种情况，后一种以后再做分析。  

计算好了发射的方向，接下来就应该把相应的属性应用给弹射物。`Projectile`类里的`shoot`方法已经帮助我们做好了这些准备。  
```java
double yModifier = Math.sqrt(dx * dx + dz * dz) * (double) 0.2F;
snowball.shoot(dx, yModifier + dy, dz, 1.6F, 12.0F);
```
这里我们计算了到目标的距离，为什么要计算距离呢？根据物理知识，物体做斜抛运动时，**若初速度方向与地面的夹角小于45°，则开始时抛得越高，最终扔得越远**，在MC中也可以认为符合这个规律。因此当目标离雪傀儡较远时，需要把雪球扔高一些，从而能击中较远的目标。同时我们将到目标的距离乘上了0.2对dy（即上文所说的$$y_2-y_1$$）进行修正，来防止扔得过高使射程反而变短。  

但是`shoot`方法还有2个参数，那么后两个`float`类型的参数是干什么的呢？  

`Projectile`：
```java
public void shoot(double x, double y, double z, float scale, float deviation) {
    Vec3 shootVector = new Vec3(x, y, z).normalize()
            .add(random.triangle(0, 0.0172275 * (double) deviation),
                    random.triangle(0, 0.0172275 * (double) deviation),
                    random.triangle(0, 0.0172275 * (double) deviation))
            .scale(scale);
    setDeltaMovement(shootVector);
    double horizontalDistance = shootVector.horizontalDistance();
    setYRot((float) (Mth.atan2(shootVector.x, shootVector.z) * (double) (180F / (float) Math.PI)));
    setXRot((float) (Mth.atan2(shootVector.y, horizontalDistance) * (double) (180F / (float) Math.PI)));
    yRotO = getYRot();
    xRotO = getXRot();
}
```

`RandomSource`（用法与`Random`类相近）：  
```java
default double triangle(double baseValue, double scale) {
    return baseValue + scale * (nextDouble() - nextDouble());
}
```
查阅源码可以发现，倒数第二个参数`scale`决定了扔出弹射物时的“力量”，值越大，则`shootVector`的模越大，扔得就越高、越远；最后一个参数`deviation`决定了射击的误差，值越大，设计的误差越大，射击精准度越低。  

下表给出了其他一些常见生物攻击时使用的`scale`和`deviation`（其中k为难度ID，和平、简单、普通、困难难度的k值分别为0，1，2，3）：

|  生物  | scale | deviation |
|:--------------:|:------------------:|:-------------------:|
|  骷髅（射箭）  |  1.6  | 14 - 4k |
|  掠夺者（射箭）  |  1.6  | 14 - 4k |
|  溺尸（三叉戟）  |  1.6  | 14 - 4k |
|  女巫（投掷药水）  |  0.75 | 8 |
|  羊驼（吐口水）  |  1.5  | 10 |

最后两行是播放声音和添加实体。
```java
playSound(SoundEvents.SNOW_GOLEM_SHOOT, 1, 0.4F / (getRandom().nextFloat() * 0.4F + 0.8F));
level().addFreshEntity(snowball);
```
调用`playSound`方法时需要提供声音种类（`SoundEvent`）、音量和音调。**一般情况下都要对音调进行一定的随机化处理，使声音更加自然**。  

而对于`addFreshEntity`方法的使用，有以下的注意事项：**一个实体只能被add一次，如果想要生成多个实体，则每次生成都要重新实例化实体**。这看似显然，但若不清楚其中的原因则极易犯错。

举生成3只僵尸为例，代码应该这样写（此处省略对僵尸的一些必要操作，如更改僵尸坐标）：
```java
for (int i = 0; i < 3; i++) {
    Zombie zombie = EntityType.ZOMBIE.create(level);
    level.addFreshEntity(zombie);
}
```
而不是这样写：
```java
Zombie zombie = EntityType.ZOMBIE.create(level);
for (int i = 0; i < 3; i++) {
    level.addFreshEntity(zombie);
}
```
如果采用下面这种错误的写法，日志中就会输出“UUID of added entity already exists”且实际只会生成1只僵尸，因为每次实例化实体的同时，也给予了实体一个唯一的UUID。这种错误的写法会使后生成的僵尸的UUID与第一只僵尸的UUID重复，违背了实体UUID的唯一性。

然后是处理用剪刀与雪傀儡交互部分的代码。
```java
//  这个方法的代码原本不是这样，但被forge处理过后等价于下面四行
@Override
protected InteractionResult mobInteract(Player player, InteractionHand hand) {
    return InteractionResult.PASS;
}

@Override
public void shear(SoundSource source) {
    level().playSound(null, this, SoundEvents.SNOW_GOLEM_SHEAR, source, 1, 1);
    if (!level().isClientSide()) {
        setPumpkin(false);
        spawnAtLocation(new ItemStack(Items.CARVED_PUMPKIN), 1.7F);
    }
}

@Override
public boolean readyForShearing() {
    return isAlive() && hasPumpkin();
}

@Override
public boolean isShearable(@NotNull ItemStack item, Level world, BlockPos pos) {
    return readyForShearing();
}

@NotNull
@Override
public List<ItemStack> onSheared(@Nullable Player player, @NotNull ItemStack item, Level world, BlockPos pos, int fortune) {
    world.playSound(null, this, SoundEvents.SNOW_GOLEM_SHEAR, player == null ? SoundSource.BLOCKS : SoundSource.PLAYERS, 1, 1);
    gameEvent(GameEvent.SHEAR, player);
    if (!world.isClientSide()) {
        setPumpkin(false);
        return Collections.singletonList(new ItemStack(Items.CARVED_PUMPKIN));
    }
    return Collections.emptyList();
}
```
这部分的难度不大，如果读者需要编写玩家手持剪刀时可以交互的实体时，可以实现`Shearable`接口并参考这部分代码，而不是重写`mobInteract`后按自己的想法实现。  

最后还有一个`getLeashOffset`方法。
```java
@Override
public Vec3 getLeashOffset() {
    return new Vec3(0, (double) (0.75F * getEyeHeight()), (double) (getBbWidth() * 0.4F));
}
```
`Entity`类中的定义是这样的：
```java
protected Vec3 getLeashOffset() {
    return new Vec3(0, (double) getEyeHeight(), (double) (getBbWidth() * 0.4F));
}
```
该方法返回的向量的x、y、z分别决定了渲染拴绳时拴住实体的位置在**左右、上下、前后**方向的偏移。拴住雪傀儡时，我们希望拴住的位置不要太高，于是重写了该方法并将返回值的y坐标乘上了0.75。  

到这里`SnowGolem`类的内容就分析完了，下一节将讲解雪傀儡的模型和渲染。
