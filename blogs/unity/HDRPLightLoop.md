# Light Loop

unity HDRP的Shader当中LightLoop是最终的光照计算函数部分。

Deferred Light的光照计算会经过这个函数。ForwarOnly的光照阶段也会经过这个函数。现在对着个函数进行详细的讲解。

LightLoop主要包含三个部分：

1. LightLoop.hlsl 是LightLoop的具体实现
2. LightLoopDef.hlsl是针对LightLoop当中的常用的工具和函数
3. 针对不同的材质，需要一个提供函数的xxx.hlsl文件，提供了在不同材质下LightLoop进行不同的光照计算。



## LightLoop主循环

下面是去掉Debug内容的LightLoop主循环的内容：

```c
void LightLoop( float3 V, PositionInputs posInput, PreLightData preLightData, BSDFData bsdfData, BuiltinData builtinData, uint featureFlags,
                out float3 diffuseLighting, // 返回的两种光照
                out float3 specularLighting)// 返回的两种光照
{
    // 保存了主要信息。
    LightLoopContext context;
	
    // 读取阴影相关内容。
    context.shadowContext    = InitShadowContext();
    context.contactShadow    = InitContactShadow(posInput);
    context.shadowValue      = 1;
    context.sampleReflection = 0;

    // First of all we compute the shadow value of the directional light to reduce the VGPR pressure
    // 标注了使用LightLoop计算的类型的类型  通过LightFeature和Flag进行各种匹配。
    // 这里表示这种材质类型需要计算直线光？？？
    if (featureFlags & LIGHTFEATUREFLAGS_DIRECTIONAL)
    {
        // Evaluate sun shadows.
        if (_DirectionalShadowIndex >= 0)
        {
            DirectionalLightData light = _DirectionalLightDatas[_DirectionalShadowIndex];

            // TODO: this will cause us to load from the normal buffer first. Does this cause a performance problem?
            // Also, the light direction is not consistent with the sun disk highlight hack, which modifies the light vector.
            float  NdotL            = dot(bsdfData.normalWS, -light.forward);
            float3 shadowBiasNormal = GetNormalForShadowBias(bsdfData);
            bool   evaluateShadows  = (NdotL > 0);

        #ifdef MATERIAL_INCLUDE_TRANSMISSION
            if (MaterialSupportsTransmission(bsdfData))
            {
                // We support some kind of transmission.
                if (HasFlag(bsdfData.materialFeatures, MATERIALFEATUREFLAGS_TRANSMISSION_MODE_THIN_THICKNESS))
                {
                    // We always evaluate shadows.
                    evaluateShadows = true;

                    // Care must be taken to bias in the direction of the light.
                    shadowBiasNormal *= FastSign(NdotL);
                }
                else
                {
                    // We only evaluate shadows for reflection, transmission shadows are handled separately.
                }
            }
        #endif

            if (evaluateShadows)
            {
                context.shadowValue = EvaluateRuntimeSunShadow(context, posInput, light, shadowBiasNormal);
            }
        }
    }

    // This struct is define in the material. the Lightloop must not access it
    // PostEvaluateBSDF call at the end will convert Lighting to diffuse and specular lighting
    AggregateLighting aggregateLighting;
    ZERO_INITIALIZE(AggregateLighting, aggregateLighting); // LightLoop is in charge of initializing the struct

    uint i = 0; // Declare once to avoid the D3D11 compiler warning.

    if (featureFlags & LIGHTFEATUREFLAGS_DIRECTIONAL)
    {
        for (i = 0; i < _DirectionalLightCount; ++i)
        {
            if (IsMatchingLightLayer(_DirectionalLightDatas[i].lightLayers, builtinData.renderingLayers))
            {
                DirectLighting lighting = EvaluateBSDF_Directional(context, V, posInput, preLightData, _DirectionalLightDatas[i], bsdfData, builtinData);
                AccumulateDirectLighting(lighting, aggregateLighting);
            }
        }
    }
    
     // 这里表示这种材质类型需要计算PUNCTUAL光？？？
    if (featureFlags & LIGHTFEATUREFLAGS_PUNCTUAL)
    {
        uint lightCount, lightStart;
        bool fastPath = false;

#ifndef LIGHTLOOP_DISABLE_TILE_AND_CLUSTER
        GetCountAndStart(posInput, LIGHTCATEGORY_PUNCTUAL, lightStart, lightCount);

#if SCALARIZE_LIGHT_LOOP
        // Fast path is when we all pixels in a wave are accessing same tile or cluster.
        uint lightStartLane0 = WaveReadLaneFirst(lightStart);
        fastPath = WaveActiveAllTrue(lightStart == lightStartLane0); 
#endif

#else   // LIGHTLOOP_DISABLE_TILE_AND_CLUSTER
        lightCount = _PunctualLightCount;
        lightStart = 0;
#endif

#if SCALARIZE_LIGHT_LOOP
        if (fastPath)
        {
            lightStart = lightStartLane0;
        }
#endif

        // Scalarized loop. All lights that are in a tile/cluster touched by any pixel in the wave are loaded (scalar load), only the one relevant to current thread/pixel are processed.
        // For clarity, the following code will follow the convention: variables starting with s_ are meant to be wave uniform (meant for scalar register),
        // v_ are variables that might have different value for each thread in the wave (meant for vector registers).
        // This will perform more loads than it is supposed to, however, the benefits should offset the downside, especially given that light data accessed should be largely coherent.
        // Note that the above is valid only if wave intriniscs are supported.
        uint v_lightListOffset = 0;
        uint v_lightIdx = lightStart;

        while (v_lightListOffset < lightCount)
        {
            v_lightIdx = FetchIndex(lightStart, v_lightListOffset);
            uint s_lightIdx = v_lightIdx;
#if SCALARIZE_LIGHT_LOOP
            if (!fastPath)
            {
                // If we are not in fast path, v_lightIdx is not scalar, so we need to query the Min value across the wave. 
                s_lightIdx = WaveActiveMin(v_lightIdx);
                // If WaveActiveMin returns 0xffffffff it means that all lanes are actually dead, so we can safely ignore the loop and move forward.
               // This could happen as an helper lane could reach this point, hence having a valid v_lightIdx, but their values will be ignored by the WaveActiveMin
                if (s_lightIdx == -1)
                {
                    break;
                }
            }
            // Note that the WaveReadLaneFirst should not be needed, but the compiler might insist in putting the result in VGPR.
            // However, we are certain at this point that the index is scalar.
            s_lightIdx = WaveReadLaneFirst(s_lightIdx);
#endif
            LightData s_lightData = FetchLight(s_lightIdx);

            // If current scalar and vector light index match, we process the light. The v_lightListOffset for current thread is increased.
            // Note that the following should really be ==, however, since helper lanes are not considered by WaveActiveMin, such helper lanes could
            // end up with a unique v_lightIdx value that is smaller than s_lightIdx hence being stuck in a loop. All the active lanes will not have this problem.
            if (s_lightIdx >= v_lightIdx)
            {
                v_lightListOffset++;
                if (IsMatchingLightLayer(s_lightData.lightLayers, builtinData.renderingLayers))
                {
                    DirectLighting lighting = EvaluateBSDF_Punctual(context, V, posInput, preLightData, s_lightData, bsdfData, builtinData);
                    AccumulateDirectLighting(lighting, aggregateLighting);
                }
            }
        }
    }
	
    
    
     // 这里表示这种材质类型需要计算区域光？？？
    if (featureFlags & LIGHTFEATUREFLAGS_AREA)
    {
        uint lightCount, lightStart;

    #ifndef LIGHTLOOP_DISABLE_TILE_AND_CLUSTER
        GetCountAndStart(posInput, LIGHTCATEGORY_AREA, lightStart, lightCount);
    #else
        lightCount = _AreaLightCount;
        lightStart = _PunctualLightCount;
    #endif

        // COMPILER BEHAVIOR WARNING!
        // If rectangle lights are before line lights, the compiler will duplicate light matrices in VGPR because they are used differently between the two types of lights.
        // By keeping line lights first we avoid this behavior and save substantial register pressure.
        // TODO: This is based on the current Lit.shader and can be different for any other way of implementing area lights, how to be generic and ensure performance ?

        if (lightCount > 0)
        {
            i = 0;

            uint      last      = lightCount - 1;
            LightData lightData = FetchLight(lightStart, i);

            while (i <= last && lightData.lightType == GPULIGHTTYPE_TUBE)
            {
                lightData.lightType = GPULIGHTTYPE_TUBE; // Enforce constant propagation
                lightData.cookieIndex = -1;              // Enforce constant propagation

                if (IsMatchingLightLayer(lightData.lightLayers, builtinData.renderingLayers))
                {
                    DirectLighting lighting = EvaluateBSDF_Area(context, V, posInput, preLightData, lightData, bsdfData, builtinData);
                    AccumulateDirectLighting(lighting, aggregateLighting);
                }

                lightData = FetchLight(lightStart, min(++i, last));
            }

            while (i <= last) // GPULIGHTTYPE_RECTANGLE
            {
                lightData.lightType = GPULIGHTTYPE_RECTANGLE; // Enforce constant propagation

                if (IsMatchingLightLayer(lightData.lightLayers, builtinData.renderingLayers))
                {
                    DirectLighting lighting = EvaluateBSDF_Area(context, V, posInput, preLightData, lightData, bsdfData, builtinData);
                    AccumulateDirectLighting(lighting, aggregateLighting);
                }

                lightData = FetchLight(lightStart, min(++i, last));
            }
        }
    }

    // Define macro for a better understanding of the loop
    // TODO: this code is now much harder to understand...
#define EVALUATE_BSDF_ENV_SKY(envLightData, TYPE, type) \
        IndirectLighting lighting = EvaluateBSDF_Env(context, V, posInput, preLightData, envLightData, bsdfData, envLightData.influenceShapeType, MERGE_NAME(GPUIMAGEBASEDLIGHTINGTYPE_, TYPE), MERGE_NAME(type, HierarchyWeight)); \
        AccumulateIndirectLighting(lighting, aggregateLighting);

// Environment cubemap test lightlayers, sky don't test it
#define EVALUATE_BSDF_ENV(envLightData, TYPE, type) if (IsMatchingLightLayer(envLightData.lightLayers, builtinData.renderingLayers)) { EVALUATE_BSDF_ENV_SKY(envLightData, TYPE, type) }

    // First loop iteration
    // 开启任何一个特殊效果：环境探针，天空球，屏幕空间反射，屏幕空间折射！！！
    if (featureFlags & (LIGHTFEATUREFLAGS_ENV | LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_SSREFRACTION | LIGHTFEATUREFLAGS_SSREFLECTION))
    {
        float reflectionHierarchyWeight = 0.0; // Max: 1.0
        float refractionHierarchyWeight = _EnableSSRefraction ? 0.0 : 1.0; // Max: 1.0

        uint envLightStart, envLightCount;

        bool fastPath = false;
        // Fetch first env light to provide the scene proxy for screen space computation
#ifndef LIGHTLOOP_DISABLE_TILE_AND_CLUSTER
        GetCountAndStart(posInput, LIGHTCATEGORY_ENV, envLightStart, envLightCount);

    #if SCALARIZE_LIGHT_LOOP
        // Fast path is when we all pixels in a wave is accessing same tile or cluster.
        uint envStartFirstLane = WaveReadLaneFirst(envLightStart);
        fastPath = WaveActiveAllTrue(envLightStart == envStartFirstLane); 
    #endif

#else   // LIGHTLOOP_DISABLE_TILE_AND_CLUSTER
        envLightCount = _EnvLightCount;
        envLightStart = 0;
#endif

        // Reflection / Refraction hierarchy is
        //  1. Screen Space Refraction / Reflection
        //  2. Environment Reflection / Refraction
        //  3. Sky Reflection / Refraction

        // Apply SSR.
    #if !defined(_SURFACE_TYPE_TRANSPARENT) && !defined(_DISABLE_SSR)
        {
            IndirectLighting indirect = EvaluateBSDF_ScreenSpaceReflection(posInput, preLightData, bsdfData,
                                                                           reflectionHierarchyWeight);
            AccumulateIndirectLighting(indirect, aggregateLighting);
        }
    #endif

        EnvLightData envLightData;
        if (envLightCount > 0)
        {
            envLightData = FetchEnvLight(envLightStart, 0);
        }
        else
        {
            envLightData = InitSkyEnvLightData(0);
        }

        // 计算屏幕空间折射 需要开启屏幕空间折射
        if ((featureFlags & LIGHTFEATUREFLAGS_SSREFRACTION) && (_EnableSSRefraction > 0))
        {
            IndirectLighting lighting = EvaluateBSDF_ScreenspaceRefraction(context, V, posInput, preLightData, bsdfData, envLightData, refractionHierarchyWeight);
            AccumulateIndirectLighting(lighting, aggregateLighting);
        }

        // Reflection probes are sorted by volume (in the increasing order).
        
     // 这里表示这种材质类型需要计算环境光：反射探针？
        if (featureFlags & LIGHTFEATUREFLAGS_ENV)
        {
            context.sampleReflection = SINGLE_PASS_CONTEXT_SAMPLE_REFLECTION_PROBES;
        #if SCALARIZE_LIGHT_LOOP
            if (fastPath)
            {
                envLightStart = envStartFirstLane;
            }
        #endif

            // Scalarized loop, same rationale of the punctual light version
            uint v_envLightListOffset = 0;
            uint v_envLightIdx = envLightStart;
            while (v_envLightListOffset < envLightCount)
            {
                v_envLightIdx = FetchIndex(envLightStart, v_envLightListOffset);
                uint s_envLightIdx = v_envLightIdx;

            #if SCALARIZE_LIGHT_LOOP
                if (!fastPath)
                {
                    s_envLightIdx = WaveActiveMin(v_envLightIdx);
                    // If we are not in fast path, s_envLightIdx is not scalar
                   // If WaveActiveMin returns 0xffffffff it means that all lanes are actually dead, so we can safely ignore the loop and move forward.
                   // This could happen as an helper lane could reach this point, hence having a valid v_lightIdx, but their values will be ignored by the WaveActiveMin
                    if (s_envLightIdx == -1)
                    {
                        break;
                    }
                }
                // Note that the WaveReadLaneFirst should not be needed, but the compiler might insist in putting the result in VGPR.
                // However, we are certain at this point that the index is scalar.
                s_envLightIdx = WaveReadLaneFirst(s_envLightIdx);

            #endif

                EnvLightData s_envLightData = FetchEnvLight(s_envLightIdx);    // Scalar load.

                // If current scalar and vector light index match, we process the light. The v_envLightListOffset for current thread is increased.
                // Note that the following should really be ==, however, since helper lanes are not considered by WaveActiveMin, such helper lanes could
                // end up with a unique v_envLightIdx value that is smaller than s_envLightIdx hence being stuck in a loop. All the active lanes will not have this problem.
                if (s_envLightIdx >= v_envLightIdx)
                {
                    v_envLightListOffset++;
                    if (reflectionHierarchyWeight < 1.0)
                    {
                        EVALUATE_BSDF_ENV(s_envLightData, REFLECTION, reflection);
                    }
                    // Refraction probe and reflection probe will process exactly the same weight. It will be good for performance to be able to share this computation
                    // However it is hard to deal with the fact that reflectionHierarchyWeight and refractionHierarchyWeight have not the same values, they are independent
                    // The refraction probe is rarely used and happen only with sphere shape and high IOR. So we accept the slow path that use more simple code and
                    // doesn't affect the performance of the reflection which is more important.
                    // We reuse LIGHTFEATUREFLAGS_SSREFRACTION flag as refraction is mainly base on the screen. Would be a waste to not use screen and only cubemap.
                    if ((featureFlags & LIGHTFEATUREFLAGS_SSREFRACTION) && (refractionHierarchyWeight < 1.0))
                    {
                        EVALUATE_BSDF_ENV(s_envLightData, REFRACTION, refraction);
                    }
                }

            }
        }

        // Only apply the sky IBL if the sky texture is available
        if ((featureFlags & LIGHTFEATUREFLAGS_SKY) && _EnvLightSkyEnabled)
        {
            // The sky is a single cubemap texture separate from the reflection probe texture array (different resolution and compression)
            context.sampleReflection = SINGLE_PASS_CONTEXT_SAMPLE_SKY;

            // The sky data are generated on the fly so the compiler can optimize the code
            EnvLightData envLightSky = InitSkyEnvLightData(0);

            // Only apply the sky if we haven't yet accumulated enough IBL lighting.
            if (reflectionHierarchyWeight < 1.0)
            {
                EVALUATE_BSDF_ENV_SKY(envLightSky, REFLECTION, reflection);
            }

            if ((featureFlags & LIGHTFEATUREFLAGS_SSREFRACTION) && (refractionHierarchyWeight < 1.0))
            {
                EVALUATE_BSDF_ENV_SKY(envLightSky, REFRACTION, refraction);
            }
        }
    }
#undef EVALUATE_BSDF_ENV
#undef EVALUATE_BSDF_ENV_SKY    

    // Also Apply indiret diffuse (GI)
    // PostEvaluateBSDF will perform any operation wanted by the material and sum everything into diffuseLighting and specularLighting
    PostEvaluateBSDF(   context, V, posInput, preLightData, bsdfData, builtinData, aggregateLighting,
                        diffuseLighting, specularLighting);

    //ApplyDebug(context, posInput.positionWS, diffuseLighting, specularLighting);
}
```

