### HairShader光照模型

创建ShaderGraph:

定义的内容

```c
#define SHADERPASS SHADERPASS_FORWARD
#pragma multi_compile _ DEBUG_DISPLAY
#pragma multi_compile _ LIGHTMAP_ON
#pragma multi_compile _ DIRLIGHTMAP_COMBINED
#pragma multi_compile _ DYNAMICLIGHTMAP_ON
#pragma multi_compile _ SHADOWS_SHADOWMASK
#pragma multi_compile DECALS_OFF DECALS_3RT DECALS_4RT
#pragma multi_compile USE_FPTL_LIGHTLIST USE_CLUSTERED_LIGHTLIST
#pragma multi_compile SHADOW_LOW SHADOW_MEDIUM SHADOW_HIGH SHADOW_VERY_HIGH
```

include文件

```c
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl"
        
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/NormalSurfaceGradient.hlsl"
        
#include "Packages/com.unity.render-pipelines.high-definition/Runtime/RenderPipeline/ShaderPass/FragInputs.hlsl"

#include "Packages/com.unity.render-pipelines.high-definition/Runtime/RenderPipeline/ShaderPass/ShaderPass.cs.hlsl"

#include "Packages/com.unity.render-pipelines.high-definition/Runtime/ShaderLibrary/ShaderVariables.hlsl"


#include "Packages/com.unity.render-pipelines.high-definition/Runtime/Material/Material.hlsl"

// 这个在前面定义了
#if (SHADERPASS == SHADERPASS_FORWARD)
#include "Packages/com.unity.render-pipelines.high-definition/Runtime/Lighting/Lighting.hlsl"

    #define HAS_LIGHTLOOP

    #include "Packages/com.unity.render-pipelines.high-definition/Runtime/Lighting/LightLoop/LightLoopDef.hlsl"
    #include "Packages/com.unity.render-pipelines.high-definition/Runtime/Material/Hair/Hair.hlsl"
    #include "Packages/com.unity.render-pipelines.high-definition/Runtime/Lighting/LightLoop/LightLoop.hlsl"
#else
#include "Packages/com.unity.render-pipelines.high-definition/Runtime/Material/Hair/Hair.hlsl"
#endif

#include "Packages/com.unity.render-pipelines.high-definition/Runtime/Material/BuiltinUtilities.hlsl"
#include "Packages/com.unity.render-pipelines.high-definition/Runtime/Material/MaterialUtilities.hlsl"
#include "Packages/com.unity.render-pipelines.high-definition/Runtime/Material/Decal/DecalUtilities.hlsl"
#include "Packages/com.unity.render-pipelines.high-definition/Runtime/ShaderLibrary/ShaderGraphFunctions.hlsl"
```

和头发表面相关的参数，这个函数主要是ShaderGraph的输入

```c
SurfaceDescription SurfaceDescriptionFunction(SurfaceDescriptionInputs IN)
{
    SurfaceDescription surface = (SurfaceDescription)0;
    surface.Albedo = float3(0.7353569, 0.7353569, 0.7353569);
    surface.Normal = IN.TangentSpaceNormal;
    surface.BentNormal = IN.TangentSpaceNormal;
    surface.HairStrandDirection = float3 (0,-1,0); //头发走向
    surface.Occlusion = 1;
    surface.Alpha = 1;
    
    // 用于就算高光的内容：高光的颜色，粗糙的，高光的Shift
    surface.SpecularTint = float3(1, 1, 1);
    surface.Smoothness = 0.5;
    surface.SpecularShift = 0.1;
    
    surface.SecondarySpecularTint =  float3(0.5, 0.5, 0.5) ;
    surface.SecondarySmoothness = 0.5;
    surface.SecondarySpecularShift = -0.1;
    return surface;
}
```

上面的的函数在GetSurfaceAndBuiltinData函数当中调用。

