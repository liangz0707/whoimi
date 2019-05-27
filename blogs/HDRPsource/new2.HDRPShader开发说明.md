## Shader开发说明

### 文件结构

**CoreRPLibrary/ShaderLibrary:**

保存了大量的工具函数：光照计算工具函数，随机数计算，矩阵工具，坐标转换工具，风场，ParallaxOcclusionMapping等。

**CoreRPLibrary/ShaderLibrary/API:**

保存了跨平台函数的定义。

**HDRP/Runtime/Material:**

保存了HDRP中默认支持的材质Shader：Lit，LayeredLit，Stacklit等，都是和各自材质相关的计算。不同的材质中包含了不同的BSDF函数的实现、不同的BuiltinData的组织方式。

**HDRP/Runtime/PostProcessing/Shaders:**

后处理的Shader，HDRP中后处理全部使用ComputeShader。

**HDRP/Runtime/RenderPipeline/ShaderPass:**

ShaderPass的定义：包括了Vertex和Fragment程序的定义。

**HDRP/Runtime/RenderPipeline/ShaderLibrary:**

从C#当中设置的Shader参数，包括：各种变换矩阵、获取矩阵的函数、摄像机参数、场景参数、全部buffer、全局纹理、shader控制参数等。

### Shader Pass

RenderPiple的代码中，在不同的时机会渲染不同的ShaderPass（通过lightmode区分）。**如果要看更详细的Pass绘制时机，以及Pass之间如何组合成正确的Shader，需要看Pipeline的代码，如果随意组合会得到无法预知的结果**，主要的Pass如下：

#### Forward

前向渲染物体使用这个Pass，正常情况下使用Deferred。透明物体可以使用Forward Pass， StackLit也是Forward Pass。

#### ForwardOnly

ForwardOnly的用处是：渲染透明物体或**在Deferred模式下强制使用Forward模式渲染不透明物体**。 例如：StackLit就是使用了{ForwardOnly和DepthForwardOnly}组合的Shader。

#### DepthForwardOnly

和ForwardOnly对应使用。

#### DepthOnly

渲染深度，Forward和Deferred必须有一个深度，使用这个Pass。

#### TransparentDepthPrepass

透明物体Prepass深度

#### TransparentDepthPostpass

透明物体Postpass深度

**注意：**上面所有的Pass需要正确的组合在一个Shader当中，不然会出错。例如：自己写的Shader中可以包括：{ForwardOnly，DepthForwardOnly}。 如果出现：{ForwardOnly，DepthOnly}就会出现不可预知的结果。

#### ShadowCaster

用于渲染阴影的Pass，和老版本不同的是：HDRP的阴影和深度使用了两个不同的Pass。

#### DistortionVectors

扭曲向量

#### DistortionVectors

屏幕运动向量

### Shader编写

#### 定义LightMode和ShaderPass

如果要实现自己的Shader，第一步需要正确的定义ShaderPass：

不透明物体:主要需要基本的光照渲染Forward或者Deferred、阴影ShadowCaster，深度DepthOnly：

```c
Shader "HDRP"
{

    SubShader
    {
        Pass
        {
            Name "Forward"
            Tags{ "LightMode" = "Forward" }
        }
         Pass
        {
            Name "Deferred"
            Tags{ "LightMode" = "Deferred" }
        }
        Pass
        {
            Name "DepthOnly"
            Tags{ "LightMode" = "DepthOnly" }
        }
        Pass
        {
            Name "ShadowCaster"
            Tags{ "LightMode" = "ShadowCaster" }
        }
    }
}
```

不透明物体也可以是（类似StackLit方式）：

```c
Shader "HDRP"
{
    SubShader
    {
        Pass
        {
            Name "ForwardOnly"
            Tags{ "LightMode" = "ForwardOnly" }
        }
        Pass
        {
            Name "DepthForwardOnly"
            Tags{ "LightMode" = "DepthForwardOnly" }
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment Frag
            ENDHLSL
        }
    }
}
```

透明物体,需要基本的光照Pass（也可以不计算光照）ForwardOnly或者Forward，深度TransparentDepthPrepass、TransparentDepthPostpass

```c
Shader "HDRP"
{

    SubShader
    {
        Pass
        {
            Name "ForwardOnly"
            Tags{ "LightMode" = "Forward" }
            HLSLPROGRAM
            #pragma vertex Vert
            #pragma fragment Frag
            ENDHLSL
        }
        Pass
        {
            Name "TransparentDepthPrepass"
            Tags{ "LightMode" = "TransparentDepthPrepass" }
        }
        Pass
        {
            Name "TransparentDepthPostpass"
            Tags{ "LightMode" = "TransparentDepthPostpass" }
        }
    }
}
```

可以根据自己的需求组合不同的Pass达到不同的效果。例如：一个物体如果不想要阴影，可以直接去掉ShadowCaster Pass，这样对深度没有任何影响。**还有一个重要的问题就是：所有ShaderGraph或者HDRP内置的Shader都包括了ShadowCaster Pass和Depth Pass。所以，需要之后应该需要我们根据需求手动去除这个Pass。**

