# HDRP HairShader代码结构

Hair只能在前向渲染阶段渲染，使用下面两个Pass：

```c
{
    LightMode = DepthForwardOnly
    LightMode = ForwardOnly
}
```

头发有一个特殊的Stencil

```c
        Stencil
        {
           WriteMask 7
           Ref  2
           Comp Always
           Pass Replace
        }
```

##透明和不透明有所区别

混合方式上的区别

```c
//透明
            Blend One OneMinusSrcAlpha 
            ZTest LEqual // 正常的深度渲染，注意这个时候也可以提前绘制深度。
            ZWrite Off
//不透明
            Blend One Zero  // 直接覆盖
            ZTest Equal  // 因为会提前绘制深度，所以需要和原本深度值相等才能绘制
            ZWrite On
// 在原本的shader当中如果开启blend默认编程了透明？
```

透明不透明的宏定义区别

```c
// 透明加不透明
#define _MATERIAL_FEATURE_HAIR_KAJIYA_KAY 1
#define _SURFACE_TYPE_TRANSPARENT 1
// 透明部分
#define _BLENDMODE_ALPHA 1   // 开启 混合
#define _BLENDMODE_PRESERVE_SPECULAR_LIGHTING 1  // ??
#define _ENABLE_FOG_ON_TRANSPARENT 1  // 开启透明物体上的雾
#define _SPECULAR_OCCLUSION_FROM_AO 1 // AO 对高光生效
```

下面是关于分割光线的 split Lighting 是什么

```c
// If we use subsurface scattering, enable output split lighting (for forward pass)
#if defined(_MATERIAL_FEATURE_SUBSURFACE_SCATTERING) && !defined(_SURFACE_TYPE_TRANSPARENT) // 子表面散射 并且不是半透明物体
#define OUTPUT_SPLIT_LIGHTING  // 分割光线？？？
#endif
```

## 头发渲染部分代码

### Shader Pass Forward

头发的Pass文件：

```c
#include "Packages/com.unity.render-pipelines.high-definition/Runtime/RenderPipeline/ShaderPass/ShaderPassForward.hlsl"
```

也就是说头发使用的事一般的前向渲染的Pass描述。

####Vert

```c
//这部分是通用的代码
PackedVaryingsType Vert(AttributesMesh inputMesh)
{
    VaryingsType varyingsType;
    varyingsType.vmesh = VertMesh(inputMesh);
    return PackVaryingsType(varyingsType);
}
```

```c
#include "Packages/com.unity.render-pipelines.high-definition/Runtime/RenderPipeline/ShaderPass/VertMesh.hlsl"
//使用的上面的内容用来计算Vert当中的参数。
```

####Frag

下面是去掉了debug信息的Frag代码：

如果使用了SSS效果并且没有使用半透则会进入OUTPUT_SPLIT_LIGHTING的部分，看起来是分开了高光和漫反射颜色，以及SSS的某个特殊buffer。

否则就是用一个单独的颜色。

这里有个_DEPTHOFFSET_ON，暂时不考虑。

```c
void Frag(PackedVaryingsToPS packedInput,
        #ifdef OUTPUT_SPLIT_LIGHTING  //使用分割光线
            out float4 outColor : SV_Target0,  // outSpecularLighting
            out float4 outDiffuseLighting : SV_Target1,
            OUTPUT_SSSBUFFER(outSSSBuffer)
        #else
            out float4 outColor : SV_Target0  // 没有SSS效果的一个输出颜色。
        #endif
        #ifdef _DEPTHOFFSET_ON
            , out float outputDepth : SV_Depth
        #endif
          )
{
    UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX(packedInput);
    FragInputs input = UnpackVaryingsMeshToFragInputs(packedInput.vmesh);

    uint2 tileIndex = uint2(input.positionSS.xy) / GetTileSize();
#if defined(UNITY_SINGLE_PASS_STEREO)
    tileIndex.x -= unity_StereoEyeIndex * _NumTileClusteredX;
#endif

    // input.positionSS is SV_Position
    PositionInputs posInput = GetPositionInput_Stereo(input.positionSS.xy, _ScreenSize.zw, input.positionSS.z, input.positionSS.w, input.positionRWS.xyz, tileIndex, unity_StereoEyeIndex); // 计算屏幕坐标 视口坐标 纹理坐标等内容（ComputeShader也会用这个函数）。

#ifdef VARYINGS_NEED_POSITION_WS
    float3 V = GetWorldSpaceNormalizeViewDir(input.positionRWS);
#else
    // Unused
    float3 V = float3(1.0, 1.0, 1.0); // Avoid the division by 0
#endif

    SurfaceData surfaceData;
    BuiltinData builtinData;
    
    // 纹理属性的计算
    GetSurfaceAndBuiltinData(input, V, posInput, surfaceData, builtinData);
	// 材质参数计算
    BSDFData bsdfData = ConvertSurfaceDataToBSDFData(input.positionSS.xy, surfaceData);

    PreLightData preLightData = GetPreLightData(V, posInput, bsdfData);
	// 定义输出颜色
    outColor = float4(0.0, 0.0, 0.0, 0.0);


    {
    //透明和不透明物体单独处理
#ifdef _SURFACE_TYPE_TRANSPARENT
        uint featureFlags = LIGHT_FEATURE_MASK_FLAGS_TRANSPARENT;
#else
        uint featureFlags = LIGHT_FEATURE_MASK_FLAGS_OPAQUE;
#endif
	// 将要分别计算的高光和漫反射光
        float3 diffuseLighting;
        float3 specularLighting;
	
	// 进入光照循环计算光照
        LightLoop(V, posInput, preLightData, bsdfData, builtinData, featureFlags, diffuseLighting, specularLighting);
	// 此时就返回了计算好的高光和 漫反射光
	// 进行全局的缩放
        diffuseLighting *= GetCurrentExposureMultiplier();
        specularLighting *= GetCurrentExposureMultiplier();
	
#ifdef OUTPUT_SPLIT_LIGHTING
	// 如果开启SSS 并且需要分割光照（通过参数判断），则把两个光照分开。
        if (_EnableSubsurfaceScattering != 0 && ShouldOutputSplitLighting(bsdfData))
        {
            outColor = float4(specularLighting, 1.0);
            outDiffuseLighting = float4(TagLightingForSSS(diffuseLighting), 1.0);
        }
        else
        {
        	//如果不需要SSS，或者不需要分割光照，就把两个颜色都合并到Specular里面。
            outColor = float4(diffuseLighting + specularLighting, 1.0);
            outDiffuseLighting = 0;
        }
        // 这里面写入一些 SSS需要用到的属性。
        ENCODE_INTO_SSSBUFFER(surfaceData, posInput.positionSS, outSSSBuffer);
        // 如果分割颜色就，在Gbuffer结束时统一计算大气散射。
#else
        outColor = ApplyBlendMode(diffuseLighting, specularLighting, builtinData.opacity);
        
        // 如果不分割颜色 就直接计算大气散射。
        outColor = EvaluateAtmosphericScattering(posInput, V, outColor);
#endif
    }

// 忽略
#ifdef _DEPTHOFFSET_ON
    outputDepth = posInput.deviceDepth;
#endif
}
```

#### LightLoop

具体Shader的光照循环部分，参见光照循环文档