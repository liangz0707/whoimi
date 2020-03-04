# UE4渲染管线代码初探

本篇笔记主要会以**DeferredShadingRenderer.cpp**文件的**Render()**方法为入口，探索UE4场景的渲染过程。

由于UE4的场景渲染过程复杂，这里主要专注于梳理流程，对于各个调用不会挖掘太深。不过会在之后去详细研究每一个模块。目前的建议是：对于各个模块（场景管理、灯光管理、阴影策略等）会以算法的角度去学习，而不是通过阅读代码。

对于每一个内容会直接截取原始代码进行注释说明。

## 总述

我们现在研究的是UE4渲染场景的流程，包含在下面的方法 中：

```c
void FDeferredShadingSceneRenderer::Render(FRHICommandListImmediate& RHICmdList)
```

该方法只有600行左右的核心代码，都会尝试做梳理，而不梳理的部分会引出其他章节或给出相关算法。

下面是相当简单概略的介绍。如需了解详情，请通读相关代码或输出的“profilegpu”日志。

### 硬件渲染接口RHI

RHI 是平台特定的图形 API 之上的一个薄层。UE4 中的 RHI 抽象层尽可能低，这样大多数功能都能以与平台无关的代码写成，从而能够在支持所需功能层级的任何平台上运行。

而我们代码中的资源最终会归结到RHI类型的资源，可以再[官方RHI索引页面](https://docs.unrealengine.com/en-US/API/Runtime/RHI/index.html)搜索到。

### 资源

在 UE4 中，渲染器所见的场景由基本组件和 FScene 中存储的多种其他结构的列表定义。将维护一个基元的八叉树，用于加速空间查询。

### 总体渲染顺序

这部分内容以[官方文档](https://docs.unrealengine.com/zh-CN/Programming/Rendering/Overview/index.html)中的总体渲染顺序为框架进行对照代码的细化、解释说明。

### 渲染状态默认值

由于渲染状态数量众多，要在每次绘制之前对它们全部设置一遍是不现实的。为此，UE4 具有隐性设置的一组状态，它们被认为是设置为了默认值（因此在变更后必须恢复为默认值），另外还有一组少得多的状态需要显性设置。没有隐性默认值的状态有：

- RHISetRenderTargets
- RHISetBoundShaderState
- RHISetDepthState
- RHISetBlendState
- RHISetRasterizerState
- 由 RHISetBoundShaderState 设置的着色器的任何依赖性

## SceneContext资源申请

按需要重新分配全局场景渲染目标，使其对当前视图足够大。

```c
//可以看到需要从RHICmdList中申请我们需要的RenderTarget,这里是创建一个渲染目标对象
FSceneRenderTargets& SceneContext = FSceneRenderTargets::Get(RHICmdList); 
// 转化成可写
GRenderTargetPool.TransitionTargetsWritable(RHICmdList);
// 情况RenderTarget内容，确认buffer格式正确
SceneContext.ReleaseSceneColor();
// 进行相关资源、线程状态的检查
...
// SCOPED_DRAW_EVENT用于标注GPU的渲染代码块，用于调试和分辨渲染阶段。
/*
官方说明：
It is also useful to do a 'profilegpu' command and look through the draw events. You can then do a Find in Files in Visual Studio on the draw event name to find the corresponding C++ implementation.
*/
SCOPED_DRAW_EVENT(RHICmdList, Scene);
// 可以用引用这些统计数据名称的SCOPED_GPU_STAT宏检测渲染线程上的代码块
SCOPED_GPU_STAT(RHICmdList, Stat_GPU_Unaccounted);
// 继续资源申请
{
    SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_Render_Init);
    // Initialize global system textures (pass-through if already initialized).
    GSystemTextures.InitializeTextures(RHICmdList, FeatureLevel);
    // Allocate the maximum scene render target space for the current view family.
    SceneContext.Allocate(RHICmdList, ViewFamily);
}
SceneContext.AllocDummyGBufferTargets(RHICmdList);
```

## InitViews可见物体筛选

通过多种剔除方法为视图初始化基元可见性，设立此帧可见的动态阴影、按需要交叉阴影视锥与世界场景（对整个场景的阴影或预阴影）。

代码如下：

```c
// Find the visible primitives.
bool bDoInitViewAftersPrepass = InitViews(RHICmdList, ILCTaskData, SortEvents);
```

这部分过程包括了遮挡剔除、透明物体排序等CPU端的处理，准备各种资源。

下面探索InitView代码内部：

### PreVisibilityFrameSetup

### ComputeViewVisibility

### CreateIndirectCapsuleShadows

### PostVisibilityFrameSetup



## PrePass / Depth only pass

RenderPrePass / FDepthDrawingPolicy。渲染遮挡物，对景深缓冲区仅输出景深。该通道可以在多种模式下工作：禁用、仅遮蔽，或完全景深，具体取决于活动状态的功能的需要。该通道通常的用途是初始化 Hierarchical Z 以降低 Base 通道的着色消耗（Base 通道的像素着色器消耗非常大）。



## Base pass

RenderBasePass / TBasePassDrawingPolicy。渲染不透明和遮盖的材质，向 GBuffer 输出材质属性。光照图贡献和天空光照也会在此计算并加入场景颜色。



## Issue Occlusion Queries / BeginOcclusionTests

提出将用于下一帧的 InitViews 的延迟遮蔽查询。这会通过渲染所查询物体周围的相邻的框、有时还会将相邻的框组合在一起以减少绘制调用来完成。





## Lighting

阴影图将对各个光照渲染，光照贡献会累加到场景颜色，并使用标准延迟和平铺延迟着色。光照也会在透明光照体积中累加。



## Fog

雾和大气在延迟通道中对不透明表面进行逐个像素计算。



## Translucency

透明度累加到屏外渲染目标，在其中它应用了逐个顶点的雾化，因而可以整合到场景中。光照透明度在一个通道中计算最终光照以正确融合。



## Post Processing

多种后期处理效果均通过 GBuffers 应用。透明度将合成到场景中。







