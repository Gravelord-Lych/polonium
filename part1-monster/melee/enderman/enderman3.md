#末影人的模型与渲染  

本节将主要分析末影人的模型的特殊之处，以及末影人的渲染器中使用的`Layer`*（`RenderLayer`，不是旧版生物群系使用的`Layer`）*

先是成员变量与构造方法。
```java
@OnlyIn(Dist.CLIENT)
public class EndermanModel<T extends LivingEntity> extends HumanoidModel<T> {
//  决定了末影人是否在搬着方块，值为true则表示正在搬着方块
    public boolean carrying;
//  决定了末影人是否处于愤怒状态，值为true则意味着处于愤怒状态
    public boolean creepy;

    public EndermanModel(ModelPart part) {
        super(part);
    }
}
```
因为末影人有2组不同的状态（生气-不生气，搬着方块-空手），所以模型中有两个公共且可变的boolean成员变量，用来控制末影人的模型。*实际开发中，不提倡这种暴露成员变量的方式，尽量使用getter和setter。*  

然后是静态方法`createBodyLayer`，因为末影人的模型与标准的“人类模型”不同，所以要重新写一个`LayerDefinition`。
```java
public static LayerDefinition createBodyLayer() {
    MeshDefinition meshDef = HumanoidModel.createMesh(CubeDeformation.NONE, -14.0F);
    PartDefinition partDef = meshDef.getRoot();
    PartPose head = PartPose.offset(0.0F, -13.0F, 0.0F); // part的相对位置（x, y, z）
    partDef.addOrReplaceChild("hat",
//          此处与较早的MC版本定义模型的方式不同，材质位置，box的大小、位置、缩放比例都在PartDefinition中而不是ModelPart中定义
            CubeListBuilder.create()
                    .texOffs(0, 16) // 材质的uv位置
                    .addBox(-4.0F, -8.0F, -4.0F, // box的相对位置（x, y, z）
                            8.0F, 8.0F, 8.0F, // box的大小（x, y, z）
                            new CubeDeformation(-0.5F) // 缩放比例（正值放大，负值收缩）
                    ),
            head);
    partDef.addOrReplaceChild("head", CubeListBuilder.create().texOffs(0, 0).addBox(-4.0F, -8.0F, -4.0F, 8.0F, 8.0F, 8.0F), head);
    partDef.addOrReplaceChild("body", CubeListBuilder.create().texOffs(32, 16).addBox(-4.0F, 0.0F, -2.0F, 8.0F, 12.0F, 4.0F), PartPose.offset(0.0F, -14.0F, 0.0F));
    partDef.addOrReplaceChild("right_arm", CubeListBuilder.create().texOffs(56, 0).addBox(-1.0F, -2.0F, -1.0F, 2.0F, 30.0F, 2.0F), PartPose.offset(-5.0F, -12.0F, 0.0F));
    partDef.addOrReplaceChild("left_arm", CubeListBuilder.create().texOffs(56, 0).mirror().addBox(-1.0F, -2.0F, -1.0F, 2.0F, 30.0F, 2.0F), PartPose.offset(5.0F, -12.0F, 0.0F));
    partDef.addOrReplaceChild("right_leg", CubeListBuilder.create().texOffs(56, 0).addBox(-1.0F, 0.0F, -1.0F, 2.0F, 30.0F, 2.0F), PartPose.offset(-2.0F, -5.0F, 0.0F));
    partDef.addOrReplaceChild("left_leg", CubeListBuilder.create().texOffs(56, 0).mirror().addBox(-1.0F, 0.0F, -1.0F, 2.0F, 30.0F, 2.0F), PartPose.offset(2.0F, -5.0F, 0.0F));
    return LayerDefinition.create(meshDef, 64, 32); // 模型使用的材质的尺寸
}
```
这一部分声明了末影人使用的模型，虽然比较抽象，但理解难度不是特别大。**只要生物的模型与继承的父类提供的模型有出入**（例如末影人的模型和一般的`HumanoidModel`不同）**，就需要重新写一个静态方法，用来创建LayerDefinition**。同时别忘了注册`LayerDefinition`。  

