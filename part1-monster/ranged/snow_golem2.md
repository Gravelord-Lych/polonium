#雪傀儡的模型与渲染

与前面说过的僵尸与末影人都不同，雪傀儡的模型类`SnowGolemModel`继承了`HierarchicalModel`而非`HumanoidModel`。`HierarchicalModel`类并未提供相应的`ModelPart`，因此需要单独声明这些成员变量。**在创建非人形实体（生物和具有较复杂模型的弹射物）的模型时，一般都需要继承这个类来实现模型的自定义。**  
```java
@OnlyIn(Dist.CLIENT)
public class SnowGolemModel<T extends Entity> extends HierarchicalModel<T> {
    private final ModelPart root;
    private final ModelPart upperBody;
    private final ModelPart head;
    private final ModelPart leftArm;
    private final ModelPart rightArm;

    public SnowGolemModel(ModelPart part) {
        this.root = part;
        this.head = part.getChild("head");
        this.leftArm = part.getChild("left_arm");
        this.rightArm = part.getChild("right_arm");
        this.upperBody = part.getChild("upper_body");
    }

//  这个静态方法用于声明雪傀儡的模型，前面讲末影人的模型时讲过，此处也大同小异，因此不做赘述
    public static LayerDefinition createBodyLayer() {
        MeshDefinition meshDef = new MeshDefinition();
        PartDefinition partDef = meshDef.getRoot();
        CubeDeformation cubeDef = new CubeDeformation(-0.5F);
        partDef.addOrReplaceChild("head", CubeListBuilder.create().texOffs(0, 0).addBox(-4.0F, -8.0F, -4.0F, 8.0F, 8.0F, 8.0F, cubeDef), PartPose.offset(0.0F, 4.0F, 0.0F));
        CubeListBuilder builder = CubeListBuilder.create().texOffs(32, 0).addBox(-1.0F, 0.0F, -1.0F, 12.0F, 2.0F, 2.0F, cubeDef);
        partDef.addOrReplaceChild("left_arm", builder, PartPose.offsetAndRotation(5.0F, 6.0F, 1.0F, 0.0F, 0.0F, 1.0F));
        partDef.addOrReplaceChild("right_arm", builder, PartPose.offsetAndRotation(-5.0F, 6.0F, -1.0F, 0.0F, (float)Math.PI, -1.0F));
        partDef.addOrReplaceChild("upper_body", CubeListBuilder.create().texOffs(0, 16).addBox(-5.0F, -10.0F, -5.0F, 10.0F, 10.0F, 10.0F, cubeDef), PartPose.offset(0.0F, 13.0F, 0.0F));
        partDef.addOrReplaceChild("lower_body", CubeListBuilder.create().texOffs(0, 36).addBox(-6.0F, -12.0F, -6.0F, 12.0F, 12.0F, 12.0F, cubeDef), PartPose.offset(0.0F, 24.0F, 0.0F));
        return LayerDefinition.create(meshDef, 64, 64);
    }
}
```

`HierarchicalModel`没有实现抽象方法`setupAnim`，要在子类中去实现该抽象方法。
```java
@Override
public void setupAnim(T entity, float limbSwing, float limbSwingAmount, float ageInTicks, float netHeadYaw, float headPitch) {
    head.yRot = netHeadYaw * ((float) Math.PI / 180F);
    head.xRot = headPitch * ((float) Math.PI / 180F);
    upperBody.yRot = netHeadYaw * ((float) Math.PI / 180F) * 0.25F;
    float sinRot = Mth.sin(upperBody.yRot);
    float cosRot = Mth.cos(upperBody.yRot);
    leftArm.yRot = upperBody.yRot;
    rightArm.yRot = upperBody.yRot + (float) Math.PI;
    leftArm.x = cosRot * 5.0F;
    leftArm.z = -sinRot * 5.0F;
    rightArm.x = -cosRot * 5.0F;
    rightArm.z = sinRot * 5.0F;
}
```
此处`setupAnim`同步了头部、身体与手部的转动，注意**在`ModelPart`中带rot的float类型变量（`xRot`，`yRot`，`zRot`）存储的都是弧度，而在`Entity`中带rot的float类型变量（`yRot`，`xRot`等）存储的却是角度**，所以开发中涉及到旋转时要**特别注意角度与弧度的转换**，防止两者混淆导致实体的旋转变得怪异。  

最后还有两个getter要实现。
```java
@Override
public ModelPart root() {
    return this.root;
}

@Override
public ModelPart getHead() {
    return this.head;
}
```

