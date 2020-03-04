# UE4Shader代码初探

## BasePassPixelShader.usf

**BasePassPixelShader.usf**是理解Unreal Shader最好的切入点。

下面是Unreal官方文档[shader开发](https://docs.unrealengine.com/zh-CN/Programming/Rendering/ShaderDevelopment/index.html)给出的简单说明：

> 典型的材质像素着色器类型将先通过调用 **GetMaterialPixelParameters** 顶点工厂函数来创建 **FMaterialPixelParameters** 构造。GetMaterialPixelParameters 将特定于顶点工厂的输入转换为任何过程可能想访问的属性，例如 WorldPosition 和 TangentNormal 等等。然后，材质着色器将调用 **CalcMaterialParameters**，后者将写出 `FMaterialPixelParameters` 的其余成员，之后 `FMaterialPixelParameters` 完全初始化。然后，材质着色器将通过 MaterialTemplate.usf 中的函数来访问该材质的某些输入（例如，通过 **GetMaterialEmissive** 访问材质的自发光输入），执行一些明暗处理，然后输出该过程的最终颜色。

上文中出现的函数和结构体均可以在**BasePassPixelShader.usf**文件当中找到对应的位置。而提到的**MaterialTemplate.usf** 文件则可以查看各种函数的实现。

```c
GetMaterialPixelParameters
FMaterialPixelParameters
CalcMaterialParameters
GetMaterialEmissive
```

首先观察函数**FPixelShaderInOut_MainPS**

这里主要是在对Gbuffer(FPixelShaderOut)进行填充。

```c
// is called in MainPS() from PixelShaderOutputCommon.usf
void FPixelShaderInOut_MainPS(FVertexFactoryInterpolantsVSToPS Interpolants,
FBasePassInterpolantsVSToPS BasePassInterpolants,
in FPixelShaderIn In, 
inout FPixelShaderOut Out)
{
	//像素着色器的主要计算过程
}
```

## PixelShaderOutputCommon.ush

根据上一段代码的注释可以延展到这个文件：PixelShaderOutputCommon.ush

文件的内容很少，就是一个通用的Shader文件，当中包括了所有Pixel的公用入口：

```c
void MainPS
(...)
{
    
}
```

**里面大量的宏控制了输入的输出的结构体类型**。

## MaterialTemplate.usf

从这个文件名字可以看出这是定义材质模板的头文件。

包括了VS-PS结构体的定义：

```c
struct FMaterialPixelParameters
{
    ...
	/** Interpolated vertex color, in linear color space. */
	half4 VertexColor;

	/** Normalized world space normal. */
	half3 WorldNormal;
	... ...
};
```

定义了重要的空间转换函数：

```c
float3 GetTranslatedWorldPosition_NoMaterialOffsets(FMaterialPixelParameters Parameters)
float4 GetScreenPosition(FMaterialVertexParameters Parameters)
float4 GetScreenPosition(FMaterialPixelParameters Parameters)
float2 GetSceneTextureUV(FMaterialVertexParameters Parameters)

```

各种材质函数：

```c
MaterialFloat2 GetLightmapUVs(FMaterialPixelParameters Parameters)
half3 GetMaterialNormal(FMaterialPixelParameters Parameters, FPixelMaterialInputs PixelMaterialInputs)
half3 GetMaterialEmissive(FPixelMaterialInputs PixelMaterialInputs)
half GetMaterialMetallic(FPixelMaterialInputs PixelMaterialInputs)
```

从函数和变量名可以看到代码很好理解。

## TiledDeferredLightShaders.usf

Unreal默认时使用Deferred渲染，上面的三个文件都是Geometry阶段的代码。而这个文件就是LIght阶段的代码。

Unreal使用TiledDeferredLight策略，和Unity HDRP的光照策略接近。 

入口函数，可以看到这个和Unity的HDRP一样使用了ComputeShader来计算：

```c
[numthreads(THREADGROUP_SIZEX, THREADGROUP_SIZEY, 1)]
void TiledDeferredLightingMain(
	uint3 GroupId : SV_GroupID,
	uint3 DispatchThreadId : SV_DispatchThreadID,
    uint3 GroupThreadId : SV_GroupThreadID) 
{
    ...
}
```

除了线程同步之外的操作，关键的就是TIle的灯光列表的获取，**通过边界剪裁获取影响每一个tile的灯光列表**：

```c
	// Compute per-tile lists of affecting lights through bounds culling
	// Each thread now operates on a sample instead of a pixel
	// 计算影响tile的灯光列表，这里的计算以Tile为单位而非pixel
	LOOP
	for (uint LightIndex = ThreadIndex; LightIndex < NumLights && LightIndex < MAX_LIGHTS; LightIndex += THREADGROUP_TOTALSIZE)
	{
       	... 
	}
//GPU线程同步
GroupMemoryBarrierWithGroupSync();
```

在灯光列表计算完成之后，就是再次遍历这个灯光列表，进行光照计算：

```c
	...	
	BRANCH
	if (InGBufferData.ShadingModelID != SHADINGMODELID_UNLIT)
	{
         // 遍历每一个灯光，读取灯光信息，填充LightData，这里和Unity出奇的相似。
		LOOP
		for (uint TileLightIndex = 0; TileLightIndex < NumLightsAffectingTile; TileLightIndex++) 
		{
			uint LightIndex = TileLightIndices[TileLightIndex];
			FDeferredLightData LightData = (FDeferredLightData)0;
			...
			// Lights requiring light attenuation are not supported tiled for now
			CompositedLighting += GetDynamicLighting(WorldPosition, CameraVector, ScreenSpaceData.GBuffer, ScreenSpaceData.AmbientOcclusion, InGBufferData.ShadingModelID, LightData, float4(1, 1, 1, 1), 0.5, uint2(0,0));
		}

		// 遍历每一个灯光，读取灯光信息，填充LightData，这里和Unity出奇的相似。和上面区别是灯光类型不同光照复杂程度不同
		LOOP
		for (uint TileLightIndex = 0; TileLightIndex < NumSimpleLightsAffectingTile; TileLightIndex++) 
		{
			uint LightIndex = TileSimpleLightIndices[TileLightIndex];

			FSimpleDeferredLightData LightData = (FSimpleDeferredLightData)0;
            	...
			// todo: doesn't support ScreenSpaceSubsurfaceScattering yet (not using alpha)
			CompositedLighting.rgb += GetSimpleDynamicLighting(...);
		}
	}
```

最终写出光照结果：

```c
	BRANCH
    if (all(DispatchThreadId.xy < ViewDimensions.zw)) 
	{
		// One some hardware we can read and write from the same UAV with a 32 bit format. We don't do that yet.
		RWOutTexture[PixelPos.xy] = InTexture[PixelPos.xy] + CompositedLighting;
    }
```

## DeferredLightingCommon.ush

这个文件延展自上一小节，TiledDeferredLightShaders.usf文件中的**光照计算代码GetDynamicLighting和GetSimpleDynamicLighting**就是来自这个文件。

GetDynamicLighting函数简要代码如下:

```c
/** Calculates lighting for a given position, normal, etc with a fully featured lighting model designed for quality. */
float4 GetDynamicLighting(float3 WorldPosition, float3 CameraVector, FGBufferData GBuffer, float AmbientOcclusion, uint ShadingModelID, FDeferredLightData LightData, float4 LightAttenuation, float Dither, uint2 Random)
{
	FLightAccumulator LightAccumulator = (FLightAccumulator)0;
	float3 V = ...;
	float3 N = ...;
	float3 L = ...;	// Already normalized
	float3 ToLight = ...;
	float NoL = ...;
	float DistanceAttenuation = 1;
	float LightRadiusMask = 1;
	float SpotFalloff = 1;

	...
	BRANCH
	if (LightRadiusMask > 0 && SpotFalloff > 0)
	{
		float SurfaceShadow = 1;
		float SubsurfaceShadow = 1;
		BRANCH
		if (LightData.ShadowedBits)
			GetShadowTerms(...);
		else
			SurfaceShadow = AmbientOcclusion;
		...
		if( LightData.ShadowedBits < 2 && GBuffer.ShadingModelID == SHADINGMODELID_HAIR )
		{
			SubsurfaceShadow = ShadowRayCast(...);
		}
	
		...
		float3 LobeEnergy = AreaLightSpecular(LightData, LobeRoughness, ToLight, L, V, N);
		float3 SurfaceLighting = SurfaceShading(GBuffer, LobeRoughness, LobeEnergy, L, V, N, Random);
		float3 SubsurfaceLighting = SubsurfaceShading(GBuffer, L, V, N, SubsurfaceShadow, Random);
		...
	}

	return LightAccumulator_GetResult(LightAccumulator);
}
```

​	从化简后的代码可以清楚的看出他的代码结构，而**AreaLightSpecular**、**SurfaceShading**和**SubsurfaceShading**就是光照计算的关键，我们的讨论到此为止，而接下来的内容涉及到光照函数、阴影算法、光照策略之后再讨论。