然后是`setUpAnim`，末影人的手臂的动画与other人形怪物不一样，所以需要重写这个方法。为方便阅读，调换了可调换的语句并删除了例如`x -= 0`的无意义的部分。
```java
@Override
public void setupAnim(T enderman, float limbSwing, float limbSwingAmount, float ageInTicks, float netHeadYaw, float headPitch) {
    super.setupAnim(enderman, limbSwing, limbSwingAmount, ageInTicks, netHeadYaw, headPitch);

    head.visible = true;
    body.xRot = 0.0F;
    body.y = -14.0F;
    body.z = -0.0F;
    rightLeg.z = 0.0F;
    leftLeg.z = 0.0F;
    rightLeg.y = -5.0F;
    leftLeg.y = -5.0F;
    head.z = -0.0F;
    head.y = -13.0F;
    hat.x = head.x;
    hat.y = head.y;
    hat.z = head.z;
    hat.xRot = head.xRot;
    hat.yRot = head.yRot;
    hat.zRot = head.zRot;
    rightArm.setPos(-5.0F, -12.0F, 0.0F);
    leftArm.setPos(5.0F, -12.0F, 0.0F);

    rightArm.xRot *= 0.5F;
    leftArm.xRot *= 0.5F;
    rightLeg.xRot *= 0.5F;
    leftLeg.xRot *= 0.5F;

    if (rightArm.xRot > 0.4F) {
        rightArm.xRot = 0.4F;
    }
    if (leftArm.xRot > 0.4F) {
        leftArm.xRot = 0.4F;
    }
    if (rightArm.xRot < -0.4F) {
        rightArm.xRot = -0.4F;
    }
    if (leftArm.xRot < -0.4F) {
        leftArm.xRot = -0.4F;
    }
    if (rightLeg.xRot > 0.4F) {
        rightLeg.xRot = 0.4F;
    }
    if (leftLeg.xRot > 0.4F) {
        leftLeg.xRot = 0.4F;
    }
    if (rightLeg.xRot < -0.4F) {
        rightLeg.xRot = -0.4F;
    }
    if (leftLeg.xRot < -0.4F) {
        leftLeg.xRot = -0.4F;
    }

    if (carrying) {
        rightArm.xRot = -0.5F;
        leftArm.xRot = -0.5F;
        rightArm.zRot = 0.05F;
        leftArm.zRot = -0.05F;
    }

    if (creepy) {
        head.y -= 5.0F;
    }    
}
```
首先是模型box的重置。之所以要这样重置一遍，是因为末影人的模型和一般的`HumanoidModel`不同，因此box的位置也会有差异，需要在**调用super方法后**重置。  
```java
head.visible = true;
body.xRot = 0.0F;
body.y = -14.0F;
body.z = -0.0F;
rightLeg.z = 0.0F;
leftLeg.z = 0.0F;
rightLeg.y = -5.0F;
leftLeg.y = -5.0F;
head.z = -0.0F;
head.y = -13.0F;
hat.x = head.x;
hat.y = head.y;
hat.z = head.z;
hat.xRot = head.xRot;
hat.yRot = head.yRot;
hat.zRot = head.zRot;
rightArm.setPos(-5.0F, -12.0F, 0.0F);
leftArm.setPos(5.0F, -12.0F, 0.0F);
```
接着是绕x轴旋转角度的限制，将`xRot`减半并把|`xRot`|限制在0.4，用于**减小动作幅度**，防止末影人做出一些“出格”的动作。  
```java
rightArm.xRot *= 0.5F;
leftArm.xRot *= 0.5F;
rightLeg.xRot *= 0.5F;
leftLeg.xRot *= 0.5F;

if (rightArm.xRot > 0.4F) {
    rightArm.xRot = 0.4F;
}
if (leftArm.xRot > 0.4F) {
    leftArm.xRot = 0.4F;
}
if (rightArm.xRot < -0.4F) {
    rightArm.xRot = -0.4F;
}
if (leftArm.xRot < -0.4F) {
    leftArm.xRot = -0.4F;
}
if (rightLeg.xRot > 0.4F) {
    rightLeg.xRot = 0.4F;
}
if (leftLeg.xRot > 0.4F) {
    leftLeg.xRot = 0.4F;
}
if (rightLeg.xRot < -0.4F) {
    rightLeg.xRot = -0.4F;
}
if (leftLeg.xRot < -0.4F) {
    leftLeg.xRot = -0.4F;
}
```
最后就是特殊状态下的姿势调整。
```java
// 搬运方块时手臂的角度调整
if (carrying) {
    rightArm.xRot = -0.5F;
    leftArm.xRot = -0.5F;
    rightArm.zRot = 0.05F;
    leftArm.zRot = -0.05F;
}

// 生气时嘴巴下移
if (creepy) {
    head.y -= 5.0F;
}    
```
模型的内容就是这么多了，接下来是渲染器。
```java
@OnlyIn(Dist.CLIENT)
public class EndermanRenderer extends MobRenderer<EnderMan, EndermanModel<EnderMan>> {
    private static final ResourceLocation ENDERMAN_LOCATION = new ResourceLocation("textures/entity/enderman/enderman.png");
    private final RandomSource random = RandomSource.create();

    public EndermanRenderer(EntityRendererProvider.Context context) {
        super(context, new EndermanModel<>(context.bakeLayer(ModelLayers.ENDERMAN)), 0.5F);
        addLayer(new EnderEyesLayer<>(this));
        addLayer(new CarriedBlockLayer(this, context.getBlockRenderDispatcher()));
    }

    @Override
    public void render(EnderMan enderman, float yRot, float partialTicks, PoseStack stack, MultiBufferSource source, int packedLight) {
        BlockState carriedBlockState = enderman.getCarriedBlock();
        EndermanModel<EnderMan> model = getModel();
        model.carrying = carriedBlockState != null;
        model.creepy = enderman.isCreepy();
        super.render(enderman, yRot, partialTicks, stack, source, packedLight);
    }

    @Override
    public Vec3 getRenderOffset(EnderMan enderman, float partialTicks) {
        if (enderman.isCreepy()) {
            double scale = 0.02D;
            return new Vec3(random.nextGaussian() * 0.02D, 0.0D, random.nextGaussian() * 0.02D);
        } else {
            return super.getRenderOffset(enderman, partialTicks);
        }
    }

    @Override
    public ResourceLocation getTextureLocation(EnderMan enderman) {
        return ENDERMAN_LOCATION;
    }
}
```
构造方法中添加了`EnderEyesLayer`与`CarriedBlockLayer`这两个Layer。`EnderEyesLayer`继承了`EyesLayer`，可以以特殊的`RenderType`渲染眼睛，同时用`EyesLayer`添加的眼睛不会因为实体隐身而消失。  