#### 设置Queue

控制渲染时机的除了LightMode之外，另一个是Queue。

HDRP的Queue和原本不同，现在只能定义到SubShader级别.

```
Shader ""
{
     SubShader
     {
        Tags { "Queue" = "Transparent" }
        Pass
        {
            // rest of the shader body...
        }
    }
}
```

HDRP使用了Priority定义Queue，具体参考HDMaterialTags.cs文件。

完成了Queue和LightMode的设置就算是完成一个完整的Shader设置。

#### Material参数

HDRP的自定义参数设置和原始版本的使用方式是一样的。

不同的是内置参数。HDRP的内置参数全部通过可查看的C#代码设置。需要包括以下两个文件。

```c
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl"
#include "Packages/com.unity.render-pipelines.high-definition/Runtime/ShaderLibrary/ShaderVariables.hlsl"

```

这两个文件包括了：版本相关的变量关键字、摄像机矩阵、场景参数、摄像机参数等内容。矩阵类型和原本也有所差异。

#### 灯光设置

HDRP和原始版本最大的区别就是灯光计算，HDPR在一个Pass当中会计算完所有的灯光（包括区域光、环境光、雾、LightMap、LightProbe）。需要从LightList当中读取各种信息。

下面是读取灯光信息的基本形式：

```c
#include "CustomLighting.hlsl"

void LightLoop(...)
{

    float NdotV = dot(bsdfData.normalWS, V);
    LightLoopContext context;
    context.shadowContext    = InitShadowContext();
    // 计算基于屏幕的细节阴影
    context.contactShadow    = InitContactShadow(posInput);
    context.shadowValue      = 1;
    context.sampleReflection = 0;

    uint lightCount, lightStart;

    // ===================  读取点光、聚光灯  ===================== 
    /*
    struct LightData
    {
        float3 positionRWS;
        uint lightLayers;
        float lightDimmer;
        float volumetricLightDimmer;
        float angleScale;
        float angleOffset;
        float3 forward;
        int lightType;
        float3 right;
        float range;
        float3 up;
        float rangeAttenuationScale;
        float3 color;
        float rangeAttenuationBias;
        int cookieIndex;
        int tileCookie;
        int shadowIndex;
        int contactShadowMask;
        int rayTracedAreaShadowIndex;
        float shadowDimmer;
        float volumetricShadowDimmer;
        int nonLightMappedOnly;
        float minRoughness;
        float4 shadowMaskSelector;
        float2 size;
        float diffuseDimmer;
        float specularDimmer;
    };
    */
    /*
    根据世界坐标位置和灯光类型，返回灯光索引的开始索引和数量.
    */
    GetCountAndStart(posInput, LIGHTCATEGORY_PUNCTUAL, lightStart, lightCount);
    uint i = 0; 
    for (i = 0; i < lightCount; ++i)
    {
        LightData lightData = FetchLight(lightStart, i);
        // 计算光照
    }
    
    // ===================  读取直线光  =====================
    /*
    _DirectionalLightCount：直线光个数
    _DirectionalLightDatas：所有直线光数组.
    直线光保存在CB当中，不参与FPTL,所以直接通过数组计算。
    _DirectionalLightDatas的数据结构：
    struct DirectionalLightData
    {
        float3 positionRWS;
        uint lightLayers;
        float lightDimmer;
        float volumetricLightDimmer;
        float angleScale;
        float angleOffset;
        float3 forward;
        int cookieIndex;
        float3 right;
        int tileCookie;
        float3 up;
        int shadowIndex;
        float3 color;
        int contactShadowMask;
        float shadowDimmer;
        float volumetricShadowDimmer;
        int nonLightMappedOnly;
        float minRoughness;
        float4 shadowMaskSelector;
        float diffuseDimmer;
        float specularDimmer;
    };
    */
    uint i = 0; 
    for (i = 0; i < _DirectionalLightCount; ++i)
    {
        _DirectionalLightDatas[i];
    }
    
    
    // ====================== 读取区域光 =====================
    /*
   	LightData定义同读取点光、聚光灯。
    */
    /*
     在Lit当中Loop当中可以看到用了两个while，主要是为了GPU优化。
    */
    GetCountAndStart(posInput, LIGHTCATEGORY_AREA, lightStart, lightCount);
    for(i = 0; i < lightCount; i++)
    {
        LightData lightData = FetchLight(lightStart, i);
       	// 计算光照
    }
    
    // ==================== 读取环境光，主要处理反射和折射光的分量 =====================
    /*
    这里有个重要的内容，就是反射和折射信息和光照计算不同，不是叠加，而是反射的上限应该是1。所以具体如何处理不同的反射和折射的关系需要参考Lit。
    */
    /*
    struct EnvLightData
    {
        uint lightLayers;
        float3 capturePositionRWS;
        int influenceShapeType;
        float3 proxyExtents;
        float minProjectionDistance;
        float3 proxyPositionRWS;
        float3 proxyForward;
        float3 proxyUp;
        float3 proxyRight;
        float3 influencePositionRWS;
        float3 influenceForward;
        float3 influenceUp;
        float3 influenceRight;
        float3 influenceExtents;
        float unused00;
        float3 blendDistancePositive;
        float3 blendDistanceNegative;
        float3 blendNormalDistancePositive;
        float3 blendNormalDistanceNegative;
        float3 boxSideFadePositive;
        float3 boxSideFadeNegative;
        float weight;
        float multiplier;
        int envIndex;
    };
    */
    //Lit当中首先定义了反射和折射的成分，初始化为0.
    float reflectionHierarchyWeight = 0.0; // Max: 1.0
    float refractionHierarchyWeight = _EnableSSRefraction ? 0.0 : 1.0; // Max: 1.0
    // 读取环境光照信息
    GetCountAndStart(posInput, LIGHTCATEGORY_ENV, envLightStart, envLightCount);
    
    // 先计算屏幕空间反射：叠加reflectionHierarchyWeight。这部分信息不再Tile当中
    {
        IndirectLighting indirect = EvaluateBSDF_ScreenSpaceReflection(posInput, preLightData, bsdfData,                                     reflectionHierarchyWeight);
        AccumulateIndirectLighting(indirect, aggregateLighting);
    }
    
    // 计算屏幕折射。
    if ((featureFlags & LIGHTFEATUREFLAGS_SSREFRACTION) && (_EnableSSRefraction > 0))
    {
        // 折射信息
        envLightData = FetchEnvLight(envLightStart, 0);
    }
        
    // 反射和折射探针。
    if (featureFlags & LIGHTFEATUREFLAGS_ENV)
    {
        for(i = 0;i<envLightCount;i++)
        {
            uint v_envLightIdx = FetchIndex(envLightStart, i);
            EnvLightData s_envLightData = FetchEnvLight(v_envLightIdx);    
   			// 处理环境光
        }

        // 使用天空纹理计算IBL
        if ((featureFlags & LIGHTFEATUREFLAGS_SKY) && _EnvLightSkyEnabled)
        {
            context.sampleReflection = SINGLE_PASS_CONTEXT_SAMPLE_SKY;
            EnvLightData envLightSky = InitSkyEnvLightData(0);
			// 处理envLightSky 
        }
    }
}
```

