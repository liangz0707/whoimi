# UE4基础源码Pipeline部分

这部分主要记录源代码中和Pipeline相关的部分。

# FSceneRenderer

 * Used as the scope for scene rendering functions.
 * It is initialized in the game thread by FSceneViewFamily::BeginRender, and then passed to the rendering thread.
 * The rendering thread calls Render(), and deletes the scene renderer when it returns.

有两个主要的继承类：FMobileSceneRenderer和FDeferredShadingSceneRenderer。手机不支持Deferred（MRT），所以都是forward渲染。

保存了要渲染的内容：

```c
//要渲染的场景。
/** The scene being rendered. */
FScene* Scene;
// 一个渲染目标RenderTarget，这个可以有多个渲染View。
/** The view family being rendered.  This references the Views array. */
FSceneViewFamily ViewFamily;

// 所有Mesh？
FMeshElementCollector MeshCollector;

// 光
/** Information about the visible lights. */
TArray<FVisibleLightInfo,SceneRenderingAllocator> VisibleLightInfos;

// 和阴影相关的内容
/** Array of dispatched parallel shadow depth passes. */
TArray<FParallelMeshDrawCommandPass*, SceneRenderingAllocator> DispatchedShadowDepthPasses;

FSortedShadowMaps SortedShadowsForShadowDepthPass;
```

渲染相关的方法：

```c
// FSceneRenderer interface
virtual void Render(FRHICommandListImmediate& RHICmdList) = 0;
```



# FMobileSceneRenderer

* Renderer that implements simple forward shading and associated features.


主要包含了各种渲染方法：

```c

void RenderMobileBasePass(FRHICommandListImmediate& RHICmdList, const TArrayView<const FViewInfo*> PassViews);

void RenderMobileEditorPrimitives(FRHICommandList& RHICmdList, const FViewInfo& View, const FMeshPassProcessorRenderState& DrawRenderState);

void RenderModulatedShadowProjections(FRHICommandListImmediate& RHICmdList);

void RenderOcclusion(FRHICommandListImmediate& RHICmdList);
	
bool RequiresTranslucencyPass(FRHICommandListImmediate& RHICmdList, const FViewInfo& View) const;

void RenderDecals(FRHICommandListImmediate& RHICmdList);

void RenderTranslucency(FRHICommandListImmediate& RHICmdList, const TArrayView<const FViewInfo*> PassViews, bool bRenderToSceneColor);


```

具体的实现在源文件MobileShadingRenderer.cpp当中。

## MobileShadingRenderer的Renderer过程

这里详细标注下面方法的渲染过程：

```c  
void FMobileSceneRenderer::Render(FRHICommandListImmediate& RHICmdList)
```

```c
RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_SceneStart));
// 计算各个视口大小和位置。
PrepareViewRectsForRendering();

// 遮挡剔除？
WaitOcclusionTests(RHICmdList);
FRHICommandListExecutor::GetImmediateCommandList().PollOcclusionQueries();
RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);

// FeatureLevel ：ES  SM4 SM5
const ERHIFeatureLevel::Type ViewFeatureLevel = ViewFamily.GetFeatureLevel();

// 初始化渲染目标
// Allocate the maximum scene render target space for the current view family.
SceneContext.Allocate(RHICmdList, this);


```



### PrepareViewRectsForRendering();

```c
void FSceneRenderer::PrepareViewRectsForRendering()
{
    // 判断是否在渲染线程
    check(IsInRenderingThread());
    ...
        
    // 主要就是计算FamilySize
    ComputeFamilySize();
    
    
    // Find the visible primitives.
	InitViews(RHICmdList);
    
    
    // 开始渲染
    RHICmdList.BeginRenderPass(SceneColorRenderPassInfo, TEXT("SceneColorRendering"));
	
    // 渲染BasePass （Opaque Forward）
    // Opaque and masked
	RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_Opaque));
	RenderMobileBasePass(RHICmdList, ViewList);
	RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);
    
}
```



### FSceneRenderTargets

渲染目标的构造

```c
/**
 * Encapsulates the render targets used for scene rendering.
 */
class RENDERER_API FSceneRenderTargets : public FRenderResource
{

}
```

### FRHICommandListImmediate

### RenderMobileBasePass

方法内容如下：

```c

// 这个循环主要遍历所有的View
for (int32 ViewIndex = 0; ViewIndex < PassViews.Num(); ViewIndex++)
{
	// 获取View的信息
    const FViewInfo& View = *PassViews[ViewIndex];
    if (!View.ShouldRenderView())
    {
        continue;
    }
	
    // 更新Buffer
    if (Scene->UniformBuffers.UpdateViewUniformBuffer(View))
    {
        UpdateOpaqueBasePassUniformBuffer(RHICmdList, View);
        UpdateDirectionalLightUniformBuffers(RHICmdList, View);
    }

    // 设置视口
    RHICmdList.SetViewport(View.ViewRect.Min.X, View.ViewRect.Min.Y, 0, View.ViewRect.Max.X, View.ViewRect.Max.Y, 1);
   								View.ParallelMeshDrawCommandPasses[EMeshPass::BasePass].DispatchDraw(nullptr, RHICmdList);

    // editor primitives
    {
        FMeshPassProcessorRenderState DrawRenderState(View, Scene->UniformBuffers.MobileOpaqueBasePassUniformBuffer);
        DrawRenderState.SetBlendState(TStaticBlendStateWriteMask<CW_RGBA>::GetRHI());
        DrawRenderState.SetDepthStencilAccess(Scene->DefaultBasePassDepthStencilAccess);
        DrawRenderState.SetDepthStencilState(TStaticDepthStencilState<true, CF_DepthNearOrEqual>::GetRHI());
        RenderMobileEditorPrimitives(RHICmdList, View, DrawRenderState);
    }
}
```



# FDeferredShadingSceneRenderer

* Scene renderer that implements a deferred shading pipeline and associated features.