上面大概标注了主循环是什么，看起来很复杂，下面每一个部分诸葛分析。

##AggregateLighting

```c
struct DirectLighting
{
    float3 diffuse;
    float3 specular;
};

struct IndirectLighting
{
    float3 specularReflected;
    float3 specularTransmitted;
};

struct AggregateLighting
{
    DirectLighting   direct;
    IndirectLighting indirect;
};
```



## LightFeature

在外部透明物体和不透明物体有不同的featureFlags:

```c
#define LIGHT_FEATURE_MASK_FLAGS (16773120)             111111111111000000000000
#define LIGHT_FEATURE_MASK_FLAGS_OPAQUE (16642048)      111111011111000000000000
#define LIGHT_FEATURE_MASK_FLAGS_TRANSPARENT (16510976) 111110111111000000000000

```

在ShaderPassForward当中有这样一段代码：

```c
#ifdef _SURFACE_TYPE_TRANSPARENT
        uint featureFlags = LIGHT_FEATURE_MASK_FLAGS_TRANSPARENT;
#else
        uint featureFlags = LIGHT_FEATURE_MASK_FLAGS_OPAQUE;
#endif
```

主要用于标注不同光照计算的特性

使用的地方有限：

```c
1. D:\srp\ScriptableRenderPipeline\com.unity.render-pipelines.high-definition\Runtime\Lighting\Deferred.shader:
  LightLoop(V, posInput, preLightData, bsdfData, builtinData, LIGHT_FEATURE_MASK_FLAGS_OPAQUE, diffuseLighting, specularLighting);

2. D:\srp\ScriptableRenderPipeline\com.unity.render-pipelines.high-definition\Runtime\Lighting\LightLoop\DeferredTile.shader:
  LightLoop(V, posInput, preLightData, bsdfData, builtinData, LIGHT_FEATURE_MASK_FLAGS_OPAQUE, diffuseLighting, specularLighting);

3. D:\srp\ScriptableRenderPipeline\com.unity.render-pipelines.high-definition\Runtime\RenderPipeline\ShaderPass\ShaderPassForward.hlsl:
{
    #ifdef _SURFACE_TYPE_TRANSPARENT
    uint featureFlags = LIGHT_FEATURE_MASK_FLAGS_TRANSPARENT;
    #else
    uint featureFlags = LIGHT_FEATURE_MASK_FLAGS_OPAQUE;
    #endif
    float3 diffuseLighting;
```

