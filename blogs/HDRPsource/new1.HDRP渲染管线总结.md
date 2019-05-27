# Unity HDRP主要功能和渲染管线的介绍

## 1. SRP和HDRP介绍

unity SRP可以让我们自己控制摄像机每一帧的渲染流程。他把渲染管线可以进行的操作都暴露给了用户，不再是一个封闭的引擎内置流程。

为了方便用户使用，Unity提供了两个已经实现好的SRP流程，分别针对不同的平台。

* Lightweight Render Pipeline (LWRP) ：支持手机平台和PC平台，目的是提供一个高性能的渲染，牺牲了引擎的表现效果。
* High Definition Render Pipeline (HDRP) ：使用了基于物理的灯光技术以及基于Compute Shader的光照计算。针对高端的PC和主机平台。


## 2. HDRP渲染效果与材质类型

Unity HDRP提供了更丰富的材质效果，包括：SSS效果、透光、Coat等。通过ShaderGraph和基础的LitShader都可以构建出支持这些效果的Shader。

在Lit Shader中可以通过Surface Options当中的Material Type来选择需要的材质效果。

Unity的说明文档:[材质类型官网介绍](https://docs.unity3d.com/Packages/com.unity.render-pipelines.high-definition@6.7/manual/Material-Type.html)。

下面简要介绍这些材质效果主要的功能。

### 2.1 Standard

Standard就是标准的光照模型。渲染流程如下：

- Gbuffer Pass ：填充GBuffer
- Deferred Lighting Pass：Compute Shader计算光照。

基础的输入参数：

* Diffuse贴图：表面颜色，同时支持用Color参数对其进行调整。Alpha通道在开启透明模式的时候，可以用作透明度。
* Normal贴图：支持Object空间和Tangent空间，支持调节法线强度。
* Mask贴图：R通道对应金属度，G通道对应平滑度，B通道对应细节贴图的区域，A通道对应平滑度，同时提供了多个滑块参数对金属度和平滑度进行调整。
* BentNormal贴图：主要用于计算AO，比AO贴图准确。
* DetialMap贴图：支持平铺的细节，可以给Diffuse、normal、smoothness调节细节，通过Mask贴图B通道控制区域。
* Height贴图：开启Displacement Mode，可以使用高度贴图。
* 开启透明模式：支持扭曲、折射。
* 支持Clear Coat：效果。

上面的效果Lit材质球默认支持，Lit的Inspector是一个专门为LitShader写的材质编辑器，通过开启选项可修改Lit的材质效果。

另外，可以看到Lit支持很多贴图输入，一般而言这样做开销很高，但是Lit Shader内部做了优化，如果不使用某一张贴图就不要给他赋值，它就不会对这个贴图进行采样。

### 2.2 Specular

这个是和Standard（金属工作流）相对的高光工作流。区别只是金属度变成了高光颜色。其他计算没有区别。

### 2.3 Anisotropy

计算高光时使用各项异性的高光计算。和Standard区别只在于高光的计算，同时多出了一张各向异性贴图和强度控制。例如：拉丝金属。

### 2.4 Iridescence

彩虹色是指：随着光照角度的变化光照颜色发生变化。例如：肥皂泡沫，昆虫翅膀。

和Standard的区别在于：光的颜色随光照角度发生变化。

### 2.5 Translucent

透光效果是指：在背光面可以看到光穿过物体的效果。主要用于半透明材质。例如：树叶。

**以上效果的开销没有太大区别，效果的差距只在于计算公式的不同，而渲染流程基本一致。**

### 2.6 Surface Scattering

SSS效果用来描述光线在表面多次散射的效果。可以用来描述灯光和半透物体的交互过程。可以用来制作：玉、冰、皮肤等物体。

Surface Scattering的效果和上面几种效果不同，并且需要额外的计算。大致的渲染流程如下：

- Gbuffer Pass ：和Standard一样填充Gbuffer
- Deferred Lighting Pass：计算光照
- Convolution Pass：对需要SSS的部位进行卷积（通过Diffuse Profile）。
- Combine Pass：计算好的SSS光和原始光融合。

### 2.7 Displacement,  高度图

Unity支持高度图计算。

所有的材质都可以使用高度图。

在Surface Options 当中选择Displacement Mode：

None：不使用高度图。

Vertex Displacement:  在Vertex阶段直接移动顶点，高度图作为移动距离。

Pixel displacement： 将高度图作为视差贴图。（POM）

当开启Pixel displacement时，视差贴图对应的参数就会开启。

官方说明：[Displacement Mode](<https://docs.unity3d.com/Packages/com.unity.render-pipelines.high-definition@6.7/manual/Displacement-Mode.html>)

## 3. 渲染流程介绍

这里主要描述实现上述效果需要的基本渲染流程。

HDRP中Lit的渲染流程和Built-in Standard当中的延迟渲染基本一致，不透明物体流程如下：

* Gbuffer Pass  :填充Gbuffer，不过Gbuffer的内容和Built-in的内容有所区别。
* Deferred Lighting Pass：进行光照计算。HDRP当中的光照计算放在了ComputeShader当中，计算效率更高。

### 3.1 Gbuffer结构

下面是HDRP当中Gbuffer的结构类型。

| G-Buffer Usage      | Format    | RGB                                   | A                                   |
| ------------------- | --------- | ------------------------------------- | ----------------------------------- |
| GBuffer0            | RGBA32    | Albedo Color /      SSS Color         | Spacular Occlusiion / SSS Parameter |
| GBuffer1            | RGBA32    | Packed Normal                         | Roughness                           |
| GBuffer2            | RGBA32    | BSDF Model Specific Parameters        | Coat Mask + Material ID             |
| GBuffer3            | R11G11B10 | GI + Emissive                         | /                                   |
| GBuffer4 - Optional | RGBA32    | R:/             G:/             B: AO | Light Layer                         |
| GBuffer5 - Optional | RGBA32    | shadowmask0 - 2                       | shadowmask3                         |

其中Gbuffer0 、Gbuffer1、Gbuffer3的内容和Built-in的内容类型基本一致，主要是金属工作流需要的参数。

Gbuffer2和Gbuffer4当中保存了用于区分材质的标记，例如：

* Light Layer: 区分使用哪个Layer的光源。
* Coat Mask：区分使用Coat效果的区域。
* Material ID：区分材质类型。

还有其他的参数可以在Shader当中查看具体使用方式。

### 3.2 灯光列表

HDRP使用FPTL和Cluster策略，将灯光列表一次性传递到一个Pass当中完成计算。

Unity灯光计算就是（使用FPTL或Cluster策略）生成影响每一个Tile的灯光列表，在Compute Shader当中根据自己所在的Tile读取光照列表，然后一次性完成灯光计算。

#### 3.2.1 FPTL和Cluster

Unity有两种灯管列表的计算方式，FPTL的方式能够依赖深度区分需要使用的灯光是否能够作用在表面。透明物体没有深度，所以需要使用Cluster方式对灯管列表进行进一步的纵向划分。

Cluster将视锥体划分成多个体素块，在体素块中保存属于它的内容索引。体素块当中不止保存了灯光列表索引，还可以保存光照探针、Decal、Density Volume的内容。

FPTL目的是为了能够在一个pass当中读取所有的光照信息，这样就能够在一个Pass当中完成光照计算，节省了大量DrawCall。

如果要想查看灯光列表计算方式，查看LightLoops.cs源文件中的BuildGPULightListsCommon函数。

**下面简要介绍FPTL的计算过程和使用方式：**

0. 首先需要场景中所有灯光，场景灯光是在CPU中统一收集的，记录了**光照信息**及其**包围盒**。

1. FPTL是基于屏幕坐标计算的所以需要将**灯光转换到摄像机空间的矩阵（上一步已经完成）**。

   **这部分计算在函数LightLoop.cs:PrepareLightsForGPU当中。这部分主要是CPU计算。统计LightList,讲LightBounds转换到View空间**

2. 生成剪裁空间的AABB包围盒：用于FPTL和Cluster。代码位于：Runtime/Lighting/LightLoop/scrbound.compute

```c
//scrbound.compute
// 读取光的AABB包围盒
 SFiniteLightBound lgtDat = g_data[eyeAdjustedLgtIndex];
/*
包围盒数据结构
    public struct SFiniteLightBound
    {
        public Vector3 boxAxisX;
        public Vector3 boxAxisY;
        public Vector3 boxAxisZ;
        public Vector3 center;        // a center in camera space inside the bounding volume of the light source.
        public Vector2 scaleXY;
        public float radius;
    };
  */

// 包围盒的计算函数 LightingConvexHullUtils.hlsl
void GetHullQuad(out float3 p0, out float3 p1, out float3 p2, out float3 p3, const float3 boxX, const float3 boxY, const float3 boxZ, const float3 center, const float2 scaleXY, const int sideIndex)
{
    // 开启6个thread分别计算light的AABB的六个面在view空间的AABB凸包。因为线程每次提交8个所以两个thread被丢弃。
    ...
    GroupMemoryBarrierWithGroupSync();
    // 保留6个thread中的第一个，组合成一个完整的AABB凸包。 计算在屏幕中的位置。其中包括了大量的优化计算。最终输出到g_vBoundsBuffer当中。这里面包括的z坐标。
    ...
    g_vBoundsBuffer[boundsIndices.min] = ...;
    g_vBoundsBuffer[boundsIndices.max] = ...;
}
```

3. 根据屏幕空间的AABB生成big-Tile：（enable coarse 2D pass on 64x64 tiles ）。目的是优化fptl和cluster的计算。

```c
//lightlistbuild-bigtile.compute:BigTileLightListGen
//CPU提交时每一个big-tile提交一次。GPU每一次提交计算64个thread。

[numthreads(NR_THREADS, 1, 1)]
void BigTileLightListGen(uint threadID : SV_GroupIndex, uint3 u3GroupID : SV_GroupID)
{
    // 首先64个thread同事计算所有光照列表，判断每一个光源的AABB是否在这个Tile当中。通过元操作进行累加和记录
    ...
	// 在这个tile中，累加并返回uIndex当前值。
    unsigned int uInc = 1;
    unsigned int uIndex;
    InterlockedAdd(lightOffs, uInc, uIndex);
	// 记录灯光索引
    if(uIndex<MAX_NR_BIGTILE_LIGHTS) lightsListLDS[uIndex] = l;     // add to light list
    ...
    // sort lights:对光源类型排序。
    SORTLIST(lightsListLDS, iNrCoarseLights, MAX_NR_BIG_TILE_LIGHTS_PLUS_ONE, t, NR_THREADS);
    ...
    // 输出 :第一个放光源个数，后面是索引。
 for(i=t; i<(iNrCoarseLights+1); i+=NR_THREADS)
        g_vLightList[MAX_NR_BIG_TILE_LIGHTS_PLUS_ONE*offs + i] = i==0 ? iNrCoarseLights : lightsListLDS[max(i-1, 0)];
```

4. 进行FPTL计算。参考相关论文或者Shader源代码。
5. 进行Cluster计算，这里面不只包括了光照，还包括了环境光、Decal、DensityVolume。

### 3.3 光照计算

​	完成光照计算和Gbuffer填充之后就可以进行最终的光照计算，HDRP的Gbuffer光照计算使用了Compute Shader。

Compute Shader源文件：

```c
HDRP/Runtime/Lighting/LightLoop: Deferred.compute.
```

在lightloop函数当中读取了光照列表，使用BSDF函数计算最终的材质颜色。

### 3.4 BSDF

​		由于物体表面上有凹凸不平的微小表面，一道入射光线射到表面会产生光的散射现象，BSDF 用来表示这种散射现象（散射到各个方向的光的强度）。

Unity HDRP支持多种BSDF函数，例如：

Lit Shader的BSDF计算位于Lit.hlsl

Hair Shader的BSDF计算位于Hair.hlsl

### 3.5 BSDF中PBR材质参数的意义

在光照计算当中除了光源的参数和BSDF函数以外，最重要的就是材质相关的参数，下面表格解释了基本的材质参数。结合了Unity人员的说明和Shader源代码总结了如下内容：

| **Property**           | **Description**                                              |
| :--------------------- | :----------------------------------------------------------- |
| **Base Color**         | 表示物体颜色，不应该包含任何明暗、AO、阴影信息。除了基础颜色外，还可以包含一些偏色、或者色相的变化。 |
| **Smoothness**         | 平滑度，用来描述物体的粗糙程度。在Shader源代码中，Smoothness会转化成粗糙度进行计算。有两种粗糙度。PerceptualRoughness = 1- smoothness。Roughness= PerceptualRoughness * PerceptualRoughness； |
| **Ambient Occlusion ** | AO贴图：unity人员说可以使用sRGB格式，也可以使用RGB格式，但是需要统一。主要用来控制环境光（间接光）的明暗，对于直接光照没有影响。AO用来表现物体的明暗层次。 |
| **Mask Map**           | Assign a Texture that packs different Material maps into each of its RGBA channels. • **Red**: Stores the metallic map.  • **Green**: Stores the ambient occlusion map. • **Blue**: Stores the detail mask map. • **Alpha**: Stores the smoothness map. |
| **Normal **            | 法线，LitShader支持模型空间和切线空间的法线，使用模型空间法线，可以在LOD当中过度更好，使用切线空间法线，可以压缩纹理空间。 |
| **Bent Normal**        | Bent normal实际上也是一个表面方向和Normal接近，主要用来优化AO效果，只对GI（lightmap/lightprobe/volume proxy）生效。shaer代码：builtinData.bakeDiffuseLighting = SampleBakedGI(posInput.positionWS, bennormalWS, texCoord1.xy, texCoord2.xy); |
| **Coat Mask**          | Coat效果强弱，0-1之间                                        |

## 4. HDRP的其他渲染特性

### 4.1 StackLit

Unity提供了ShaderGraph Package。能够实现LitShader当中的所有功能。但是无法修改光照计算。如果需要修改光照计算就需要手写Shader。默认的Shader Graph的Lit节点使用的是Deferred光照，所以能够支持的效果有限（由于Gbuffer容量限制）

为了实现更好的效果，**Shader Graph提供了Forward模式的StackLit Master节点**，它支持更加物理正确的Shading模型，提供了更多的渲染功能。

使用StackLit可以制作：包括头发、毛发、布料、皮肤等效果。但是**Forward模式渲染开销非常大，所以不建议在游戏当中使用**，但是可以在选人界面等简单场景使用。

### 4.2 透明物体渲染

HDRP当中的所有透明物体都是在Forward下渲染的，所以渲染开销比较大。

除此之外透明物体还支持两个pass：TransparentDepthPrepas和TransparentDepthPostpass。下面是透明物体渲染部分的Pipeline代码和对应说明：

```c
// 渲染天空球 
RenderSky(hdCamera, cmd);
// 渲染透明物体的Prepass，用于处理透明物体之间的遮挡。
RenderTransparentDepthPrepass(cullingResults, hdCamera, renderContext, cmd);


// 渲染需要被折射的透明物体。RenderQueue:[2750-100,2750+100]
RenderForward(cullingResults, hdCamera, renderContext, cmd, ForwardPass.PreRefraction);

// 如果开启RoughRefraction，则生成ColorPyramid
{
	...
    RenderColorPyramid(hdCamera, cmd, true);
}

cmd.SetGlobalTexture(HDShaderIDs._ColorPyramidTexture, currentColorPyramid);

// 渲染所有其他透明物体。
// RenderQueue:[3000-100,3000+100]（没有开启LowResolutionTrans）
// RenderQueue:[3000-100,3400+100]（开启LowResolutionTrans）
RenderForward(cullingResults, hdCamera, renderContext, cmd, ForwardPass.Transparent);

...

// 渲染LowResTransparency。RenderQueue:[3400-100,3400+100]
DownsampleDepthForLowResTransparency(hdCamera, cmd);
RenderLowResTransparent(cullingResults, hdCamera, renderContext, cmd);
UpsampleTransparent(hdCamera, cmd);

// 渲染TransparentDepthPostpass，为了能够在需要深度的后处理效果中正确的处理透明物体  
RenderTransparentDepthPostpass(cullingResults, hdCamera, renderContext, cmd);

// 渲染颜色金字塔。
RenderColorPyramid(hdCamera, cmd, false);

// 渲染扭曲效果。
AccumulateDistortion(cullingResults, hdCamera, renderContext, cmd);
RenderDistortion(hdCamera, cmd);
```

上面提到了渲染LowResTransparency，如果要使用这个功能就需要在HDRPRenderPipelineAsset中开启LowResTransparency。

开启LowResTransparency后，材质中的LowRes属性生效。勾选这个属性，就会在低分辨率的RenderTarget上渲染这个透明物体，由于透明物体是Forward渲染。所以这样可以大幅提升效果。

**另外可以看到，代码是通过RenderQueue区分一个物体在什么时候被渲染的，所以不建议修改material的Queue，而是使用Priority。**例如：在3200到3300这段区间的物体，如果HDRP设置中没有开启LowResTran选项，则永远不会被渲染。

### 4.3 Decal

Unity HDRP提供了Decal的支持。原始Decal直接往Gbuffer写数据，而HDRP的Decal使用了Dbuffer先存储Decal的信息，然后在计算Gbuffer的过程中在合并到光照计算中。

**Decal有两种一种是Dbuffer Decal一种是ClusterDecal。**

**Dbuffer Decal：**（Projector Decal就是一个方块，可以往Dbuffer（类似Gbuffer）写入Decal纹理）。

**Dbuffer 结构：**类似Gbuffer结构，可以影响物体Diffuse，Normal，Emssive等，不需要GI信息（Dbuffer0,Dbuffer1,Dbuffer2）。

HDRenderPipelineAsset有设置:Metal & Amitent Propterty，用来控制Decal是否影响金属度和AO，勾选以后就会多使用一张Dbuffer，增加开销。

**Emssive Decal：**会多绘制一次。Emissive不能受光照影响。

**Mesh Decal：**和透明物体很像，直接绘制到Debuffer当中。

**关于Decal物体的剪裁：**

Decal 参与Culling，所以在CPU上有消耗。我们可以仿照Decal的方式对我们自己管理的对象用Culling Group手动culling。

**ClusterDecal：**HDRP支持透明物体的Decal，Dbuffer需要深度。不透明物体没有深度，所以不透明物体需要使用Cluster信息。与Cluster灯光原理一样。Decal 有Affect Transparent选项，开启后生效。

**不过LightList当中不光有Light信息，还有Reflection Probe ，Decal，Local Density Volume信息。**

**其他：**

视锥体内不依赖深度的简单物体都可以使用Cluster方式来做。这样就不用在CPU上做物体和物体的相交计算。通过记录索引可以让大部分物体进行合并。

例如：不是同一个Reflection Probe，可以Batch在一起。因为他们不依赖CPU来决定这次Batch使用哪一个Probe，而是通过LightList在GPU上实时计算。 然后通过Cluster当中的List查找信息。

Cluster Decal会被patch在一个大的纹理，一起传送给GPU，通过UV偏移来找。在Setting当中有decal相关选项：Atlas Weight和Atlas Height，这个选项直接影响占用显存的大小。

### 4.4 体积光

**全局体积雾设置：**场景当中的SceneSetting /Volumetric Fog 是全局的体积光的设置。

**Density Volume局部的体积雾**：只控制Abledo和浓度，其他的参数通过Global控制。

**基础原理：**

HDRP体积光不是屏幕空间的体积光（从屏幕空间 Reymarching），他是视锥体空间的3D体素纹理。有一个体素化的Pass，会把能看见的Density Volume体素化（Voxelization）到3D纹理里。Density Volume也是通过Cluster来记录的。体素化的时候通过ID知道是哪个Density Volume。

**Volumetric 参数：**

体积雾的颜色由大气散射决定（需要使用Precedual Sky），如果要用HDRISky，就需要自己计算大气散射颜色。

计算方式参考：Pre-compute Atemosphere Scattering相关文章。

**Phase Function（全局各向异性）：**光线在各个方向的散射强度。类似BRDF函数。

不同的物体Phase Function不同，例如：冰晶（强透射和折射）、尘土（吸收阻挡光照）。Phase Function主要控制朝向光的时候亮，还是背向光的方向亮、已经高光环的形状。灰尘就是Phase Function（0）只有颜色，水就是越朝光看越亮，大晴天很少有所以一般做成0。

**相关设置：**HDRenderPipelineAsset（涉及到资源分配的内容）当中可以设置精度（3D体素的分辨率）：Volumetric Hight Quality。

Hight Quality 是屏幕1/4大小，slice是128.

Midum 是屏幕1/8大小，深度slice是64.

PS4上测试：  屏幕1/16 大小 , slice64耗时0.8毫秒。 优化时需要考虑：体素化、Volume数量。

**体积雾纵向切分方式：**

Slice Distribution Uniformition：Slice的分布，近处密集还是等距划分。

Depth Extrent：覆盖多远。



## 5. HDRP与引擎自带模块

**Culling 模块：**

我们可以定义自己的Custom数据，然后使用Unity的Culling Group，每帧收集数据转换成AABB让引擎Culling。HDRP的Volume、Decal都是使用的这种方式。

**Baking 模块：**

支持原始所有内容。

GI信息都是一个物体计算一次，在填充Gbuffer时计算。

小型物体推荐使用light probe。大物体推荐用Lightmap或者开启light probe volume。

## 6. Package

**Core Package：**

包含大量的基础c#渲染工具。

包含光照计算等基础的Shader函数。

包含重要的功能：Volume System。

**ShaderGraph：**

**基础功能：**Lit和StackLit等自带master节点。

**扩展功能：**ShaderGraph模板和Shader Custom Node。

**VFX：**

必须和HDRP一起用。需要Compute Shader，基于indrect Dispatch效率高。

**基础原理：**将Compute buffer作为一个RWbuffer，放到compute Shader里面。在Compute Shader里面讲vertex buffer和index buffer动态生成好，然后把这个Compue buffer作为VB和IB传递到Graphic Pipeline里进行渲染。

LWRP不计划加入ComputeShader。

**扩展功能：**VFX模板、自定义Shader、ShaderGraph（还不支持）

