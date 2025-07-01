# 唤魔者的渲染

本节将会对唤魔者的渲染器（`EvokerRenderer`）以及唤魔者使用法术时粒子效果的渲染进行分析。

`EvokerRenderer`类的内容并不复杂，仅在`IllagerRenderer`的基础上添加了一层`ItemInHandLayer`（这里使用了匿名内部类，且重写了`render`方法，以确保只有在唤魔者施法时才会在手上渲染物品的模型）。

```java
@OnlyIn(Dist.CLIENT)
public class EvokerRenderer<T extends SpellcasterIllager> extends IllagerRenderer<T> {
    private static final ResourceLocation EVOKER_ILLAGER = new ResourceLocation("textures/entity/illager/evoker.png");
    public EvokerRenderer(EntityRendererProvider.Context context) {
        // 唤魔者没有单独的模型，而是直接使用了IllagerModel
        super(context, new IllagerModel<>(context.bakeLayer(ModelLayers.EVOKER)), 0.5F);
        addLayer(new ItemInHandLayer<>(this, context.getItemInHandRenderer()) {
            @Override
            public void render(PoseStack poseStack, MultiBufferSource source, int packedLight, T evoker, float limbSwing, float limbSwingAmount, float partialTicks, float ageInTicks, float netHeadYaw, float headPitch) {
                if (evoker.isCastingSpell()) {
                    super.render(poseStack, source, packedLight, evoker, limbSwing, limbSwingAmount, partialTicks, ageInTicks, netHeadYaw, headPitch);
                }
            }
        });
    }

    @Override
    public ResourceLocation getTextureLocation(T evoker) {
        return EVOKER_ILLAGER;
    }
}
```

唤魔者使用法术时粒子效果的渲染逻辑位于`SpellcasterIllager`中的`tick`方法中。
```java
@Override
public void tick() {
    super.tick();
    if (level().isClientSide && isCastingSpell()) {
        IllagerSpell spell = getCurrentSpell();
        double r = spell.spellColor[0];
        double g = spell.spellColor[1];
        double b = spell.spellColor[2];
        // 这里利用余弦函数的特点，使len随时间周期性变化（len变量的值正比于粒子位置到唤魔者位置的水平距离）
        float len = yBodyRot * ((float) Math.PI / 180F) + Mth.cos((float) tickCount * 0.6662F) * 0.25F;
        // 与上面不同，这里使用三角函数则是用于获取len在x、z方向的分量
        float dx = Mth.cos(len);
        float dz = Mth.sin(len);
        // 注意当ParticleType为ENTITY_EFFECT时，最后三个参数不为粒子运动的方向向量，而是粒子效果的颜色
        // 渲染左手上的粒子效果
        level().addParticle(ParticleTypes.ENTITY_EFFECT, getX() + (double) dx * 0.6D, getY() + 1.8D, getZ() + (double) dz * 0.6D, r, g, b);
        // 渲染右手上的粒子效果
        level().addParticle(ParticleTypes.ENTITY_EFFECT, getX() - (double) dx * 0.6D, getY() + 1.8D, getZ() - (double) dz * 0.6D, r, g, b);
    }
}
```

本节的内容，以及整个1.2.3.2的内容就到此为止了哦，接下来我们来看一个稍复杂的有关远程攻击的怪物的实例。