还找到了 下面一段代码

```c
static const uint kFeatureVariantFlags[NUM_FEATURE_VARIANTS] =
{
    // Precomputed illumination (no dynamic lights) for all material types
    /*  0 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_ENV | LIGHTFEATUREFLAGS_SSREFLECTION | MATERIAL_FEATURE_MASK_FLAGS,

    /*  1 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_PUNCTUAL | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /*  2 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_AREA | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /*  3 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_ENV | LIGHTFEATUREFLAGS_SSREFLECTION | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /*  4 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_PUNCTUAL | LIGHTFEATUREFLAGS_ENV | LIGHTFEATUREFLAGS_SSREFLECTION | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /*  5 */ LIGHT_FEATURE_MASK_FLAGS_OPAQUE | MATERIALFEATUREFLAGS_LIT_STANDARD,

    // Standard with SSS and Transmission
    /*  6 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_PUNCTUAL | MATERIALFEATUREFLAGS_LIT_SUBSURFACE_SCATTERING | MATERIALFEATUREFLAGS_LIT_TRANSMISSION | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /*  7 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_AREA | MATERIALFEATUREFLAGS_LIT_SUBSURFACE_SCATTERING | MATERIALFEATUREFLAGS_LIT_TRANSMISSION | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /*  8 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_ENV | LIGHTFEATUREFLAGS_SSREFLECTION | MATERIALFEATUREFLAGS_LIT_SUBSURFACE_SCATTERING | MATERIALFEATUREFLAGS_LIT_TRANSMISSION | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /*  9 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_PUNCTUAL | LIGHTFEATUREFLAGS_ENV | LIGHTFEATUREFLAGS_SSREFLECTION | MATERIALFEATUREFLAGS_LIT_SUBSURFACE_SCATTERING | MATERIALFEATUREFLAGS_LIT_TRANSMISSION | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /* 10 */ LIGHT_FEATURE_MASK_FLAGS_OPAQUE | MATERIALFEATUREFLAGS_LIT_SUBSURFACE_SCATTERING | MATERIALFEATUREFLAGS_LIT_TRANSMISSION | MATERIALFEATUREFLAGS_LIT_STANDARD,

    // Anisotropy
    /* 11 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_PUNCTUAL | MATERIALFEATUREFLAGS_LIT_ANISOTROPY | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /* 12 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_AREA | MATERIALFEATUREFLAGS_LIT_ANISOTROPY | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /* 13 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_ENV | LIGHTFEATUREFLAGS_SSREFLECTION | MATERIALFEATUREFLAGS_LIT_ANISOTROPY | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /* 14 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_PUNCTUAL | LIGHTFEATUREFLAGS_ENV | LIGHTFEATUREFLAGS_SSREFLECTION | MATERIALFEATUREFLAGS_LIT_ANISOTROPY | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /* 15 */ LIGHT_FEATURE_MASK_FLAGS_OPAQUE | MATERIALFEATUREFLAGS_LIT_ANISOTROPY | MATERIALFEATUREFLAGS_LIT_STANDARD,

    // Standard with clear coat
    /* 16 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_PUNCTUAL | MATERIALFEATUREFLAGS_LIT_CLEAR_COAT | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /* 17 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_AREA | MATERIALFEATUREFLAGS_LIT_CLEAR_COAT | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /* 18 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_ENV | LIGHTFEATUREFLAGS_SSREFLECTION | MATERIALFEATUREFLAGS_LIT_CLEAR_COAT | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /* 19 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_PUNCTUAL | LIGHTFEATUREFLAGS_ENV | LIGHTFEATUREFLAGS_SSREFLECTION | MATERIALFEATUREFLAGS_LIT_CLEAR_COAT | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /* 20 */ LIGHT_FEATURE_MASK_FLAGS_OPAQUE | MATERIALFEATUREFLAGS_LIT_CLEAR_COAT | MATERIALFEATUREFLAGS_LIT_STANDARD,

    // Standard with Iridescence
    /* 21 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_PUNCTUAL | MATERIALFEATUREFLAGS_LIT_IRIDESCENCE | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /* 22 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_AREA | MATERIALFEATUREFLAGS_LIT_IRIDESCENCE | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /* 23 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_ENV | LIGHTFEATUREFLAGS_SSREFLECTION | MATERIALFEATUREFLAGS_LIT_IRIDESCENCE | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /* 24 */ LIGHTFEATUREFLAGS_SKY | LIGHTFEATUREFLAGS_DIRECTIONAL | LIGHTFEATUREFLAGS_PUNCTUAL | LIGHTFEATUREFLAGS_ENV | LIGHTFEATUREFLAGS_SSREFLECTION | MATERIALFEATUREFLAGS_LIT_IRIDESCENCE | MATERIALFEATUREFLAGS_LIT_STANDARD,
    /* 25 */ LIGHT_FEATURE_MASK_FLAGS_OPAQUE | MATERIALFEATUREFLAGS_LIT_IRIDESCENCE | MATERIALFEATUREFLAGS_LIT_STANDARD,

    /* 26 */ LIGHT_FEATURE_MASK_FLAGS_OPAQUE | MATERIAL_FEATURE_MASK_FLAGS, // Catch all case with MATERIAL_FEATURE_MASK_FLAGS is needed in case we disable material classification
};
```