想知道这是为什么吗？先来看`EyesLayer`的`render`方法。
```java
@Override
public void render(PoseStack stack, MultiBufferSource source, int packedLight, T entity, float limbSwing, float limbSwingAmount, float partialTicks, float ageInTicks, float netHeadYaw, float headPitch) {
    VertexConsumer consumer = source.getBuffer(renderType());
    getParentModel().renderToBuffer(stack, consumer, 15728640, OverlayTexture.NO_OVERLAY, 1.0F, 1.0F, 1.0F, 1.0F);
}
```
而游戏中狼却不会在隐身时显示项圈，对比一下`WolfCollarLayer`的片段。
```java
@Override
public void render(PoseStack stack, MultiBufferSource source, int packedLight, Wolf wolf, float limbSwing, float limbSwingAmount, float partialTicks, float ageInTicks, float netHeadYaw, float headPitch) {
    if (wolf.isTame() && !wolf.isInvisible()) {
        float[] colors = wolf.getCollarColor().getTextureDiffuseColors();
        renderColoredCutoutModel(getParentModel(), WOLF_COLLAR_LOCATION, stack, source, packedLight, wolf, colors[0], colors[1], colors[2]);
    }
}
```
真相就大白了！`WolfCollarLayer`中**判断了狼是否隐身**，如果隐身就不渲染项圈，所以隐身的狼的项圈是看不见的。`EyesLayer`中**没有判断末影人是否隐身**，因此隐身的末影人眼睛可见。

`CarriedBlockLayer`则用`rotate`, `translate`, `scale`等操作渲染了末影人手上拿着的方块，并将它移动到正确的位置。这个方法如下：
```java
@Override
public void render(PoseStack stack, MultiBufferSource source, int packedLight, Wolf wolf, float limbSwing, float limbSwingAmount, float partialTicks, float ageInTicks, float netHeadYaw, float headPitch) {
    BlockState carriedBlock = enderman.getCarriedBlock();
    if (carriedBlock != null) {
//  这些方法的含义可以参考一些渲染方面的教程
        stack.pushPose();
        stack.translate(0.0F, 0.6875F, -0.75F);
        stack.mulPose(Axis.XP.rotationDegrees(20.0F));
        stack.mulPose(Axis.YP.rotationDegrees(45.0F));
        stack.translate(0.25F, 0.1875F, 0.25F);
        stack.scale(-0.5F, -0.5F, 0.5F);
        stack.mulPose(Axis.YP.rotationDegrees(90.0F));
        blockRenderer.renderSingleBlock(carriedBlock, stack, bufferSource, packedLight, OverlayTexture.NO_OVERLAY);
        stack.popPose();
    }
}
```
应该不难理解其中的内容，如果对`PoseStack`（旧称`MatrixStack`）的操作不太熟悉，可以去阅读相关的渲染方面的教程，此处不做过多赘述。

接下来就没有什么难点了。然后来看`render`方法。
```java
@Override
public void render(EnderMan enderman, float rotation, float delta, PoseStack stack, MultiBufferSource bufferSource, int packedLight) {
    BlockState carriedBlock = enderman.getCarriedBlock();
    EndermanModel<EnderMan> model = getModel();
    model.carrying = carriedBlock != null;
    model.creepy = enderman.isCreepy();
    super.render(enderman, rotation, delta, stack, bufferSource, packedLight);
}
```
render方法重写的内容很简单，只是根据末影人的状态更新了模型的`carrying`和`creepy`两个boolean的值。  

最后要讲的就是`getRenderOffset`方法。
```java
@Override
public Vec3 getRenderOffset(EnderMan enderman, float delta) {
    if (enderman.isCreepy()) {
        return new Vec3(random.nextGaussian() * 0.02D, 0.0D, random.nextGaussian() * 0.02D);
    } else {
        return super.getRenderOffset(enderman, delta);
    }
}
```
`getRenderOffset`方法的返回值被用来“translate”渲染实体用的`PoseStack`。因为末影人愤怒时身体会抖动，所以要重写此方法。  

本节的内容就到此为止了，本章的内容也快要结束了。下一节我们再来讲一个自定义的近战怪物的实例~