```c
void GetSurfaceAndBuiltinData(FragInputs fragInputs, inout SurfaceDescription surfaceDescription, float3 V, PositionInputs posInput, out SurfaceData surfaceData, out float3 bentNormalWS)
{
    ...
	
    // 双面处理
    float3 doubleSidedConstants = float3(1.0, 1.0, 1.0);
    ApplyDoubleSidedFlipOrMirror(fragInputs, doubleSidedConstants);
	
    // 基本的Normal 切线之类的
    SurfaceDescriptionInputs surfaceDescriptionInputs = FragInputsToSurfaceDescriptionInputs(fragInputs, V);
    
    // 表面信息 高光 粗糙度 从Graph当中读取
    SurfaceDescription surfaceDescription = SurfaceDescriptionFunction(surfaceDescriptionInputs);

    // 将信息写入到 需要计算的两个结构体当中
    float3 bentNormalWS;
    BuildSurfaceData(fragInputs, surfaceDescription, V, posInput, surfaceData, bentNormalWS);

    // Builtin Data
    // For back lighting we use the oposite vertex normal  背面的光使用相反的法线，主要用来采样GI？
    InitBuiltinData(surfaceDescription.Alpha, bentNormalWS, -fragInputs.worldToTangent[2]/*这部分内容是反向法线*/, fragInputs.positionRWS, fragInputs.texCoord1, fragInputs.texCoord2, builtinData);
        

    PostInitBuiltinData(V, posInput, surfaceData, builtinData);
}
```



BuilSurfaceData()

```c
void BuildSurfaceData(FragInputs fragInputs, inout SurfaceDescription surfaceDescription, float3 V, PositionInputs posInput, out SurfaceData surfaceData, out float3 bentNormalWS)
{
   	// 初始化
    ZERO_INITIALIZE(SurfaceData, surfaceData);

    // copy across graph values, if defined 直接复制ShaderGraph的内容
    surfaceData.diffuseColor =                  surfaceDescription.Albedo;
    surfaceData.ambientOcclusion =              surfaceDescription.Occlusion; // AO

    // 两组高光相关的内容
    surfaceData.perceptualSmoothness =          surfaceDescription.Smoothness; // 粗糙度 
    surfaceData.specularTint =                  surfaceDescription.SpecularTint;
    surfaceData.specularShift =                 surfaceDescription.SpecularShift;

    surfaceData.secondaryPerceptualSmoothness = surfaceDescription.SecondarySmoothness;
    surfaceData.secondarySpecularTint =         surfaceDescription.SecondarySpecularTint;
    surfaceData.secondarySpecularShift =        surfaceDescription.SecondarySpecularShift;

    // These static material feature allow compile time optimization
    surfaceData.materialFeatures = 0;

    // 材质特性：头发的透光以及头发的 SSS效果
    #ifdef _MATERIAL_FEATURE_HAIR_KAJIYA_KAY
    surfaceData.materialFeatures = MATERIALFEATUREFLAGS_HAIR_KAJIYA_KAY;
    #endif

    #ifdef _MATERIAL_FEATURE_SUBSURFACE_SCATTERING
    surfaceData.materialFeatures |= MATERIALFEATUREFLAGS_HAIR_SUBSURFACE_SCATTERING;
    #endif

    #ifdef _MATERIAL_FEATURE_TRANSMISSION
    surfaceData.materialFeatures |= MATERIALFEATUREFLAGS_HAIR_TRANSMISSION;
    #endif
    

    // 法线 切线
    float3 doubleSidedConstants = float3(1.0, 1.0, 1.0);
    // tangent-space normal
    float3 normalTS = float3(0.0f, 0.0f, 1.0f);
    normalTS = surfaceDescription.Normal;
    // compute world space normal
    GetNormalWS(fragInputs, normalTS, surfaceData.normalWS, doubleSidedConstants);
    bentNormalWS = surfaceData.normalWS;
    surfaceData.geomNormalWS = fragInputs.worldToTangent[2];

    // For a typical Unity quad, you have tangent vectors pointing to the right (X axis),
    // and bitangent vectors pointing up (Y axis).
    // The current hair setup uses mesh cards (e.g. quads).
    // Hair is usually painted top-down, from the root to the tip.
    // Therefore, DefaultHairStrandTangent = -MeshCardBitangent. // <==这句话关键
    // Both the SurfaceData and the BSDFData store the hair tangent
    // (which represents the hair strand direction, root to tip). // 头发的方向
    surfaceData.hairStrandDirectionWS = -fragInputs.worldToTangent[1].xyz;
    // The hair strand direction texture contains tangent-space vectors.
    // We use the same convention for the texture, which means that
    // to get the default behavior (DefaultHairStrandTangent = -MeshCardBitangent),
    // the artist has to paint (0, -1, 0).
    // TODO: pending artist feedback...
    // The original Kajiya-Kay BRDF model expects an orthonormal TN frame.
    // Since we use the tangent shift hack 
    
    // 这个文档需要看一下
    // (http://web.engr.oregonstate.edu/~mjb/cs519/Projects/Papers/HairRendering.pdf),
    // we may as well not bother to orthonormalize anymore.
    // The tangent should still be a unit vector, though.
    surfaceData.hairStrandDirectionWS = normalize(surfaceData.hairStrandDirectionWS);

    // By default we use the ambient occlusion with Tri-ace trick (apply outside) for specular occlusion.
    // If user provide bent normal then we process a better term
    surfaceData.specularOcclusion = 1.0;
	
    // 下面有一部分是计算 specularOcclusion的
	..........
        
    // 下面是计算GAA的，后面需要看一下

    // Propagate the geometry normal
    surfaceData.geomNormalWS = fragInputs.worldToTangent[2];
}
```