## 直线光部分

下面对直线光部分的代码分析, **因为直线光是全部照亮的所以不使用Tiled**:

```c
// 这个if用来计算直线光阴影： 输出到 context.shadowValue当中 
if (featureFlags & LIGHTFEATUREFLAGS_DIRECTIONAL)
    /*
     LIGHTFEATUREFLAGS_DIRECTIONAL         =          100000000000000
     LIGHT_FEATURE_MASK_FLAGS_OPAQUE       = 111111011111000000000000
     LIGHT_FEATURE_MASK_FLAGS_TRANSPARENT  = 111110111111000000000000
    下面两个是透明和不透明支持不一样的,也就是：透明不支持反射，不透明不支持折射。
     LIGHTFEATUREFLAGS_SSREFRACTION        =       100000000000000000
     LIGHTFEATUREFLAGS_SSREFLECTION        =      1000000000000000000  
    */
{
    // Evaluate sun shadows.
    if (_DirectionalShadowIndex >= 0)
    {
        // 太阳光信息，_DirectionalShadowIndex说明见标题：_DirectionalShadowIndex。
        DirectionalLightData light = _DirectionalLightDatas[_DirectionalShadowIndex];

        // TODO: this will cause us to load from the normal buffer first. Does this cause a performance problem?
        // Also, the light direction is not consistent with the sun disk highlight hack, which modifies the light vector.
        float  NdotL            = dot(bsdfData.normalWS, -light.forward);
        float3 shadowBiasNormal = GetNormalForShadowBias(bsdfData);
        bool   evaluateShadows  = (NdotL > 0);  // 只有迎光面需要计算阴影

        #ifdef MATERIAL_INCLUDE_TRANSMISSION
        if (MaterialSupportsTransmission(bsdfData))
        {
            // We support some kind of transmission.
            if (HasFlag(bsdfData.materialFeatures, MATERIALFEATUREFLAGS_TRANSMISSION_MODE_THIN_THICKNESS))
            {
                // We always evaluate shadows.
                evaluateShadows = true;

                // Care must be taken to bias in the direction of the light.
                shadowBiasNormal *= FastSign(NdotL);
            }
            else
            {
                // We only evaluate shadows for reflection, transmission shadows are handled separately.
            }
        }
        #endif
        // 计算阴影值
        if (evaluateShadows)
        {
            context.shadowValue = EvaluateRuntimeSunShadow(context, posInput, light, shadowBiasNormal);
        }
    }
}

// This struct is define in the material. the Lightloop must not access it
// PostEvaluateBSDF call at the end will convert Lighting to diffuse and specular lighting
// 主要用来积累计算不同的光照，内容见：AggregateLighting二级目录
AggregateLighting aggregateLighting;
ZERO_INITIALIZE(AggregateLighting, aggregateLighting); 

uint i = 0; 

// 积累直线光
if (featureFlags & LIGHTFEATUREFLAGS_DIRECTIONAL)
{
    for (i = 0; i < _DirectionalLightCount; ++i)
    {
        // 光照层过滤，制定光照照在不同的物体上。
        if (IsMatchingLightLayer(_DirectionalLightDatas[i].lightLayers, builtinData.renderingLayers))
        {
            // 这里是计算直线光  ，考虑一下Tiledlight在哪里产生作用？？
            DirectLighting lighting = EvaluateBSDF_Directional(context, V, posInput, preLightData, _DirectionalLightDatas[i], bsdfData, builtinData);
            
            // 叠加光
            AccumulateDirectLighting(lighting, aggregateLighting);
        }
    }
}
```

###_DirectionalShadowIndex

下面主要说明了_DirectionalShadowIndex是用来标注太阳光的索引。

```c
D:\srp\ScriptableRenderPipeline\com.unity.render-pipelines.high-definition\Runtime\Lighting\LightLoop\LightLoop.cs:
 2605                  if (sunLightShadow)
 2606                  {
 2607:                     cmd.SetGlobalInt(HDShaderIDs._DirectionalShadowIndex, m_CurrentShadowSortedSunLightIndex);
 2608                  }
 2609                  else
 2610                  {
 2611:                     cmd.SetGlobalInt(HDShaderIDs._DirectionalShadowIndex, -1);
 2612                  }
```

###EvaluateBSDF_Directional

这个主要用来输入光源参数、材质参数，得到最终的光的颜色。

这个函数包含在各种光照模型中：

Hair.hlsl

Fabric.hlsl

AxF.hlsl

Lit.hlsl

StackLit.hlsl

这里先看Lit.hlsl当中如何实现的。