接下来是渲染类。
```java
@OnlyIn(Dist.CLIENT)
public class SnowGolemRenderer extends MobRenderer<SnowGolem, SnowGolemModel<SnowGolem>> {
    private static final ResourceLocation SNOW_GOLEM_LOCATION = new ResourceLocation("textures/entity/snow_golem.png");

    public SnowGolemRenderer(EntityRendererProvider.Context context) {
        super(context, new SnowGolemModel<>(context.bakeLayer(ModelLayers.SNOW_GOLEM)), 0.5F);
        addLayer(new SnowGolemHeadLayer(this, context.getBlockRenderDispatcher(), context.getItemRenderer()));
    }

    @Override
    public ResourceLocation getTextureLocation(SnowGolem golem) {
        return SNOW_GOLEM_LOCATION;
    }
}
```
渲染类很短，似乎没什么可说的……等等，第7行是干什么的呢？这得从`SnowGolemHeadLayer`说起。  
```java
@OnlyIn(Dist.CLIENT)
public class SnowGolemHeadLayer extends RenderLayer<SnowGolem, SnowGolemModel<SnowGolem>> {
    private final BlockRenderDispatcher blockRenderer;
    private final ItemRenderer itemRenderer;

    public SnowGolemHeadLayer(RenderLayerParent<SnowGolem, SnowGolemModel<SnowGolem>> parent, BlockRenderDispatcher blockRenderer, ItemRenderer itemRenderer) {
        super(parent);
        this.blockRenderer = blockRenderer;
        this.itemRenderer = itemRenderer;
    }

    @Override
    public void render(PoseStack poseStack, MultiBufferSource source, int packedLight, SnowGolem golem, float limbSwing, float limbSwingAmount, float partialTicks, float ageInTicks, float netHeadYaw, float headPitch) {
//  ......
    }
}
```
我们来看`render`方法。
```java
@Override
public void render(PoseStack poseStack, MultiBufferSource source, int packedLight, SnowGolem golem, float limbSwing, float limbSwingAmount, float partialTicks, float ageInTicks, float netHeadYaw, float headPitch) {
    if (golem.hasPumpkin()) {
        boolean invisibleButGlowing = Minecraft.getInstance().shouldEntityAppearGlowing(golem) && golem.isInvisible();
        if (!golem.isInvisible() || invisibleButGlowing) {
            poseStack.pushPose();
            getParentModel().getHead().translateAndRotate(poseStack);
//          初步移动到指定的位置
            poseStack.translate(0.0F, -0.34375F, 0.0F);
//          旋转到适合的角度
            poseStack.mulPose(Axis.YP.rotationDegrees(180.0F));
//          缩放到适当的大小
            poseStack.scale(0.625F, -0.625F, -0.625F)；
            ItemStack itemStack = new ItemStack(Blocks.CARVED_PUMPKIN);
            if (invisibleButGlowing) {
                BlockState pumpkinHead = Blocks.CARVED_PUMPKIN.defaultBlockState();
                BakedModel pumpkinHeadModel = blockRenderer.getBlockModel(pumpkinHead);
                int overlayCoords = LivingEntityRenderer.getOverlayCoords(golem, 0.0F);
                poseStack.translate(-0.5F, -0.5F, -0.5F);
                blockRenderer.getModelRenderer().renderModel(poseStack.last(), source.getBuffer(RenderType.outline(TextureAtlas.LOCATION_BLOCKS)), pumpkinHead, pumpkinHeadModel, 0.0F, 0.0F, 0.0F, packedLight, overlayCoords);
            } else {
                itemRenderer.renderStatic(golem, itemStack, ItemDisplayContext.HEAD, false, poseStack, source, golem.level(), packedLight, LivingEntityRenderer.getOverlayCoords(golem, 0.0F), golem.getId());
            }
            poseStack.popPose();
        }
    }
}
```
这里我们在雪傀儡可见或发光时渲染了它的南瓜头（不可见但发光时提供了南瓜头的轮廓），同时通过传入`overlayCoords`使南瓜头能在雪傀儡受伤时与雪傀儡一起“变红”。  

本节内容到这里就结束啦，接下来是实战吗？是的！那么之前说过还有一类远程攻击方式，怎么写这类生物的代码呢？别急，使用这类方式的生物多半没继承`RangedAttackMob`且更为复杂，属于1.2.3章节讨论的范畴，我们等到1.2.3再讲~