其他部分的计算最终都会归结到BSDF函数

```
void BSDF(  float3 V, float3 L, float NdotL, float3 positionWS, PreLightData preLightData, BSDFData bsdfData,
            out float3 diffuseLighting,
            out float3 specularLighting)
{
	//preLightData	当中比较重要的就用到了NdotV
    float LdotV, NdotH, LdotH, NdotV, invLenLV;
    GetBSDFAngle(V, L, NdotL, preLightData.NdotV, LdotV, NdotH, LdotH, NdotV, invLenLV);

    if (HasFlag(bsdfData.materialFeatures, MATERIALFEATUREFLAGS_HAIR_KAJIYA_KAY))
    {
    	// 颜色以高光为主
        float3 t1 = ShiftTangent(bsdfData.hairStrandDirectionWS, bsdfData.normalWS, bsdfData.specularShift);
        float3 t2 = ShiftTangent(bsdfData.hairStrandDirectionWS, bsdfData.normalWS, bsdfData.secondarySpecularShift);

        float3 H = (L + V) * invLenLV;

        float3 hairSpec1 = bsdfData.specularTint * D_KajiyaKay(t1, H, bsdfData.specularExponent);
        float3 hairSpec2 = bsdfData.secondarySpecularTint * D_KajiyaKay(t2, H, bsdfData.secondarySpecularExponent);

        float3 F = F_Schlick(bsdfData.fresnel0, LdotH);
        specularLighting = F * (hairSpec1 + hairSpec2);

        // Diffuse lighting
        // #define INV_PI      0.31830988618379067154
        float diffuseTerm = Lambert();  // 漫反射项的强度
        diffuseLighting = diffuseTerm;
    }
    else
    {
        specularLighting = float3(0.0, 0.0, 0.0);
        diffuseLighting = float3(0.0, 0.0, 0.0);
    }
}
```

高光

```c
float3 D_KajiyaKay(float3 T, float3 H, float specularExponent)
{
    float TdotH = dot(T, H);
    float sinTHSq = saturate(1.0 - (TdotH * TdotH));

    float dirAttn = saturate(TdotH + 1.0);

    return dirAttn * PositivePow(sinTHSq, specularExponent);
}
```

和BSDF相关的内容

```c
// return usual BSDF angle
void GetBSDFAngle(float3 V, float3 L, float NdotL, float unclampNdotV, out float LdotV, out float NdotH, out float LdotH, out float clampNdotV, out float invLenLV)
{
    // Optimized math. Ref: PBR Diffuse Lighting for GGX + Smith Microsurfaces (slide 114).
    LdotV = dot(L, V);
    invLenLV = rsqrt(max(2.0 * LdotV + 2.0, FLT_EPS));    // invLenLV = rcp(length(L + V)), clamp to avoid rsqrt(0) = inf, inf * 0 = NaN
    NdotH = saturate((NdotL + unclampNdotV) * invLenLV);        // Do not clamp NdotV here
    LdotH = saturate(invLenLV * LdotV + invLenLV);
    clampNdotV = ClampNdotV(unclampNdotV);
}
```