```c

DirectLighting EvaluateBSDF_Directional(LightLoopContext lightLoopContext,
                                        float3 V, PositionInputs posInput, PreLightData preLightData,
                                        DirectionalLightData lightData, BSDFData bsdfData,
                                        BuiltinData builtinData)
{
    return ShadeSurface_Directional(lightLoopContext, posInput, builtinData, preLightData, lightData,
                                    bsdfData, bsdfData.normalWS, V);
}
```

再看头发

```c
DirectLighting EvaluateBSDF_Directional(LightLoopContext lightLoopContext,
                                        float3 V, PositionInputs posInput, PreLightData preLightData,
                                        DirectionalLightData lightData, BSDFData bsdfData,
                                        BuiltinData builtinData)
{
    return ShadeSurface_Directional(lightLoopContext, posInput, builtinData, preLightData, lightData,
                                    bsdfData, bsdfData.normalWS, V);
}
```

他们都引入了一个ShadeSurface_Directional

### ShadeSurface_Directional

ShadeSurface_Directional在同一个函数当中。

Lighting\SurfaceShading.hlsl

```c
DirectLighting ShadeSurface_Directional(LightLoopContext lightLoopContext,
                                        PositionInputs posInput, BuiltinData builtinData,
                                        PreLightData preLightData, DirectionalLightData light,
                                        BSDFData bsdfData, float3 N, float3 V)
{
    DirectLighting lighting;
    ZERO_INITIALIZE(DirectLighting, lighting);

    float3 L     = ComputeSunLightDirection(light, N, V);
    float  NdotL = dot(N, L); // Do not saturate

    // Note: We use NdotL here to early out, but in case of clear coat this is not correct. But we are OK with this
    bool surfaceReflection = NdotL > 0;

    // Caution: this function modifies N, NdotL, contactShadowIndex and shadowMaskSelector.
    float3 transmittance = PreEvaluateDirectionalLightTransmission(bsdfData, light, N, NdotL);

    float3 color; float attenuation;
    EvaluateLight_Directional(lightLoopContext, posInput, light, builtinData, N, L, NdotL,
                              color, attenuation);
	
    // 这里就是根据雾的透视率和阴影计算光的颜色。
    // TODO: transmittance contributes to attenuation, how can we use it for early-out?
    if (attenuation > 0)
    {
        // We must clamp here, otherwise our disk light hack for smooth surfaces does not work.
        // Explanation: for a perfectly smooth surface, lighting is only reflected if (NdotL = NdotV).
        // This implies that (NdotH = 1).
        // Due to the floating point arithmetic (see math in ComputeSunLightDirection() and
        // GetBSDFAngle()), we will never arrive at this exact number, so no lighting will be reflected.
        // If we increase the roughness somewhat, the trick still works.
        ClampRoughness(bsdfData, light.minRoughness);

        float3 diffuseBsdf, specularBsdf;
        BSDF(V, L, NdotL, posInput.positionWS, preLightData, bsdfData, diffuseBsdf, specularBsdf);
		
        // 光的摄入方向。
        if (surfaceReflection)
        {
            attenuation    *= ComputeMicroShadowing(bsdfData, NdotL);
            float intensity = attenuation * NdotL;

            lighting.diffuse  = diffuseBsdf  * (intensity * light.diffuseDimmer);
            lighting.specular = specularBsdf * (intensity * light.specularDimmer);
        }
        // 光在反面同时开启了透射。
        else if (MaterialSupportsTransmission(bsdfData))
        {
             // Apply wrapped lighting to better handle thin objects at grazing angles.
            // 如果是材质支持透视？  则启用边缘光。
            float wrapNdotL = ComputeWrappedDiffuseLighting(NdotL, TRANSMISSION_WRAP_LIGHT);
            float intensity = attenuation * wrapNdotL;

            // We use diffuse lighting for accumulation since it is going to be blurred during the SSS pass.
            // Note: Disney's LdoV term in 'diffuseBsdf' does not hold a meaningful value
            // in the context of transmission, but we keep it unaltered for performance reasons.
            lighting.diffuse  = transmittance * (diffuseBsdf * (intensity * light.diffuseDimmer));
            // 高光没有透射光。
            lighting.specular = 0; // No spec trans, the compiler should optimize
        }

        // Save ALU by applying light and cookie colors only once.
        lighting.diffuse  *= color;
        lighting.specular *= color;
    }

    return lighting;
}
```

### EvaluateLight_Directional

用来计算光,输入位置 ，光线，纹理和方向信息，输出光的颜色和衰减。

```c
void EvaluateLight_Directional(LightLoopContext lightLoopContext, PositionInputs posInput,
                               DirectionalLightData light, BuiltinData builtinData,
                               float3 N, float3 L, float NdotL,
                               out float3 color, out float attenuation)
{
    //light 信息是从外部C#设置的
    // 初始化
    // 衰减由阴影和雾影响，颜色是Cookie
    color = attenuation = 0;
    // early Out  背面 和 光颜色削减为0  不进行光照。
    if ((light.lightDimmer <= 0) || (NdotL <= 0)) return;

    // 没有阴影开始
    float3 positionWS = posInput.positionWS;
    float  shadow     = 1.0;
    float  shadowMask = 1.0;
	
    // 读取光的信息 ，开始没有衰减
    color       = light.color;
    attenuation = 1.0;

    // Height fog attenuation. 根据高度雾计算光照衰减。
    {
        float cosZenithAngle = L.y;
        float fragmentHeight = posInput.positionWS.y;
        attenuation *= TransmittanceHeightFog(_HeightFogBaseExtinction, _HeightFogBaseHeight,
                                              _HeightFogExponents, cosZenithAngle, fragmentHeight);
    }
	
    // cookie 纹理
    if (light.cookieIndex >= 0)
    {
        float3 lightToSample = positionWS - light.positionRWS;
        float3 cookie = EvaluateCookie_Directional(lightLoopContext, light, lightToSample);

        color *= cookie; // 用cookie衰减纹理。
    }

#ifdef SHADOWS_SHADOWMASK
    // shadowMaskSelector.x is -1 if there is no shadow mask
    // Note that we override shadow value (in case we don't have any dynamic shadow)
    shadow = shadowMask = (light.shadowMaskSelector.x >= 0.0) ? dot(BUILTIN_DATA_SHADOW_MASK, light.shadowMaskSelector) : 1.0;
#endif

    if ((light.shadowIndex >= 0) && (light.shadowDimmer > 0))
    {
        shadow = lightLoopContext.shadowValue;

    #ifdef SHADOWS_SHADOWMASK
        // TODO: Optimize this code! Currently it is a bit like brute force to get the last transistion and fade to shadow mask, but there is
        // certainly more efficient to do
        // We reuse the transition from the cascade system to fade between shadow mask at max distance
        uint  payloadOffset;
        real  fade;
        int cascadeCount;
        int shadowSplitIndex = 0;

        shadowSplitIndex = EvalShadow_GetSplitIndex(lightLoopContext.shadowContext, light.shadowIndex, positionWS, fade, cascadeCount);

        // we have a fade caclulation for each cascade but we must lerp with shadow mask only for the last one
        // if shadowSplitIndex is -1 it mean we are outside cascade and should return 1.0 to use shadowmask: saturate(-shadowSplitIndex) return 0 for >= 0 and 1 for -1
        fade = ((shadowSplitIndex + 1) == cascadeCount) ? fade : saturate(-shadowSplitIndex);

        // In the transition code (both dithering and blend) we use shadow = lerp( shadow, 1.0, fade ) for last transition
        // mean if we expend the code we have (shadow * (1 - fade) + fade). Here to make transition with shadow mask
        // we will remove fade and add fade * shadowMask which mean we do a lerp with shadow mask
        shadow = shadow - fade + fade * shadowMask;

        // See comment in EvaluateBSDF_Punctual
        shadow = light.nonLightMappedOnly ? min(shadowMask, shadow) : shadow;
    #endif

        shadow = lerp(shadowMask, shadow, light.shadowDimmer);
    }

    // Transparents have no contact shadow information
#ifndef _SURFACE_TYPE_TRANSPARENT
    shadow = min(shadow, GetContactShadow(lightLoopContext, light.contactShadowIndex));
#endif

	
    attenuation *= shadow;
}
```

