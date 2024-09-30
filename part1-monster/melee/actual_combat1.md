# 实战1 - 强化僵尸

***注：***
*为避免实战部分过于复杂与偏离主题，除特殊说明外，此部分对资源包与数据包部分（如掉落物、语言文件等）不作要求，感兴趣的读者可以学习相关的资源包、数据包相关知识，以及`DataGenerator`使用教程（非必须）后自行添加。*

---

让我们从一个最简单的实例开始。  

### 任务
**制作一种新的僵尸——强化僵尸**

---

### 要求
1. 强化僵尸的属性同普通僵尸，但有**40的最大生命值**  
2. 强化僵尸的生命值**低于（<）最大生命值的一半**时，在阳光下**不会燃烧**  
3. 被强化僵尸伤害的**非怪物**生物会获得**20s的缓慢II**状态效果（持续时间和区域难度无关）  
4. 强化僵尸在水中**不会转化为溺尸**，击杀时也**不会掉落头颅**  
5. 强化僵尸**一定会尝试生成增援的强化僵尸**，但前来**增援的强化僵尸不会再次生成增援**，且**一个强化僵尸最多尝试生成增援3次**
6. 强化僵尸**需要使用僵尸的模型**，但音效和材质可以任意设置  

---

### 参考步骤
首先创建`ReinforcementZombie`类，因为强化僵尸是一种特殊的僵尸，所以继承`Zombie`类。  
```java
public class ReinforcedZombie extends Zombie {
    public ReinforcedZombie(EntityType<? extends ReinforcedZombie> type, Level level) {
        super(type, level);
    }
}
```
以及`createAttributes`方法。  
```java
public static AttributeSupplier.Builder createAttributes() {
    return Zombie.createAttributes().add(Attributes.MAX_HEALTH, 40);
}
```
因为“强化僵尸的生命值低于（<）最大生命值的一半时，在阳光下不会燃烧”，所以重写`isSunSensitive`方法。
```java
@Override
protected boolean isSunSensitive() {
    return getHealth() >= getMaxHealth() * 0.5F;
}
```  
然后重写`doHurtTarget`方法，实现添加缓慢II的效果。
```java
@Override
public boolean doHurtTarget(Entity target) {
    boolean hurt = super.doHurtTarget(target);
    if (hurt && target instanceof LivingEntity living && !(target instanceof Enemy)) {
        living.addEffect(new MobEffectInstance(MobEffects.MOVEMENT_SLOWDOWN, 20 * 20, 1));
    }
    return hurt;
}
```
这里我们通过调用`addEffect`方法并实例化了一个`MobEffectInstance`，给非怪物生物添加了20s的缓慢II效果（`MobEffectInstance`构造方法中的第三个参数是“倍率”而非等级，而**倍率比等级小1**）。  
比较复杂的是要求5，我们先**用一个成员变量保存剩余的尝试生成增援的次数**。
```java
private static final String AIDS_REMAINING = "aids_remaining";
private int aidsRemaining = 3;

public int getAidsRemaining() {
    return aidsRemaining;
}

public void setAidsRemaining(int aidsRemaining) {
    this.aidsRemaining = aidsRemaining;
}
// 不要忘了数据保存！
@Override
public void addAdditionalSaveData(CompoundTag tag) {
    super.addAdditionalSaveData(tag);
    tag.putInt(AIDS_REMAINING, getAidsRemaining());
}

@Override
public void readAdditionalSaveData(CompoundTag tag) {
    super.readAdditionalSaveData(tag);
    if (tag.contains(AIDS_REMAINING, CompoundTag.TAG_ANY_NUMERIC)) {
        setAidsRemaining(tag.getInt(AIDS_REMAINING));
    }
}
```
根据要求5，直接使用`Attributes.SPAWN_REINFORCEMENTS_CHANCE`来控制增援的生成是不行的。我们选择**用事件监听器，并监听ZombieEvent.SummonAidEvent**。
```java
@Mod.EventBusSubscriber(modid = Polonium.MOD_ID)
public final class EntityEventListener {
    @SubscribeEvent
    public static void onZombieSummonAid(ZombieEvent.SummonAidEvent event) {
        if (event.getEntity() instanceof ReinforcedZombie reinforcedZombie) {
            if (reinforcedZombie.getAidsRemaining() <= 0) {
            //  强制取消召唤
                event.setResult(Event.Result.DENY);
                return;
            }
            ReinforcedZombie aid = ModEntities.REINFORCED_ZOMBIE.get().create(event.getLevel());
            if (aid != null) {
            //  前来增援的强化僵尸不会再次生成增援
                aid.setAidsRemaining(0);
            //  强制允许召唤
                event.setResult(Event.Result.ALLOW);
                event.setCustomSummonedAid(aid);
                reinforcedZombie.setAidsRemaining(reinforcedZombie.getAidsRemaining() - 1);
            }
        }
    }
}
```
要求4很容易达到，重写下列2个方法即可。
```java
@Override
protected boolean convertsInWater() {
    return false;
}

@Override
protected ItemStack getSkull() {
    return ItemStack.EMPTY;
}
```
最后完善强化僵尸的音效。（此处使用了尸壳的音效）
```java
@Override
protected SoundEvent getAmbientSound() {
    return SoundEvents.HUSK_AMBIENT;
}

@Override
protected SoundEvent getHurtSound(DamageSource source) {
    return SoundEvents.HUSK_HURT;
}

@Override
protected SoundEvent getDeathSound() {
    return SoundEvents.HUSK_DEATH;
}

@Override
protected SoundEvent getStepSound() {
    return SoundEvents.HUSK_STEP;
}
```
接下来注册我们的实体。
```java
@Mod.EventBusSubscriber(modid = Polonium.MOD_ID, bus = Mod.EventBusSubscriber.Bus.MOD)
public final class ModEntities {
    public static final DeferredRegister<EntityType<?>> ENTITIES = DeferredRegister.create(ForgeRegistries.ENTITY_TYPES, Polonium.MOD_ID);
    public static final RegistryObject<EntityType<ReinforcedZombie>> REINFORCED_ZOMBIE = ENTITIES.register(ModEntityNames.REINFORCED_ZOMBIE,
            () -> EntityType.Builder.of(ReinforcedZombie::new, MobCategory.MONSTER).sized(0.6F, 1.95F).clientTrackingRange(8).build(ModEntityNames.REINFORCED_ZOMBIE));

    @SubscribeEvent
    public static void registerAttributes(EntityAttributeCreationEvent event) {
        event.put(REINFORCED_ZOMBIE.get(), ReinforcedZombie.createAttributes().build());
    }

    private ModEntities() {}
}
```
对于模型，直接使用僵尸的模型即可，下面是渲染部分。
```java
public class ReinforcedZombieRenderer extends ZombieRenderer {
    private static final ResourceLocation REINFORCED_ZOMBIE = Utils.prefix("textures/entity/reinforced_zombie.png");

    public ReinforcedZombieRenderer(EntityRendererProvider.Context context) {
        super(context,
                ModModelLayers.REINFORCED_ZOMBIE,
                ModModelLayers.REINFORCED_ZOMBIE_INNER_ARMOR,
                ModModelLayers.REINFORCED_ZOMBIE_OUTER_ARMOR);
    }

    @Override
    public ResourceLocation getTextureLocation(Zombie zombie) {
        return REINFORCED_ZOMBIE;
    }
}

//  ModelLayer
public final class ModModelLayers {
    public static final ModelLayerLocation REINFORCED_ZOMBIE = createMain(ModEntityNames.REINFORCED_ZOMBIE);
    public static final ModelLayerLocation REINFORCED_ZOMBIE_INNER_ARMOR = createInnerArmor(ModEntityNames.REINFORCED_ZOMBIE);
    public static final ModelLayerLocation REINFORCED_ZOMBIE_OUTER_ARMOR = createOuterArmor(ModEntityNames.REINFORCED_ZOMBIE);

    private ModModelLayers() {}

    private static ModelLayerLocation createMain(String model) {
        return create(model, "main");
    }

    private static ModelLayerLocation createInnerArmor(String model) {
        return create(model, "inner_armor");
    }

    private static ModelLayerLocation createOuterArmor(String model) {
        return create(model, "outer_armor");
    }

    private static ModelLayerLocation create(String model, String layer) {
    //  这里的Utils.prefix方法返回new ResourceLocation(Polonium.MOD_ID, model)
        return new ModelLayerLocation(Utils.prefix(model), layer);
    }
}
```
别忘了注册实体渲染器以及`LayerDefinition`。
```java
@Mod.EventBusSubscriber(modid = Polonium.MOD_ID, bus = Mod.EventBusSubscriber.Bus.MOD, value = Dist.CLIENT)
public class ModEntityRenderers {
    @SubscribeEvent
    public static void registerEntityRenderers(EntityRenderersEvent.RegisterRenderers event) {
        event.registerEntityRenderer(ModEntities.REINFORCED_ZOMBIE.get(), ReinforcedZombieRenderer::new);
    }

    @SubscribeEvent
    public static void registerLayerDefinitions(EntityRenderersEvent.RegisterLayerDefinitions event) {
        Supplier<LayerDefinition> zombie = () -> LayerDefinition.create(HumanoidModel.createMesh(CubeDeformation.NONE, 0.0F), 64, 64);
        Supplier<LayerDefinition> innerArmor = () -> LayerDefinition.create(HumanoidArmorModel.createBodyLayer(LayerDefinitions.INNER_ARMOR_DEFORMATION), 64, 32);
        Supplier<LayerDefinition> outerArmor = () -> LayerDefinition.create(HumanoidArmorModel.createBodyLayer(LayerDefinitions.OUTER_ARMOR_DEFORMATION), 64, 32);
        event.registerLayerDefinition(ModModelLayers.REINFORCED_ZOMBIE, zombie);
        event.registerLayerDefinition(ModModelLayers.REINFORCED_ZOMBIE_INNER_ARMOR, innerArmor);
        event.registerLayerDefinition(ModModelLayers.REINFORCED_ZOMBIE_OUTER_ARMOR, outerArmor);
    }
}
```
到此我们就做完了强化僵尸的全部内容。  

[源代码（`ReinforcedZombie`类）](https://github.com/Gravelord-Lych/polonium-ExampleMod/blob/main/src/main/java/lych/polonium/entity/monster/ReinforcedZombie.java)  
[源代码（`ReinforcedZombieRenderer`类）](https://github.com/Gravelord-Lych/polonium-ExampleMod/blob/main/src/main/java/lych/polonium/client/renderer/ReinforcedZombieRenderer.java)

---

### 效果图（强化僵尸的材质使用了红眼的僵尸）
*强化僵尸攻击玩家*
![强化僵尸攻击玩家](images/result.webp)

---

### 思考与练习  
*此部分内容仅提供了思考方向，不给出解答与源代码*  
- 能否让强化僵尸造成的缓慢II效果的持续时间**与区域难度呈正相关**呢？  
- 能否让强化僵尸**主动攻击骷髅**？  
- 能否让强化僵尸**泡在水中30s后转化为尸壳**？
- 能否让强化僵尸掉落**2倍于（普通）僵尸**的xp？
