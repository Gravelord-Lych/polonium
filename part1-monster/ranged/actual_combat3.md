# 实战3 - 发射器僵尸

让我们仿照pvz里的豌豆僵尸制作一种新的僵尸吧~  

### 任务
**制作一种新的僵尸——发射器僵尸**

---

### 要求
1. 发射器僵尸除`Attributes.SPAWN_REINFORCEMENTS_CHANCE`以外的所有属性同普通的僵尸，而且也会在阳光下燃烧
2. 发射器僵尸除**不会以任何形式攻击海龟与海龟蛋**外，行为与普通的僵尸一致
3. 发射器僵尸对**玩家**和**铁傀儡**会**射箭**进行攻击，对**敌对生物**则只会对其**发射雪球**以进行警告  
4. 发射器僵尸攻击时**移速降低15%**，**每2s**攻击**1次**，攻击**半径**为**10格**
5. 当发射器僵尸感知到**自身所在的方块坐标四周有红石信号**时，**在四周有概率生成红石粒子效果**，且**每秒**恢复**1点**生命值，
6. 发射器僵尸在水中**不会转化为溺尸**，**若满足掉落头颅的条件**，击杀时会掉落**发射器**  
7. 发射器僵尸**不可能尝试生成增援**
8. 发射器僵尸**没有幼年状态**
9. 发射器僵尸**需要使用僵尸的材质和模型**
10. 发射器僵尸发射**任何**弹射物时都要使用**发射器发射成功**时的音效，其他音效可以任意设置
11. 与未经剪刀处理的雪傀儡相似，发射器僵尸会在**头部**渲染**发射器**的模型  

---

### 提示

*注：由于实战所制作实体的复杂程度的加大，笔者向教程中加入了“提示”部分，在提示部分中会对难度较大的内容进行提示或对“要求”部分进行补充说明*

- 发射**最普通的箭**的方式非常简单，与发射雪球类似，只要把实例化的雪球替换成箭即可。就像这样：
```java
Arrow arrow = new Arrow(level(), this);
arrow.shoot(...);
level().addFreshEntity(arrow);
```
制作发射器僵尸时不需要对箭做过多处理，但是当箭由弓发射时，需要根据所持有箭的种类与弓的种类来综合决定发射出的箭的类型，这一块在后续的章节中会进行讲解  

- 如何获取红石信号的大小呢？去`SignalGetter`类里寻找答案吧！

- 红石粒子效果的`ParticleOptions`（旧称`IParticleData`）是`DustParticleOptions.REDSTONE`

---