```c
// 平滑度到粗糙度
real PerceptualSmoothnessToPerceptualRoughness(real perceptualSmoothness)
{
    return (1.0 - perceptualSmoothness);
}
```

```c
real PerceptualRoughnessToRoughness(real perceptualRoughness)
{
    return perceptualRoughness * perceptualRoughness;
}
```

Bsdf数据当中需要的内容“

```c

BSDFData ConvertSurfaceDataToBSDFData(uint2 positionSS, SurfaceData surfaceData)
{
    BSDFData bsdfData;
    ZERO_INITIALIZE(BSDFData, bsdfData);

    // IMPORTANT: All enable flags are statically know at compile time, so the compiler can do compile time optimization
    bsdfData.materialFeatures = surfaceData.materialFeatures;

    bsdfData.ambientOcclusion = surfaceData.ambientOcclusion;
    bsdfData.specularOcclusion = surfaceData.specularOcclusion;

    bsdfData.diffuseColor = surfaceData.diffuseColor;

    bsdfData.normalWS = surfaceData.normalWS;
    bsdfData.geomNormalWS = surfaceData.geomNormalWS;
    bsdfData.perceptualRoughness = PerceptualSmoothnessToPerceptualRoughness(surfaceData.perceptualSmoothness);

    // This value will be override by the value in diffusion profile
    bsdfData.fresnel0 = DEFAULT_HAIR_SPECULAR_VALUE;

    // Note: we have ZERO_INITIALIZE the struct so bsdfData.anisotropy == 0.0
    // Note: DIFFUSION_PROFILE_NEUTRAL_ID is 0
    
    bsdfData.diffusionProfileIndex = FindDiffusionProfileIndex(surfaceData.diffusionProfileHash);

    if (HasFlag(surfaceData.materialFeatures, MATERIALFEATUREFLAGS_HAIR_SUBSURFACE_SCATTERING))
    {
        // Assign profile id and overwrite fresnel0
        FillMaterialSSS(bsdfData.diffusionProfileIndex, surfaceData.subsurfaceMask, bsdfData);
    }

    if (HasFlag(surfaceData.materialFeatures, MATERIALFEATUREFLAGS_HAIR_TRANSMISSION))
    {
        // Assign profile id and overwrite fresnel0
        FillMaterialTransmission(bsdfData.diffusionProfileIndex, surfaceData.thickness, bsdfData);
    }

    // This is the hair tangent (which represents the hair strand direction, root to tip).
    bsdfData.hairStrandDirectionWS = surfaceData.hairStrandDirectionWS;

    // Kajiya kay
    if (HasFlag(surfaceData.materialFeatures, MATERIALFEATUREFLAGS_HAIR_KAJIYA_KAY))
    {
        bsdfData.secondaryPerceptualRoughness = PerceptualSmoothnessToPerceptualRoughness(surfaceData.secondaryPerceptualSmoothness);        
        bsdfData.specularTint = surfaceData.specularTint;
        bsdfData.secondarySpecularTint = surfaceData.secondarySpecularTint;
        bsdfData.specularShift = surfaceData.specularShift;
        bsdfData.secondarySpecularShift = surfaceData.secondarySpecularShift;

        // We can rewrite specExp from exp2(10 * (1.0 - roughness)) in order
        // to remove the need to take the square root of sinTH
        // 这里是通过粗糙度计算曝光信息
        bsdfData.specularExponent = exp2(9.0 - 10.0 * PerceptualRoughnessToRoughness(bsdfData.perceptualRoughness));
        bsdfData.secondarySpecularExponent = exp2(9.0 - 10.0 * PerceptualRoughnessToRoughness(bsdfData.secondaryPerceptualRoughness));

        bsdfData.anisotropy = 0.8; // For hair we fix the anisotropy
    }

    ApplyDebugToBSDFData(bsdfData);

    return bsdfData;
}
```





