## Point 和 Spot

这两种灯光和距离相关，同时需要用到Tiled

```c
if (featureFlags & LIGHTFEATUREFLAGS_PUNCTUAL)
{
    uint lightCount, lightStart;
    bool fastPath = false;

    #ifndef LIGHTLOOP_DISABLE_TILE_AND_CLUSTER
    GetCountAndStart(posInput, LIGHTCATEGORY_PUNCTUAL, lightStart, lightCount);

    #if SCALARIZE_LIGHT_LOOP
    // Fast path is when we all pixels in a wave are accessing same tile or cluster.
    uint lightStartLane0 = WaveReadLaneFirst(lightStart);
    fastPath = WaveActiveAllTrue(lightStart == lightStartLane0); 
    #endif

    #else   // LIGHTLOOP_DISABLE_TILE_AND_CLUSTER
    lightCount = _PunctualLightCount;
    lightStart = 0;
    #endif

    #if SCALARIZE_LIGHT_LOOP
    if (fastPath)
    {
        lightStart = lightStartLane0;
    }
    #endif

    // Scalarized loop. All lights that are in a tile/cluster touched by any pixel in the wave are loaded (scalar load), only the one relevant to current thread/pixel are processed.
    // For clarity, the following code will follow the convention: variables starting with s_ are meant to be wave uniform (meant for scalar register),
    // v_ are variables that might have different value for each thread in the wave (meant for vector registers).
    // This will perform more loads than it is supposed to, however, the benefits should offset the downside, especially given that light data accessed should be largely coherent.
    // Note that the above is valid only if wave intriniscs are supported.
    uint v_lightListOffset = 0;
    uint v_lightIdx = lightStart;

    while (v_lightListOffset < lightCount)
    {
        v_lightIdx = FetchIndex(lightStart, v_lightListOffset);
        uint s_lightIdx = v_lightIdx;
        #if SCALARIZE_LIGHT_LOOP
        if (!fastPath)
        {
            // If we are not in fast path, v_lightIdx is not scalar, so we need to query the Min value across the wave. 
            s_lightIdx = WaveActiveMin(v_lightIdx);
            // If WaveActiveMin returns 0xffffffff it means that all lanes are actually dead, so we can safely ignore the loop and move forward.
            // This could happen as an helper lane could reach this point, hence having a valid v_lightIdx, but their values will be ignored by the WaveActiveMin
            if (s_lightIdx == -1)
            {
                break;
            }
        }
        // Note that the WaveReadLaneFirst should not be needed, but the compiler might insist in putting the result in VGPR.
        // However, we are certain at this point that the index is scalar.
        s_lightIdx = WaveReadLaneFirst(s_lightIdx);
        #endif
        
        // 上面一部分计算应该是 光源计算的Tiled算法有关。，这里获取到光照信息。
        LightData s_lightData = FetchLight(s_lightIdx);

        // If current scalar and vector light index match, we process the light. The v_lightListOffset for current thread is increased.
        // Note that the following should really be ==, however, since helper lanes are not considered by WaveActiveMin, such helper lanes could
        // end up with a unique v_lightIdx value that is smaller than s_lightIdx hence being stuck in a loop. All the active lanes will not have this problem.
        if (s_lightIdx >= v_lightIdx)
        {
            v_lightListOffset++;
            if (IsMatchingLightLayer(s_lightData.lightLayers, builtinData.renderingLayers))
            {
                DirectLighting lighting = EvaluateBSDF_Punctual(context, V, posInput, preLightData, s_lightData, bsdfData, builtinData);
                AccumulateDirectLighting(lighting, aggregateLighting);
            }
        }
    }
}

```

### ShadeSurface_Punctual

这个函数和指向光部分的一样

```c
DirectLighting ShadeSurface_Punctual(LightLoopContext lightLoopContext,
                                     PositionInputs posInput, BuiltinData builtinData,
                                     PreLightData preLightData, LightData light,
                                     BSDFData bsdfData, float3 N, float3 V)
{
    DirectLighting lighting;
    ZERO_INITIALIZE(DirectLighting, lighting);

    float3 L;
    float3 lightToSample;
    float4 distances; // {d, d^2, 1/d, d_proj}
    GetPunctualLightVectors(posInput.positionWS, light, L, lightToSample, distances);

    float NdotL = dot(N, L); // Do not saturate

    // Note: We use NdotL here to early out, but in case of clear coat this is not correct. But we are OK with this
    bool surfaceReflection = NdotL > 0;

    // Caution: this function modifies N, NdotL, shadowIndex, contactShadowIndex and shadowMaskSelector. 
    // 计算光照投射
    float3 transmittance = PreEvaluatePunctualLightTransmission(lightLoopContext, posInput, bsdfData,
                                                                light, distances.x, N, L, NdotL);
    float3 color; float attenuation;
    EvaluateLight_Punctual(lightLoopContext, posInput, light, builtinData, N, L, NdotL, lightToSample, distances,
                           color, attenuation);

    // TODO: transmittance contributes to attenuation, how can we use it for early-out?
    if (attenuation > 0)
    {
        // Simulate a sphere/disk light with this hack
        // Note that it is not correct with our pre-computation of PartLambdaV (mean if we disable the optimization we will not have the
        // same result) but we don't care as it is a hack anyway
        ClampRoughness(bsdfData, light.minRoughness);

        float3 diffuseBsdf, specularBsdf;
        BSDF(V, L, NdotL, posInput.positionWS, preLightData, bsdfData, diffuseBsdf, specularBsdf);

        if (surfaceReflection)
        {
            float intensity = attenuation * NdotL;

            lighting.diffuse  = diffuseBsdf  * (intensity * light.diffuseDimmer);
            lighting.specular = specularBsdf * (intensity * light.specularDimmer);
        }
        else if (MaterialSupportsTransmission(bsdfData))
        {
             // Apply wrapped lighting to better handle thin objects at grazing angles.
            float wrapNdotL = ComputeWrappedDiffuseLighting(NdotL, TRANSMISSION_WRAP_LIGHT);
            float intensity = attenuation * wrapNdotL;

            // We use diffuse lighting for accumulation since it is going to be blurred during the SSS pass.
            // Note: Disney's LdoV term in 'diffuseBsdf' does not hold a meaningful value
            // in the context of transmission, but we keep it unaltered for performance reasons.
            lighting.diffuse  = transmittance * (diffuseBsdf * (intensity * light.diffuseDimmer));
            lighting.specular = 0; // No spec trans, the compiler should optimize
        }

        // Save ALU by applying light and cookie colors only once.
        lighting.diffuse  *= color;
        lighting.specular *= color;
    }

    return lighting;
}
```

### PreEvaluatePunctualLightTransmission

这个函数用来计算光照的透射度，可能会修改法线。

