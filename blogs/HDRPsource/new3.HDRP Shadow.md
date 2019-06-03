准备阴影Atlas：

```c
// 在PrepareLightsForGPU当中分配阴影,PrepareLightsForGPU是用来CPU准备光照数据的

// 给阴影分配渲染请求，设置摄像机参数，阴影分辨率。
additionalData.ReserveShadows(camera, m_ShadowManager, m_ShadowInitParameters, cullResults, m_FrameSettings, lightIndex);

...
// Now that all the lights have requested a shadow resolution, we can layout them in the atlas
// And if needed rescale the whole atlas
m_ShadowManager.LayoutShadowMaps(debugDisplaySettings.data.lightingDebugSettings);
```

当我设置多个方向光阴影的时候会报错，分配阴影贴图布局的方法：

```c
public void LayoutShadowMaps(LightingDebugSettings lightingDebugSettings)
{
    m_Atlas.UpdateDebugSettings(lightingDebugSettings);

    if (m_CascadeAtlas != null)
        m_CascadeAtlas.UpdateDebugSettings(lightingDebugSettings);

    m_AreaLightShadowAtlas.UpdateDebugSettings(lightingDebugSettings);

    if (lightingDebugSettings.shadowResolutionScaleFactor != 1.0f)
    {
        foreach (var shadowResolutionRequest in m_ShadowResolutionRequests)
        {
            // We don't rescale the directional shadows with the global shadow scale factor
            // because there is no dynamic atlas rescale when it overflow.
            if (shadowResolutionRequest.shadowMapType != ShadowMapType.CascadedDirectional)
                shadowResolutionRequest.resolution *= lightingDebugSettings.shadowResolutionScaleFactor;
        }
    }

    // Assign a position to all the shadows in the atlas, and scale shadows if needed
    if (m_CascadeAtlas != null && !m_CascadeAtlas.Layout(false))
        Debug.LogError("Cascade Shadow atlasing has failed, only one directional light can cast shadows at a time");
    m_Atlas.Layout();
    m_AreaLightShadowAtlas.Layout();
}
```

渲染阴影的位置是在HDShadowAtlas类当中：

```c
// HDShadowAtlas类
// 所有需要渲染阴影的灯光都会组织成一个HDShadowAtlas类。
public void RenderShadows(ScriptableRenderContext renderContext, CommandBuffer cmd, ShadowDrawingSettings dss)
{
    // m_ShadowRequests是渲染阴影需要的所有的参数。
    if (m_ShadowRequests.Count == 0)
        return;
	
    // shadow mask 纹理id
    cmd.SetRenderTarget(identifier);
    // shadow 大小 
    cmd.SetGlobalVector(m_AtlasSizeShaderID, new Vector4(width, height, 1.0f / width, 1.0f / height));

    // debug设置
    if (m_LightingDebugSettings.clearShadowAtlas)
        CoreUtils.DrawFullScreen(cmd, m_ClearMaterial, null, 0);
	
    // 渲染所以阴影请求，直线光渲染次数为级联阴影数。
    foreach (var shadowRequest in m_ShadowRequests)
    {
        // 设置渲染区域。
        cmd.SetViewport(shadowRequest.atlasViewport);
		// 根据阴影设置是否启用zClip
        /*
        zClip的设置：
        shadowRequest.zClip = (legacyLight.type != LightType.Directional);
        为什么关闭直线光的ZClip
        */
        cmd.SetGlobalFloat(HDShaderIDs._ZClip, shadowRequest.zClip ? 1.0f : 0.0f);
        if (!m_LightingDebugSettings.clearShadowAtlas)
        {
            CoreUtils.DrawFullScreen(cmd, m_ClearMaterial, null, 0);
        }
		
        // 读取灯光信息和剪裁信息
        dss.lightIndex = shadowRequest.lightIndex;
        dss.splitData = shadowRequest.splitData;

        // 设置摄像机参数
        // Setup matrices for shadow rendering:
        Matrix4x4 viewProjection = shadowRequest.deviceProjectionYFlip * shadowRequest.view;
        cmd.SetGlobalMatrix(HDShaderIDs._ViewMatrix, shadowRequest.view);
        cmd.SetGlobalMatrix(HDShaderIDs._InvViewMatrix, shadowRequest.view.inverse);
        cmd.SetGlobalMatrix(HDShaderIDs._ProjMatrix, shadowRequest.deviceProjectionYFlip);
        cmd.SetGlobalMatrix(HDShaderIDs._InvProjMatrix, shadowRequest.deviceProjectionYFlip.inverse);
        cmd.SetGlobalMatrix(HDShaderIDs._ViewProjMatrix, viewProjection);
        cmd.SetGlobalMatrix(HDShaderIDs._InvViewProjMatrix, viewProjection.inverse);
        cmd.SetGlobalVectorArray(HDShaderIDs._ShadowClipPlanes, shadowRequest.frustumPlanes);

        // TODO: remove this execute when DrawShadows will use a CommandBuffer
        // 使用Command来绘制阴影，目前还没有实现？？
        renderContext.ExecuteCommandBuffer(cmd);
        cmd.Clear();

        renderContext.DrawShadows(ref dss);
    }
	
    // 设置Clip启用
    cmd.SetGlobalFloat(HDShaderIDs._ZClip, 1.0f);   // Re-enable zclip globally
}
```



