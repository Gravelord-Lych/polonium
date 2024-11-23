# 女巫的渲染细节

我们可以通过观察女巫发现如下两个问题：
1. 女巫的鼻子会抖动，这具体是如何实现的呢？
2. 当女巫手持药水瓶准备喝药水时，可以发现女巫手中的药水瓶进行了一定的旋转与偏移，这又是怎么实现的呢？  

这部分我们就来探讨一下这两个问题。  

先看`WitchModel`，注册的部分就省略不看了。
```java
@Override
public void setupAnim(T witch, float limbSwing, float limbSwingAmount, float ageInTicks, float netHeadYaw, float headPitch) {
    super.setupAnim(witch, limbSwing, limbSwingAmount, ageInTicks, netHeadYaw, headPitch);
    nose.setPos(0.0F, -2.0F, 0.0F);
    // 这个值决定了女巫鼻子晃动的频率。它的大小实际上与女巫的id有关，但因为实体的id不容易确定，所以可以理解为随机的
    float shakeFrequency = 0.01F * (float) (witch.getId() % 10);
    nose.xRot = Mth.sin((float) witch.tickCount * shakeFrequency) * 4.5F * ((float) Math.PI / 180F);
    nose.yRot = 0.0F;
    nose.zRot = Mth.cos((float) witch.tickCount * shakeFrequency) * 2.5F * ((float) Math.PI / 180F);
    // 当女巫手持物品时，鼻子应当改变位置
    if (holdingItem) {
        nose.setPos(0.0F, 1.0F, -1.5F);
        nose.xRot = -0.9F;
    }
}

// 这两个方法在女巫的渲染中会用到
public ModelPart getNose() {
    return nose;
}

public void setHoldingItem(boolean holdingItem) {
    this.holdingItem = holdingItem;
}
```
这里在`setupAnim`中为女巫的鼻子进行了一定的变换，从而实现了鼻子的抖动效果。

女巫的渲染类`WitchRenderer`似乎平淡无奇，但是我们可以发现一个重要的`Layer`（`WitchItemLayer`）。  

`WitchRenderer`（节选）：
```java
public WitchRenderer(EntityRendererProvider.Context context) {
    super(context, new WitchModel<>(context.bakeLayer(ModelLayers.WITCH)), 0.5F);
    addLayer(new WitchItemLayer<>(this, context.getItemInHandRenderer()));
}

@Override
public void render(Witch witch, float yRot, float partialTicks, PoseStack stack, MultiBufferSource source, int packedLight) {
    model.setHoldingItem(!witch.getMainHandItem().isEmpty());
    super.render(witch, yRot, partialTicks, stack, source, packedLight);
}
```

`WitchItemLayer`：
```java
@OnlyIn(Dist.CLIENT)
public class WitchItemLayer<T extends LivingEntity> extends CrossedArmsItemLayer<T, WitchModel<T>> {
    public WitchItemLayer(RenderLayerParent<T, WitchModel<T>> parent, ItemInHandRenderer itemInHandRenderer) {
        super(parent, itemInHandRenderer);
    }

    @Override
    public void render(PoseStack stack, MultiBufferSource source, int packedLight, T witch, float limbSwing, float limbSwingAmount, float partialTicks, float ageInTicks, float netHeadYaw, float headPitch) {
        ItemStack itemstack = witch.getMainHandItem();
        stack.pushPose();
        if (itemstack.is(Items.POTION)) {
            getParentModel().getHead().translateAndRotate(stack);
            getParentModel().getNose().translateAndRotate(stack);
            stack.translate(0.0625F, 0.25F, 0.0F);
            stack.mulPose(Axis.ZP.rotationDegrees(180.0F));
            stack.mulPose(Axis.XP.rotationDegrees(140.0F));
            stack.mulPose(Axis.ZP.rotationDegrees(10.0F));
            stack.translate(0.0F, -0.4F, 0.4F);
        }
        super.render(stack, source, packedLight, witch, limbSwing, limbSwingAmount, partialTicks, ageInTicks, netHeadYaw, headPitch);
        stack.popPose();
    }
}
```

看一下`WitchItemLayer`中的`render`方法。
```java
@Override
public void render(PoseStack stack, MultiBufferSource source, int packedLight, T witch, float limbSwing, float limbSwingAmount, float partialTicks, float ageInTicks, float netHeadYaw, float headPitch) {
    ItemStack mainHandItem = witch.getMainHandItem();
    stack.pushPose();
    if (mainHandItem.is(Items.POTION)) {
        // 更新头、鼻子的位置与旋转角度
        getParentModel().getHead().translateAndRotate(stack);
        getParentModel().getNose().translateAndRotate(stack);
        // 对PoseStack进行translate和mulPose，以确保将来会在正确的位置渲染物品 
        stack.translate(0.0625F, 0.25F, 0.0F);
        stack.mulPose(Axis.ZP.rotationDegrees(180.0F));
        stack.mulPose(Axis.XP.rotationDegrees(140.0F));
        stack.mulPose(Axis.ZP.rotationDegrees(10.0F));
        stack.translate(0.0F, -0.4F, 0.4F);
    }
    super.render(stack, source, packedLight, witch, limbSwing, limbSwingAmount, partialTicks, ageInTicks, netHeadYaw, headPitch);
    stack.popPose();
}
```
但是仿佛找不到一行与渲染物品有关的代码……别急，我们看看它的父类`CrossedArmsItemLayer`的`render`方法。
```java
public void render(PoseStack stack, MultiBufferSource source, int packedLight, T entity, float limbSwing, float limbSwingAmount, float partialTicks, float ageInTicks, float netHeadYaw, float headPitch) {
    stack.pushPose();
    stack.translate(0.0F, 0.4F, -0.4F);
    stack.mulPose(Axis.XP.rotationDegrees(180.0F));
    ItemStack mainHandItem = entity.getItemBySlot(EquipmentSlot.MAINHAND);
    itemInHandRenderer.renderItem(entity, mainHandItem, ItemDisplayContext.GROUND, false, stack, source, packedLight);
    stack.popPose();
}
```
可以发现物品的渲染是在父类完成的，而`WitchItemLayer`则完成了对药水位置的重新确定。  

女巫就到此为止吧……下一节我们将讲解骷髅的具体实现，并作为1.2.2的最后一个原版实例讲解。骷髅的综合性较强，可能需要联系1.2.1所学的内容进行理解。