```c
// This function return transmittance to provide to EvaluateTransmission
float3 PreEvaluatePunctualLightTransmission(LightLoopContext lightLoopContext,
                                            PositionInputs posInput, BSDFData bsdfData,
                                            inout LightData light, float distFrontFaceToLight,
                                            inout float3 N, float3 L, inout float NdotL)
{
    float3 transmittance = 0;
// 判断功能是否开启 如果材质没有包含透射 就没有透射
#ifdef MATERIAL_INCLUDE_TRANSMISSION
    // 判断材质是否支持
    if (MaterialSupportsTransmission(bsdfData))
    {
        // We support some kind of transmission.
        // 背面才会生效。
        if (NdotL <= 0)
        {
            // And since the light is back-facing, it's active.
            // Care must be taken to bias in the direction of the light.
            // TODO: change the sign of the bias: faster & uses fewer VGPRs.
            //扭转法线！！！！！！！！
            N = -N;

            // We want to evaluate cookies and light attenuation, so we flip NdotL.
            // 扭转计算结果
            NdotL = -NdotL;

            // However, we don't want baked or contact shadows.
            
            // 关闭阴影
            light.contactShadowIndex   = -1;
            light.shadowMaskSelector.x = -1;
			
            // 材质的透视度。
            transmittance = bsdfData.transmittance;
			
            //
            if (!HasFlag(bsdfData.materialFeatures, MATERIALFEATUREFLAGS_TRANSMISSION_MODE_THIN_THICKNESS) && (light.shadowIndex >= 0))
            {
                // We can compute thickness from shadow.
                // Compute the distance from the light to the back face of the object along the light direction.
                // TODO: SHADOW BIAS.
                float distBackFaceToLight = GetPunctualShadowClosestDistance(lightLoopContext.shadowContext, s_linear_clamp_sampler,
                                                                             posInput.positionWS, light.shadowIndex, L, light.positionRWS,
                                                                             light.lightType == GPULIGHTTYPE_POINT);

                // Our subsurface scattering models use the semi-infinite planar slab assumption.
                // Therefore, we need to find the thickness along the normal.
                // Warning: based on the artist's input, dependence on the NdotL has been disabled.
                float thicknessInUnits       = (distFrontFaceToLight - distBackFaceToLight) /* * -NdotL */;
                float thicknessInMeters      = thicknessInUnits * _WorldScales[bsdfData.diffusionProfile].x;
                float thicknessInMillimeters = thicknessInMeters * MILLIMETERS_PER_METER;

                // We need to make sure it's not less than the baked thickness to minimize light leaking.
                float thicknessDelta = max(0, thicknessInMillimeters - bsdfData.thickness);

                float3 S = _ShapeParams[bsdfData.diffusionProfile].rgb;

            #if 0
                float3 expOneThird = exp(((-1.0 / 3.0) * thicknessDelta) * S);
            #else
                // Help the compiler. S is premultiplied by ((-1.0 / 3.0) * LOG2_E) on the CPU.
                float3 p = thicknessDelta * S;
                float3 expOneThird = exp2(p);
            #endif

                // Approximate the decrease of transmittance by e^(-1/3 * dt * S).
                transmittance *= expOneThird;

                // Avoid double shadowing. TODO: is there a faster option?
                light.shadowIndex = -1;

                // Note: we do not modify the distance to the light, or the light angle for the back face.
                // This is a performance-saving optimization which makes sense as long as the thickness is small.
            }
        }
    }
#endif

    return transmittance;
}
```



## 区域光部分



##屏幕空间反射部分



## 反射探针部分



## 天空球部分





## 光照探针、lightmap

PostEvaluateBSDF计算了间接光

```c


void PostEvaluateBSDF(  LightLoopContext lightLoopContext,
                        float3 V, PositionInputs posInput,
                        PreLightData preLightData, BSDFData bsdfData, BuiltinData builtinData, AggregateLighting lighting,
                        out float3 diffuseLighting, out float3 specularLighting)
{
    AmbientOcclusionFactor aoFactor;
    // Use GTAOMultiBounce approximation for ambient occlusion (allow to get a tint from the baseColor)
#if 0
    GetScreenSpaceAmbientOcclusion(posInput.positionSS, preLightData.NdotV, bsdfData.perceptualRoughness, bsdfData.ambientOcclusion, bsdfData.specularOcclusion, aoFactor);
#else
    GetScreenSpaceAmbientOcclusionMultibounce(posInput.positionSS, preLightData.NdotV, bsdfData.perceptualRoughness, bsdfData.ambientOcclusion, bsdfData.specularOcclusion, bsdfData.diffuseColor, bsdfData.fresnel0, aoFactor);
#endif
    // AmbientOcclusion 技术需要了解一下
    ApplyAmbientOcclusionFactor(aoFactor, builtinData, lighting);

    // Subsurface scattering mode
    // SSS效果也是在这里应用的
    float3 modifiedDiffuseColor = GetModifiedDiffuseColorForSSS(bsdfData);

    // Apply the albedo to the direct diffuse lighting (only once). The indirect (baked)
    // diffuse lighting has already multiply the albedo in ModifyBakedDiffuseLighting().
    // Note: In deferred bakeDiffuseLighting also contain emissive and in this case emissiveColor is 0
    // 漫反射光包含了前面计算 经过模糊的SSS光， 烘焙的关照贴图和自发光。
    diffuseLighting = modifiedDiffuseColor * lighting.direct.diffuse + builtinData.bakeDiffuseLighting + builtinData.emissiveColor;

    // If refraction is enable we use the transmittanceMask to lerp between current diffuse lighting and refraction value
    // Physically speaking, transmittanceMask should be 1, but for artistic reasons, we let the value vary
    //
    // Note we also transfer the refracted light (lighting.indirect.specularTransmitted) into diffuseLighting
    // since we know it won't be further processed: it is called at the end of the LightLoop(), but doing this
    // enables opacity to affect it (in ApplyBlendMode()) while the rest of specularLighting escapes it.
    
    // 如果有折射则还需要和折射进行混合
#if HAS_REFRACTION
    diffuseLighting = lerp(diffuseLighting, lighting.indirect.specularTransmitted, bsdfData.transmittanceMask * _EnableSSRefraction);
#endif
	
    // 高光部分，直接高光，反射探针。
    specularLighting = lighting.direct.specular + lighting.indirect.specularReflected;
    // Rescale the GGX to account for the multiple scattering.
    specularLighting *= 1.0 + bsdfData.fresnel0 * preLightData.energyCompensation;

}
```

## GetPreLightData

预光照信息，保存了能量守恒等各种系数，一次性计算在后面需要的时候一次性使用。