#### 重要函数

##### 需要通过Tile读取的灯光

#define LIGHTCATEGORY_PUNCTUAL (0)
#define LIGHTCATEGORY_AREA (1)
#define LIGHTCATEGORY_ENV (2)

##### FetchLight

灯光就是直接从Buffer当中读取的，但是索引是要通过FPTL、Cluster、Big-Tile等方法读取的，下面是直接读取灯光信息：

```c
LightData FetchLight(uint start, uint i)
{
    uint j = FetchIndex(start, i);

    return _LightDatas[j];
}

LightData FetchLight(uint index)
{
    return _LightDatas[index];
}

EnvLightData FetchEnvLight(uint start, uint i)
{
    int j = FetchIndex(start, i);

    return _EnvLightDatas[j];
}

EnvLightData FetchEnvLight(uint index)
{
    return _EnvLightDatas[index];
}
```

##### FetchIndex

具体的索引如何读取，就要看是否使用FPTL、Cluster、Big-Tile或者不用光照优化策略：

```c
#ifdef USE_FPTL_LIGHTLIST    // 使用FPTL
uint FetchIndex(uint tileOffset, uint lightOffset)
{
    const uint lightOffsetPlusOne = lightOffset + 1; // Add +1 as first slot is reserved to store number of light
    // Light index are store on 16bit
    return (g_vLightListGlobal[DWORD_PER_TILE * tileOffset + (lightOffsetPlusOne >> 1)] >> ((lightOffsetPlusOne & 1) * DWORD_PER_TILE)) & 0xffff;
}

#elif defined(USE_CLUSTERED_LIGHTLIST) // 使用Cluster  例如：对于透明物体就用Cluster

uint FetchIndex(uint lightStart, uint lightOffset)
{
    return g_vLightListGlobal[lightStart + lightOffset];
}

#elif defined(USE_BIG_TILE_LIGHTLIST) // 使用BigTile

uint FetchIndex(uint lightStart, uint lightOffset)
{
    return g_vBigTileLightList[lightStart + lightOffset];
}

#else             // 没有
// Fallback case (mainly for raytracing right now)
uint FetchIndex(uint lightStart, uint lightOffset)
{
    return 0;
}
#endif // USE_FPTL_LIGHTLIST
```

##### GetCountAndStart

获取灯光列表的起始索引和数量，同上。具体查看LightLoopDef.hlsl源文件。

