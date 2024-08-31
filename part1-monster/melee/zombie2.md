#僵尸的模型与渲染  

由于模型和渲染部分不是本教程的重点，并且僵尸的模型也不复杂，所以本节的内容会稍少。

下面是僵尸的模型。
```java
@OnlyIn(Dist.CLIENT)
public class ZombieModel<T extends Zombie> extends AbstractZombieModel<T> {
    public ZombieModel(ModelPart part) {
        super(part);
    }

    public boolean isAggressive(T zombie) {
        return zombie.isAggressive();
    }
}

@OnlyIn(Dist.CLIENT)
public abstract class AbstractZombieModel<T extends Monster> extends HumanoidModel<T> {
    protected AbstractZombieModel(ModelPart part) {
        super(part);
    }

    @Override
    public void setupAnim(T zombie, float limbSwing, float limbSwingAmount, float ageInTicks, float netHeadYaw, float headPitch) {
        super.setupAnim(zombie, limbSwing, limbSwingAmount, ageInTicks, netHeadYaw, headPitch);
        // 渲染僵尸的手臂的动画
        AnimationUtils.animateZombieArms(leftArm, rightArm, isAggressive(zombie), attackTime, ageInTicks);
    }

    public abstract boolean isAggressive(T zombie);
}
```
不难发现，僵尸的模型`ZombieModel`只是继承了`AbstractZombieModel`抽象类并实现了抽象方法`isAggressive`。而在`AbstractZombieModel`中，也只是在`HumanoidModel`（旧称`BipedModel`，用来提供人形的模型）的基础上调用了`AnimationUtils.animateZombieArms`方法来重写手臂的动画（僵尸的手臂不会自然下垂，攻击玩家时会双手同时攻击）。  
``@OnlyIn(Dist.CLIENT)``用来标记只在客户端存在的包，类，成员变量或方法等，实体模型只会在客户端用到，因此**不要省略这个注解**。

然后是僵尸的渲染。
```java
@OnlyIn(Dist.CLIENT)
public class ZombieRenderer extends AbstractZombieRenderer<Zombie, ZombieModel<Zombie>> {
    public ZombieRenderer(EntityRendererProvider.Context context) {
        this(context, ModelLayers.ZOMBIE, ModelLayers.ZOMBIE_INNER_ARMOR, ModelLayers.ZOMBIE_OUTER_ARMOR);
    }

    public ZombieRenderer(EntityRendererProvider.Context context, ModelLayerLocation mainModelLocation, ModelLayerLocation innerArmorModelLocation, ModelLayerLocation outerArmorModelLocation) {
        super(context, new ZombieModel<>(context.bakeLayer(mainModelLocation)), new ZombieModel<>(context.bakeLayer(innerArmorModelLocation)), new ZombieModel<>(context.bakeLayer(outerArmorModelLocation)));
    }
}

@OnlyIn(Dist.CLIENT)
public abstract class AbstractZombieRenderer<T extends Zombie, M extends ZombieModel<T>> extends HumanoidMobRenderer<T, M> {
    private static final ResourceLocation ZOMBIE_LOCATION = new ResourceLocation("textures/entity/zombie/zombie.png");

    protected AbstractZombieRenderer(EntityRendererProvider.Context context, M mainModel, M armorInnerModel, M armorOuterModel) {
  //    0.5F表示阴影的半径是0.5m
        super(context, mainModel, 0.5F);
        addLayer(new HumanoidArmorLayer<>(this, armorInnerModel, armorOuterModel, context.getModelManager()));
    }

    @Override
    public ResourceLocation getTextureLocation(Zombie zombie) {
        return ZOMBIE_LOCATION;
    }

    @Override
    protected boolean isShaking(T zombie) {
        return super.isShaking(zombie) || zombie.isUnderWaterConverting();
    }
}
```
这里僵尸的渲染器也只是继承了`AbstractZombieRenderer`。这儿说一下`AbstractZombieRenderer`的构造方法中的三个模型参数：
- 第一个参数表示的是**主要的模型**，也就是渲染僵尸材质时用的模型
- 第二个参数表示的是**盔甲的内层使用的模型**，这个模型只会在渲染护腿时使用
- 第三个参数表示的是**盔甲的外层使用的模型**，这个模型会在渲染除护腿外盔甲时使用  

同时三个模型的尺寸是依次增大的，这与玩家的2层皮肤相似。  
最后不要忘记当你写实体时，必须注册你的实体的渲染器。

僵尸的相关内容便告一段落了，下一节我们将讲一个近战怪物的实例~