### 参考步骤
新建`DispenserZombie`类继承`Zombie`类，注意别忘了实现`RangedAttackMob`接口。
```java
public class DispenserZombie extends Zombie implements RangedAttackMob {
    public DispenserZombie(EntityType<? extends Zombie> type, Level level) {
        super(type, level);
    }
}

```
由要求1、7可知，发射器僵尸的`Attributes.SPAWN_REINFORCEMENTS_CHANCE`值应为0，不要忘了还要重写`randomizeReinforcementsChance`方法。
```java
public static AttributeSupplier.Builder createAttributes() {
    return Zombie.createAttributes()
            .add(Attributes.SPAWN_REINFORCEMENTS_CHANCE, 0);
}

@Override
protected void randomizeReinforcementsChance() {
    Objects.requireNonNull(getAttribute(Attributes.SPAWN_REINFORCEMENTS_CHANCE)).setBaseValue(0);
}
```
下面是AI。因为发射器僵尸不会攻击海龟与海龟蛋，所以要直接重写`registerGoals`方法。并且根据要求4，将相应的参数传给`RangedAttackGoal`的构造方法。
```java
@Override
protected void registerGoals() {
    goalSelector.addGoal(8, new LookAtPlayerGoal(this, Player.class, 8));
    goalSelector.addGoal(8, new RandomLookAroundGoal(this));
    addBehaviourGoals();
}

@Override
protected void addBehaviourGoals() {
    goalSelector.addGoal(2, new RangedAttackGoal(this, 0.85, 40, 10));
    goalSelector.addGoal(6, new MoveThroughVillageGoal(this, 1, true, 4, this::canBreakDoors));
    goalSelector.addGoal(7, new WaterAvoidingRandomStrollGoal(this, 1));
    targetSelector.addGoal(1, (new HurtByTargetGoal(this)).setAlertOthers(ZombifiedPiglin.class));
    targetSelector.addGoal(2, new NearestAttackableTargetGoal<>(this, Player.class, true));
    targetSelector.addGoal(3, new NearestAttackableTargetGoal<>(this, AbstractVillager.class, false));
    targetSelector.addGoal(3, new NearestAttackableTargetGoal<>(this, IronGolem.class, true));
}
```
接着是重要的`performRangedAttack`，其中`findProjectileFor`方法用于根据目标实体选择不同的弹射物。
```java
@Override
public void performRangedAttack(LivingEntity target, float power) {
    Projectile projectile = findProjectileFor(target);
    double targetY = target.getEyeY() - (double) 1.1F;
    double dx = target.getX() - this.getX();
    double dy = targetY - projectile.getY();
    double dz = target.getZ() - this.getZ();
    double yModifier = Math.sqrt(dx * dx + dz * dz) * (double) 0.2F;
    projectile.shoot(dx, yModifier + dy, dz, 1.6F, 12.0F);
//  此处修改了音效来达到要求10
    playSound(SoundEvents.DISPENSER_DISPENSE, 1, 0.4F / (getRandom().nextFloat() * 0.4F + 0.8F));
    level().addFreshEntity(projectile);
}

private Projectile findProjectileFor(LivingEntity target) {
    if (target instanceof Enemy) {
        return new Snowball(level(), this);
    }
    return new Arrow(level(), this);
}
```
接下来达到要求5。此处我们在发射器僵尸被tick时，调用`hasNeighborSignal`方法进行判断。
```java
@Override
public void tick() {
    super.tick();
    if (canHealSelf()) {
        if (level().isClientSide()) {
            maybeAddParticles();
        } else {
            healSelf();
        }
    }
}

private boolean canHealSelf() {
    BlockPos pos = blockPosition();
    return level().hasNeighborSignal(pos) || level().hasSignal(pos.below(), Direction.DOWN);
}

// 每tick都有10%的概率生成粒子效果
private void maybeAddParticles() {
    if (random.nextInt(10) == 0) {
        level().addParticle(DustParticleOptions.REDSTONE, getRandomX(1), getRandomY() + 0.5, getRandomZ(1), 0, 0, 0);
    }
}

// 当tickCount是20的倍数时进行治疗，这样可以保证tps为20时每秒治疗1点生命值
private void healSelf() {
    if (tickCount % 20 == 0) {
        heal(1);
    }
}
```
要达到要求6、8很简单，只需重写下面的方法即可。
```java
@Override
protected ItemStack getSkull() {
    return new ItemStack(Items.DISPENSER);
}

@Override
protected boolean convertsInWater() {
    return false;
}

@Override
public boolean isBaby() {
    return false;
}

@Override
public void setBaby(boolean baby) {}
```
除发射弹射物以外的音效这里均使用僵尸的音效，又因为`DispenserZombie`类继承了`Zombie`类，所以不需要重写那一部分决定音效的方法了。  

渲染类也很简单，但是要新的`Layer`以渲染发射器的模型。
```java
@OnlyIn(Dist.CLIENT)
public class DispenserZombieRenderer extends AbstractZombieRenderer<DispenserZombie, ZombieModel<DispenserZombie>> {
    public DispenserZombieRenderer(EntityRendererProvider.Context context) {
        super(context,
                new ZombieModel<>(context.bakeLayer(ModModelLayers.DISPENSER_ZOMBIE)),
                new ZombieModel<>(context.bakeLayer(ModModelLayers.DISPENSER_ZOMBIE_INNER_ARMOR)),
                new ZombieModel<>(context.bakeLayer(ModModelLayers.DISPENSER_ZOMBIE_OUTER_ARMOR)));
        addLayer(new DispenserZombieHeadLayer(this, context.getBlockRenderDispatcher(), context.getItemRenderer()));
    }
}
```