HDShadowResolutionRequest表示一个灯光需要渲染阴影的次数。目前通过灯光类型，和shadow级联数决定，下面是所有相关的方法：

```c
// =================HDAdditionalLightData:ReserveShadows====
int count = HDAdditionalLightData:GetShadowRequestCount();
for (int index = 0; index < count; index++)
    m_ShadowRequestIndices[index] = shadowManager.ReserveShadowResolutions(viewportSize, shadowMapType);

// =================HDAdditionalLightData:GetShadowRequestCount====
int GetShadowRequestCount()
{
    return (legacyLight.type == LightType.Point 
            && lightTypeExtent == LightTypeExtent.Punctual) ? 
        	6 : 
    		// 如果是直线光，则渲染次数等于级联阴影数。
            (legacyLight.type == LightType.Directional) ? 				    m_ShadowSettings.cascadeShadowSplitCount.value : 1;
}

//==================HDShadowManager:ReserveShadowResolutions ======
switch (shadowMapType)
{
    case ShadowMapType.PunctualAtlas:
      m_Atlas.ReserveResolution(resolutionRequest);
        break;
    case ShadowMapType.AreaLightAtlas:
      m_AreaLightShadowAtlas.ReserveResolution(resolutionRequest);
        break;
    case ShadowMapType.CascadedDirectional:
      m_CascadeAtlas.ReserveResolution(resolutionRequest);
        break;
}
```

通过级联数据和剪裁结果计算剪裁平面。

```c
 cullResults.ComputeDirectionalShadowMatricesAndCullingPrimitives(lightIndex, (int)cascadeIndex, cascadeCount, ratios, (int)viewportSize.x, nearPlaneOffset, out view, out projection, out splitData);
            
```

ShadowAtlas的纹理声明：

```c
//	HDShadowManager:
// The cascade atlas will be allocated only if there is a directional light
m_Atlas = new HDShadowAtlas(renderPipelineResources, punctualLightAtlasInfo.shadowAtlasResolution, punctualLightAtlasInfo.shadowAtlasResolution, HDShaderIDs._ShadowAtlasSize, clearMaterial, false, depthBufferBits: punctualLightAtlasInfo.shadowAtlasDepthBits, name: "Shadow Map Atlas");
// Cascade atlas render texture will only be allocated if there is a shadow casting directional light
bool useMomentShadows = GetDirectionalShadowAlgorithm() == DirectionalShadowAlgorithm.IMS;
m_CascadeAtlas = new HDShadowAtlas(renderPipelineResources, 1, 1, HDShaderIDs._CascadeShadowAtlasSize, clearMaterial, useMomentShadows, depthBufferBits: directionalShadowDepthBits, name: "Cascade Shadow Map Atlas");

m_AreaLightShadowAtlas = new HDShadowAtlas(renderPipelineResources, areaLightAtlasInfo.shadowAtlasResolution, areaLightAtlasInfo.shadowAtlasResolution, HDShaderIDs._AreaShadowAtlasSize, clearMaterial, false, BlurredEVSM: true, depthBufferBits: areaLightAtlasInfo.shadowAtlasDepthBits, name: "Area Light Shadow Map Atlas");

```



在HDRenderPipelineAsset当中可以设置每种Atlas的大小，以及每种纹理深度的精度。

```c
// LightLoop
// 下面是阴影相关设置的使用位置。
void InitShadowSystem(HDRenderPipelineAsset hdAsset)
{
    m_ShadowInitParameters = hdAsset.currentPlatformRenderPipelineSettings.hdShadowInitParams;
    m_ShadowManager = new HDShadowManager(
        hdAsset.renderPipelineResources,
        m_ShadowInitParameters.directionalShadowsDepthBits,
        m_ShadowInitParameters.punctualLightShadowAtlas,
        m_ShadowInitParameters.areaLightShadowAtlas,
        m_ShadowInitParameters.maxShadowRequests,
        hdAsset.renderPipelineResources.shaders.shadowClearPS
    );
}
```



Unity支持的Soft Shadow 类型：

```c
public static DirectionalShadowAlgorithm GetDirectionalShadowAlgorithm()
{
    var hdAsset = (GraphicsSettings.renderPipelineAsset as HDRenderPipelineAsset);
    switch (hdAsset.currentPlatformRenderPipelineSettings.hdShadowInitParams.shadowQuality)
    {
        case HDShadowQuality.Low:
            {
                return DirectionalShadowAlgorithm.PCF5x5;
            }
        case HDShadowQuality.Medium:
            {
                return DirectionalShadowAlgorithm.PCF7x7;
            }
        case HDShadowQuality.High:
            {
                return DirectionalShadowAlgorithm.PCSS;
            }
        case HDShadowQuality.VeryHigh:
            {
                return DirectionalShadowAlgorithm.IMS;
            }
    };
    return DirectionalShadowAlgorithm.PCF5x5;
}
```