```c

PreLightData GetPreLightData(float3 V, PositionInputs posInput, inout BSDFData bsdfData)
{
    PreLightData preLightData;
    ZERO_INITIALIZE(PreLightData, preLightData);

    float3 N = bsdfData.normalWS;
    preLightData.NdotV = dot(N, V);
    preLightData.iblPerceptualRoughness = bsdfData.perceptualRoughness;

    float NdotV = ClampNdotV(preLightData.NdotV);

    // We modify the bsdfData.fresnel0 here for iridescence
    // 修改frenel0是为了得到彩虹色高光效果。
    if (HasFlag(bsdfData.materialFeatures, MATERIALFEATUREFLAGS_LIT_IRIDESCENCE))
    {
        float viewAngle = NdotV;
        float topIor = 1.0; // Default is air
        if (HasFlag(bsdfData.materialFeatures, MATERIALFEATUREFLAGS_LIT_CLEAR_COAT))
        {
            topIor = lerp(1.0, CLEAR_COAT_IOR, bsdfData.coatMask);
            // HACK: Use the reflected direction to specify the Fresnel coefficient for pre-convolved envmaps
            viewAngle = sqrt(1.0 + Sq(1.0 / topIor) * (Sq(dot(bsdfData.normalWS, V)) - 1.0));
        }

        if (bsdfData.iridescenceMask > 0.0)
        {
            bsdfData.fresnel0 = lerp(bsdfData.fresnel0, EvalIridescence(topIor, viewAngle, bsdfData.iridescenceThickness, bsdfData.fresnel0), bsdfData.iridescenceMask);
        }
    }
	
    // 修改fresnel0为了clearcoat效果
    // We modify the bsdfData.fresnel0 here for clearCoat
    if (HasFlag(bsdfData.materialFeatures, MATERIALFEATUREFLAGS_LIT_CLEAR_COAT))
    {
        // Fresnel0 is deduced from interface between air and material (Assume to be 1.5 in Unity, or a metal).
        // but here we go from clear coat (1.5) to material, we need to update fresnel0
        // Note: Schlick is a poor approximation of Fresnel when ieta is 1 (1.5 / 1.5), schlick target 1.4 to 2.2 IOR.
        bsdfData.fresnel0 = lerp(bsdfData.fresnel0, ConvertF0ForAirInterfaceToF0ForClearCoat15(bsdfData.fresnel0), bsdfData.coatMask);

        preLightData.coatPartLambdaV = GetSmithJointGGXPartLambdaV(NdotV, CLEAR_COAT_ROUGHNESS);
        preLightData.coatIblR = reflect(-V, N);
        preLightData.coatIblF = F_Schlick(CLEAR_COAT_F0, NdotV) * bsdfData.coatMask;
    }

    // Handle IBL + area light + multiscattering.
    // Note: use the not modified by anisotropy iblPerceptualRoughness here.
    float specularReflectivity;
    GetPreIntegratedFGDGGXAndDisneyDiffuse(NdotV, preLightData.iblPerceptualRoughness, bsdfData.fresnel0, preLightData.specularFGD, preLightData.diffuseFGD, specularReflectivity);
#ifdef USE_DIFFUSE_LAMBERT_BRDF
    preLightData.diffuseFGD = 1.0;
#endif

#ifdef LIT_USE_GGX_ENERGY_COMPENSATION
    // Ref: Practical multiple scattering compensation for microfacet models.
    // We only apply the formulation for metals.
    // For dielectrics, the change of reflectance is negligible.
    // We deem the intensity difference of a couple of percent for high values of roughness
    // to not be worth the cost of another precomputed table.
    // Note: this formulation bakes the BSDF non-symmetric!
    preLightData.energyCompensation = 1.0 / specularReflectivity - 1.0;
#else
    preLightData.energyCompensation = 0.0;
#endif // LIT_USE_GGX_ENERGY_COMPENSATION

    float3 iblN;

    // We avoid divergent evaluation of the GGX, as that nearly doubles the cost.
    // If the tile has anisotropy, all the pixels within the tile are evaluated as anisotropic.
    if (HasFlag(bsdfData.materialFeatures, MATERIALFEATUREFLAGS_LIT_ANISOTROPY))
    {
        float TdotV = dot(bsdfData.tangentWS,   V);
        float BdotV = dot(bsdfData.bitangentWS, V);

        preLightData.partLambdaV = GetSmithJointGGXAnisoPartLambdaV(TdotV, BdotV, NdotV, bsdfData.roughnessT, bsdfData.roughnessB);

        // perceptualRoughness is use as input and output here
        GetGGXAnisotropicModifiedNormalAndRoughness(bsdfData.bitangentWS, bsdfData.tangentWS, N, V, bsdfData.anisotropy, preLightData.iblPerceptualRoughness, iblN, preLightData.iblPerceptualRoughness);
    }
    else
    {
        preLightData.partLambdaV = GetSmithJointGGXPartLambdaV(NdotV, bsdfData.roughnessT);
        iblN = N;
    }

    preLightData.iblR = reflect(-V, iblN);

    // Area light
    // UVs for sampling the LUTs
    float theta = FastACosPos(NdotV); // For Area light - UVs for sampling the LUTs
    float2 uv = Remap01ToHalfTexelCoord(float2(bsdfData.perceptualRoughness, theta * INV_HALF_PI), LTC_LUT_SIZE);

    // Note we load the matrix transpose (avoid to have to transpose it in shader)
#ifdef USE_DIFFUSE_LAMBERT_BRDF
    preLightData.ltcTransformDiffuse = k_identity3x3;
#else
    // Get the inverse LTC matrix for Disney Diffuse
    preLightData.ltcTransformDiffuse      = 0.0;
    preLightData.ltcTransformDiffuse._m22 = 1.0;
    preLightData.ltcTransformDiffuse._m00_m02_m11_m20 = SAMPLE_TEXTURE2D_ARRAY_LOD(_LtcData, s_linear_clamp_sampler, uv, LTC_DISNEY_DIFFUSE_MATRIX_INDEX, 0);
#endif

    // Get the inverse LTC matrix for GGX
    // Note we load the matrix transpose (avoid to have to transpose it in shader)
    preLightData.ltcTransformSpecular      = 0.0;
    preLightData.ltcTransformSpecular._m22 = 1.0;
    preLightData.ltcTransformSpecular._m00_m02_m11_m20 = SAMPLE_TEXTURE2D_ARRAY_LOD(_LtcData, s_linear_clamp_sampler, uv, LTC_GGX_MATRIX_INDEX, 0);

    // Construct a right-handed view-dependent orthogonal basis around the normal
    preLightData.orthoBasisViewNormal = GetOrthoBasisViewNormal(V, N, preLightData.NdotV);

    preLightData.ltcTransformCoat = 0.0;
    if (HasFlag(bsdfData.materialFeatures, MATERIALFEATUREFLAGS_LIT_CLEAR_COAT))
    {
        float2 uv = LTC_LUT_OFFSET + LTC_LUT_SCALE * float2(CLEAR_COAT_PERCEPTUAL_ROUGHNESS, theta * INV_HALF_PI);

        // Get the inverse LTC matrix for GGX
        // Note we load the matrix transpose (avoid to have to transpose it in shader)
        preLightData.ltcTransformCoat._m22 = 1.0;
        preLightData.ltcTransformCoat._m00_m02_m11_m20 = SAMPLE_TEXTURE2D_ARRAY_LOD(_LtcData, s_linear_clamp_sampler, uv, LTC_GGX_MATRIX_INDEX, 0);
    }

    // refraction (forward only)
#if HAS_REFRACTION
    RefractionModelResult refraction = REFRACTION_MODEL(V, posInput, bsdfData);
    preLightData.transparentRefractV = refraction.rayWS;
    preLightData.transparentPositionWS = refraction.positionWS;
    preLightData.transparentTransmittance = exp(-bsdfData.absorptionCoefficient * refraction.dist);
    // Empirical remap to try to match a bit the refraction probe blurring for the fallback
    // Use IblPerceptualRoughness so we can handle approx of clear coat.
    preLightData.transparentSSMipLevel = PositivePow(preLightData.iblPerceptualRoughness, 1.3) * uint(max(_ColorPyramidScale.z - 1, 0));
#endif

    return preLightData;
}
```