`DispenserZombieHeadLayer`只需要仿照`SnowGolemHeadLayer`来写就可以了。
```java
@OnlyIn(Dist.CLIENT)
public class DispenserZombieHeadLayer extends RenderLayer<DispenserZombie, ZombieModel<DispenserZombie>> {
    private final BlockRenderDispatcher blockRenderer;
    private final ItemRenderer itemRenderer;

    public DispenserZombieHeadLayer(RenderLayerParent<DispenserZombie, ZombieModel<DispenserZombie>> parent, BlockRenderDispatcher blockRenderer, ItemRenderer itemRenderer) {
        super(parent);
        this.blockRenderer = blockRenderer;
        this.itemRenderer = itemRenderer;
    }

    /**
     * [VanillaCopy]
     * <p>
     * {@link net.minecraft.client.renderer.entity.layers.SnowGolemHeadLayer#render(PoseStack, MultiBufferSource, int, SnowGolem, float, float, float, float, float, float)}
     */
    @SuppressWarnings("deprecation")
    @Override
    public void render(PoseStack poseStack, MultiBufferSource source, int packedLight, DispenserZombie zombie, float limbSwing, float limbSwingAmount, float partialTicks, float ageInTicks, float netHeadYaw, float headPitch) {
        boolean invisibleButGlowing = Minecraft.getInstance().shouldEntityAppearGlowing(zombie) && zombie.isInvisible();
        if (!zombie.isInvisible() || invisibleButGlowing) {
            poseStack.pushPose();
            getParentModel().getHead().translateAndRotate(poseStack);
            poseStack.translate(0.0F, -0.34375F, 0.0F);
            poseStack.mulPose(Axis.YP.rotationDegrees(180.0F));
            poseStack.scale(0.625F, -0.625F, -0.625F);
            Block dispenserHeadBlock = Blocks.DISPENSER;
            ItemStack itemStack = new ItemStack(dispenserHeadBlock);
            if (invisibleButGlowing) {
                BlockState dispenserHead = dispenserHeadBlock.defaultBlockState();
                BakedModel dispenserHeadModel = blockRenderer.getBlockModel(dispenserHead);
                int overlayCoords = LivingEntityRenderer.getOverlayCoords(zombie, 0.0F);
                poseStack.translate(-0.5F, -0.5F, -0.5F);
                blockRenderer.getModelRenderer().renderModel(poseStack.last(), source.getBuffer(RenderType.outline(TextureAtlas.LOCATION_BLOCKS)), dispenserHead, dispenserHeadModel, 0.0F, 0.0F, 0.0F, packedLight, overlayCoords);
            } else {
                itemRenderer.renderStatic(zombie, itemStack, ItemDisplayContext.HEAD, false, poseStack, source, zombie.level(), packedLight, LivingEntityRenderer.getOverlayCoords(zombie, 0.0F), zombie.getId());
            }
            poseStack.popPose();
        }
    }
}
```
最后不要忘记实体的注册。  

到此我们就做完了发射器僵尸的全部内容。  

[源代码（`DispenserZombie`类）](https://github.com/Gravelord-Lych/polonium-ExampleMod/blob/main/src/main/java/lych/polonium/entity/monster/DispenserZombie.java)  
[源代码（`DispenserZombieRenderer`类）](https://github.com/Gravelord-Lych/polonium-ExampleMod/blob/main/src/main/java/lych/polonium/client/renderer/DispenserZombieRenderer.java)  
[源代码（`DispenserZombieHeadLayer`类）](https://github.com/Gravelord-Lych/polonium-ExampleMod/blob/main/src/main/java/lych/polonium/client/layer/DispenserZombieHeadLayer.java)

---

### 效果图
*发射器僵尸攻击铁傀儡，同时在有红石信号处恢复自身生命值*
![发射器僵尸攻击铁傀儡，同时在有红石信号处恢复自身生命值](images/attack.webp)
*发射器僵尸用雪球警告攻击自己的骷髅*
![发射器僵尸用雪球警告攻击自己的骷髅](images/warn.webp)

---

### 思考与练习  
- 测试实体时不难发现，**发射器僵尸使用雪球进行攻击时，雪球有时会发射得偏高**。是否可以通过调整相关的参数修复这一问题？ 
- 能否让发射器僵尸也拥有对应的幼年形态，且**小型发射器僵尸的发射器头颅也会缩小**？
- 能否让发射器僵尸**发射雪球的速度变为发射箭的速度的2倍**？（提示：需要写一个新的类似`RangedAttackGoal`的AI来实现）
- 能否让发射器僵尸在附近**5x3x5**区域内检测到**红石块**或**点燃的红石火把**时也能治疗自身？
