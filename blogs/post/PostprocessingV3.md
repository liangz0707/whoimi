# PostProcessing V3版本的代码结构

Hdrp目前使用了PostV3版本，即将抛弃postV2，这里总结一下PostprocessingV3的基本代码结构。

PostV3是直接集成在HDRP当中的不需要在单独使用一个Package。

## 源代码结构

源代码的位置如下：

```c
package/High Definition RP/Runtime/PostProcessing
```

主要包括四个文件：

```c
GlobalPostProcessingSettings
IPostProcessComponent// 后处理接口
PostProcessSystem // 完整的后处理
TexctureCurve //为了支持曲线属性，这里给AnimationCurve封装了一层。
```

Shader文件都保存在Shaders目录当中，PostV3全部使用了ComputeShader。

所有后处理的组件保存在Components目录当中。下面以bloom为例分析V3后处理的实现方式。

## Bloom与后处理参数

在Bloom.cs源文件当中和V2一样使用了**Parameter类型的类，内部实现了插值接口。这部分内容是和Volume是通用的。

每一个Parameter继承了VolumeParameter。目前支持的Parameter类型也都在VolumeParameter.cs源文件当中。

V3版本的后处理实现和V2当中不同的是: 不再单独的实现后处理(例如:Bloom.cs当中只保存了Parameter参数，而没有后处理实现)，而是所有的后处理全部都集中在了PostProcessSystem.cs文件当中。

## 后处理实现

后处理实现都在PostProcessSystem.cs文件当中。

下面查看如何进行完整的后处理。

主要的类是：

```c
public sealed class PostProcessSystem{}
```

类当中首先保存了后处理的临时数据，纹理格式等不需要通过面板调整的参数：

```c
 // Exposure data
        const int k_ExposureCurvePrecision = 128;
        readonly Color[] m_ExposureCurveColorArray = new Color[k_ExposureCurvePrecision];
        readonly int[] m_ExposureVariants = new int[4];

        Texture2D m_ExposureCurveTexture;
        RTHandle m_EmptyExposureTexture; // RGHalf

        // Depth of field data
        ComputeBuffer m_BokehNearKernel;
        ComputeBuffer m_BokehFarKernel;
        ComputeBuffer m_BokehIndirectCmd;
        ComputeBuffer m_NearBokehTileList;
        ComputeBuffer m_FarBokehTileList;

        // Bloom data
        const int k_MaxBloomMipCount = 16;
        readonly RTHandle[] m_BloomMipsDown = new RTHandle[k_MaxBloomMipCount + 1];
        readonly RTHandle[] m_BloomMipsUp = new RTHandle[k_MaxBloomMipCount + 1];
        RTHandle m_BloomTexture;

        // Chromatic aberration data
        Texture2D m_InternalSpectralLut;

        // Color grading data
        readonly int m_LutSize;
        RTHandle m_InternalLogLut; // ARGBHalf

```

然后保存了各种后处理参数:

```c
Exposure m_Exposure;
DepthOfField m_DepthOfField;
MotionBlur m_MotionBlur;
PaniniProjection m_PaniniProjection;
Bloom m_Bloom;
ChromaticAberration m_ChromaticAberration;
LensDistortion m_LensDistortion;
Vignette m_Vignette;
Tonemapping m_Tonemapping;
WhiteBalance m_WhiteBalance;
ColorAdjustments m_ColorAdjustments;
ChannelMixer m_ChannelMixer;
SplitToning m_SplitToning;
LiftGammaGain m_LiftGammaGain;
ShadowsMidtonesHighlights m_ShadowsMidtonesHighlights;
ColorCurves m_Curves;
FilmGrain m_FilmGrain;
```

然后是初始化和清空数据的方法：

```c
public PostProcessSystem(HDRenderPipelineAsset hdAsset)
public void Cleanup()
```

抓取设置数据：

```c
BeginFrame（） // 这里面从VolumeManager当中读取所有的后处理设置。

var stack = VolumeManager.instance.stack;
m_Exposure                  = stack.GetComponent<Exposure>();
m_DepthOfField              = stack.GetComponent<DepthOfField>();
m_MotionBlur                = stack.GetComponent<MotionBlur>();
m_PaniniProjection          = stack.GetComponent<PaniniProjection>();
m_Bloom                     = stack.GetComponent<Bloom>();
m_ChromaticAberration       = stack.GetComponent<ChromaticAberration>();
m_LensDistortion            = stack.GetComponent<LensDistortion>();
m_Vignette                  = stack.GetComponent<Vignette>();
m_Tonemapping               = stack.GetComponent<Tonemapping>();
m_WhiteBalance              = stack.GetComponent<WhiteBalance>();
m_ColorAdjustments          = stack.GetComponent<ColorAdjustments>();
m_ChannelMixer              = stack.GetComponent<ChannelMixer>();
m_SplitToning               = stack.GetComponent<SplitToning>();
m_LiftGammaGain             = stack.GetComponent<LiftGammaGain>();
m_ShadowsMidtonesHighlights = stack.GetComponent<ShadowsMidtonesHighlights>();
m_Curves                    = stack.GetComponent<ColorCurves>();
m_FilmGrain                 = stack.GetComponent<FilmGrain>();
```

进行后处理：

```c
public void Render(CommandBuffer cmd, HDCamera camera, BlueNoise blueNoise, RTHandle colorBuffer, RTHandle afterPostProcessTexture, RTHandle lightingBuffer, RenderTargetIdentifier finalRT, RTHandle depthBuffer, bool flipY)
```

所有的后处理都在名字为Do**的方法中：

```c
void DoStopNaNs
void DoDynamicExposure
void DoBloom
...
```

## Render

Render方法中是整合后处理的完整流程

首先是开启Sampler()

```c
// 首先获取Handler？
var dynResHandler = HDDynamicResolutionHandler.instance;

// Handler相关的出来
if (dynResHandler.HardwareDynamicResIsEnabled() && dynResHandler.hasSwitchedResolution)
    m_Pool.Cleanup();

void PoolSource(ref RTHandle src, RTHandle dst)
{
    // Special case to handle the source buffer, we only want to send it back to our
    // target pool if it's not the input color buffer
    if (src != colorBuffer) m_Pool.Recycle(src);
    src = dst;
}

// 上面Handler部分没有详细研究，这个和SRP应该是相关的
//判断视图场景
bool isSceneView = camera.camera.cameraType == CameraType.SceneView;


using (new ProfilingSample(cmd, "Post-processing", 
                      CustomSamplerId.PostProcessing.GetSampler()))
{
    ...
}
```

在Sampler内部是完整的后处理过程，

```c
if (m_PostProcessEnabled)
{
    // 读取摄像机大小，设置渲染目标，可以看出来，这里的渲染使用的是commandbuffer的DrapProcedural，这里主要用来清空buffer，暂时这两个buffer用来做什么还没有详细研究。
    {
        int w = camera.actualWidth;
        int h = camera.actualHeight;
        cmd.SetRenderTarget(source, 0, CubemapFace.Unknown, -1);

        if (w < source.rt.width || h < source.rt.height)
        {
            cmd.SetViewport(new Rect(w, 0, k_RTGuardBandSize, h));
            cmd.DrawProcedural(Matrix4x4.identity, m_ClearBlackMaterial, 0, MeshTopology.Triangles, 3, 1);
            cmd.SetViewport(new Rect(0, h, w + k_RTGuardBandSize, k_RTGuardBandSize));
            cmd.DrawProcedural(Matrix4x4.identity, m_ClearBlackMaterial, 0, MeshTopology.Triangles, 3, 1);
        }
    }
    //首先判断是否启用了后处理，启用后处理的第一步，就是处理stopNaNs。
    //而且可以发现，如果要在Scene产生效果就需要特殊的设置。
    bool stopNaNs = camera.stopNaNs;
    #if UNITY_EDITOR
    if (isSceneView)
            stopNaNs = HDRenderPipelinePreferences.sceneViewStopNaNs;
    #endif
	
    // 清楚坏像素，否则在Bloom阶段会影响整个屏幕
    if (stopNaNs)
    {
        using (new ProfilingSample(cmd, "Stop NaNs", CustomSamplerId.StopNaNs.GetSampler()))
        {
            var destination = m_Pool.Get(Vector2.one, k_ColorFormat);
            DoStopNaNs(cmd, camera, source, destination);
            PoolSource(ref source, destination);
        }
    }
}
```

## Bloom

后处理的过程大体如上，现在详细看一下Bloom效果。

下面是Bloom部分的代码：

```c
bool bloomActive = m_Bloom.IsActive();

if (bloomActive)
{
    using (new ProfilingSample(cmd, "Bloom", CustomSamplerId.Bloom.GetSampler()))
    {
        // 这里开启Bloom
        DoBloom(cmd, camera, source, cs, kernel);
    }
}
else
{
    // 这里可以看到如果没有开启Bloom那么会设置几个默认的Bloom参数，也就是DoBloom最终要生成的结果。
    /*
    _BloomTexture Bloom纹理
    _BloomDirtTexture BloomDirtexture纹理
    _BloomParams Bloom参数
    上面的三个参数在Bloom开启式，在DoBloom中生产
    */
    cmd.SetComputeTextureParam(cs, kernel, HDShaderIDs._BloomTexture, TextureXR.GetBlackTexture());
    cmd.SetComputeTextureParam(cs, kernel, HDShaderIDs._BloomDirtTexture, Texture2D.blackTexture);
    cmd.SetComputeVectorParam(cs, HDShaderIDs._BloomParams, Vector4.zero);
}
```

下面看在DoBloom当中如何生成参数：

​    _BloomTexture Bloom纹理
​    _BloomDirtTexture BloomDirtexture纹理
​    _BloomParams Bloom参数

下面是DoBloom方法：

```c
// 首先计算Bloom贴图的大小，1/2 或者 1/4 分别对应half和quater
var resolution = m_Bloom.resolution.value;
float scaleW = 1f / ((int)resolution / 2f);
float scaleH = 1f / ((int)resolution / 2f);

// 下面是Bloom的变形和失真
if (m_Bloom.anamorphic.value)
{
    // 这里需要一个屋里摄像机的参数，用来控制XY方向的变形。
    float anamorphism = m_PhysicalCamera.anamorphism * 0.5f;
    scaleW *= anamorphism < 0 ? 1f + anamorphism : 1f;
    scaleH *= anamorphism > 0 ? 1f - anamorphism : 1f;
}

// 下面决定Bloom的迭代次数
// 首先计算最大长宽
int maxSize = Mathf.Max(camera.actualWidth, camera.actualHeight);
// 根据Log得到/2的整数并且考虑是half和quater，计算/2次数
int iterations = Mathf.FloorToInt(Mathf.Log(maxSize, 2f) - 2 - (resolution == BloomResolution.Half ? 0 : 1));
// 根据迭代次数
int mipCount = Mathf.Clamp(iterations, 1, k_MaxBloomMipCount);
var mipSizes = stackalloc Vector2Int[mipCount];

//第一次循环
for (int i = 0; i < mipCount; i++)
{	
    // 缩小的比例，每次缩小1/2
    float p = 1f / Mathf.Pow(2f, i + 1f);
    // 缩小后的比例，叠加了走样比例
    float sw = scaleW * p;
    float sh = scaleH * p;
    // 当前mip的实际像素大小。
    int pw = Mathf.Max(1, Mathf.RoundToInt(sw * camera.actualWidth));
    int ph = Mathf.Max(1, Mathf.RoundToInt(sh * camera.actualHeight));
    // 缩放比例
    var scale = new Vector2(sw, sh);
    // 像素大小
    var pixelSize = new Vector2Int(pw, ph);

    mipSizes[i] = pixelSize;
    // 获取对应的RT
    m_BloomMipsDown[i] = m_Pool.Get(scale, k_ColorFormat);
    m_BloomMipsUp[i] = m_Pool.Get(scale, k_ColorFormat);
}

// 这里定义了一个局部函数，主要的作用是进行上下采样。
void DispatchWithGuardBands(ComputeShader shader, int kernelId, in Vector2Int size)
{
    int w = size.x;
    int h = size.y;

    if (w < source.rt.width && w % 8 < k_RTGuardBandSize)
        w += k_RTGuardBandSize;
    if (h < source.rt.height && h % 8 < k_RTGuardBandSize)
        h += k_RTGuardBandSize;

    cmd.DispatchCompute(shader, kernelId, (w + 7) / 8, (h + 7) / 8, camera.computePassCount);
}

// 首先第一步是prefilter，对图像进行预滤波
// Pre-filtering
ComputeShader cs;
int kernel;

if (m_Bloom.prefilter.value)
{
    var size = mipSizes[0];
    cs = m_Resources.shaders.bloomPrefilterCS;
    kernel = cs.FindKernel("KMain");

    cmd.SetComputeTextureParam(cs, kernel, HDShaderIDs._InputTexture, source);
    cmd.SetComputeTextureParam(cs, kernel, HDShaderIDs._OutputTexture, m_BloomMipsUp[0]); // Use m_BloomMipsUp as temp target
    cmd.SetComputeVectorParam(cs, HDShaderIDs._TexelSize, new Vector4(size.x, size.y, 1f / size.x, 1f / size.y));
    DispatchWithGuardBands(cs, kernel, size);

    cs = m_Resources.shaders.bloomBlurCS;
    kernel = cs.FindKernel("KMain"); // Only blur

    cmd.SetComputeTextureParam(cs, kernel, HDShaderIDs._InputTexture, m_BloomMipsUp[0]);
    cmd.SetComputeTextureParam(cs, kernel, HDShaderIDs._OutputTexture, m_BloomMipsDown[0]);
    cmd.SetComputeVectorParam(cs, HDShaderIDs._TexelSize, new Vector4(size.x, size.y, 1f / size.x, 1f / size.y));
    DispatchWithGuardBands(cs, kernel, size);
}
else
{
    var size = mipSizes[0];
    // bloom的Blur用来下采样
    cs = m_Resources.shaders.bloomBlurCS;
    kernel = cs.FindKernel("KMainDownsample");
	
    // 设置输入和输出的纹理
    cmd.SetComputeTextureParam(cs, kernel, HDShaderIDs._InputTexture, source);
    cmd.SetComputeTextureParam(cs, kernel, HDShaderIDs._OutputTexture, m_BloomMipsDown[0]);
    
    // 设置尺寸参数，进行第一次下采样
    cmd.SetComputeVectorParam(cs, HDShaderIDs._TexelSize, new Vector4(size.x, size.y, 1f / size.x, 1f / size.y));
    DispatchWithGuardBands(cs, kernel, size);
}

// Blur pyramid

// 进行模糊金字塔的构建。这部分就是不断的进行1/2的下采样。
kernel = cs.FindKernel("KMainDownsample");
for (int i = 0; i < mipCount - 1; i++)
{
    var src = m_BloomMipsDown[i];
    var dst = m_BloomMipsDown[i + 1];
    var size = mipSizes[i + 1];

    cmd.SetComputeTextureParam(cs, kernel, HDShaderIDs._InputTexture, src);
    cmd.SetComputeTextureParam(cs, kernel, HDShaderIDs._OutputTexture, dst);
    cmd.SetComputeVectorParam(cs, HDShaderIDs._TexelSize, new Vector4(size.x, size.y, 1f / size.x, 1f / size.y));
    DispatchWithGuardBands(cs, kernel, size);
}

// Upsample & combine

// 进行模糊金字塔的构建。这部分就是不断的进行1/2的上采样。
cs = m_Resources.shaders.bloomUpsampleCS;
kernel = cs.FindKernel(m_Bloom.highQualityFiltering.value ? "KMainHighQ" : "KMainLowQ");

float scatter = Mathf.Lerp(0.05f, 0.95f, m_Bloom.scatter.value);

for (int i = mipCount - 2; i >= 0; i--)
{
    var low = (i == mipCount - 2) ? m_BloomMipsDown : m_BloomMipsUp;
    var srcLow = low[i + 1];
    var srcHigh = m_BloomMipsDown[i];
    var dst = m_BloomMipsUp[i];
    var highSize = mipSizes[i];
    var lowSize = mipSizes[i + 1];

    cmd.SetComputeTextureParam(cs, kernel, HDShaderIDs._InputLowTexture, srcLow);
    cmd.SetComputeTextureParam(cs, kernel, HDShaderIDs._InputHighTexture, srcHigh);
    cmd.SetComputeTextureParam(cs, kernel, HDShaderIDs._OutputTexture, dst);
    cmd.SetComputeVectorParam(cs, HDShaderIDs._Params, new Vector4(scatter, 0f, 0f, 0f));
    cmd.SetComputeVectorParam(cs, HDShaderIDs._BloomBicubicParams, new Vector4(lowSize.x, lowSize.y, 1f / lowSize.x, 1f / lowSize.y));
    cmd.SetComputeVectorParam(cs, HDShaderIDs._TexelSize, new Vector4(highSize.x, highSize.y, 1f / highSize.x, 1f / highSize.y));
    DispatchWithGuardBands(cs, kernel, highSize);
}

//上采样和下采样的具体算法需要和V2进行比较一下。

// Cleanup 清楚掉Buffer的数据，回收
for (int i = 0; i < mipCount; i++)
{
    m_Pool.Recycle(m_BloomMipsDown[i]);
    if (i > 0) m_Pool.Recycle(m_BloomMipsUp[i]);
}

// Set uber data 设置Bloom的相关数据
var bloomSize = mipSizes[0];
m_BloomTexture = m_BloomMipsUp[0]; //得到了Bloom的纹理

// 计算Bloom强度
float intensity = Mathf.Pow(2f, m_Bloom.intensity.value) - 1f; // Makes intensity easier to control
// Bloom颜色
var tint = m_Bloom.tint.value.linear;
var luma = ColorUtils.Luminance(tint);
tint = luma > 0f ? tint * (1f / luma) : Color.white;

// 镜头污渍：这部分应该不怎么使用，后续再了解原理
// Lens dirtiness
// Keep the aspect ratio correct & center the dirt texture, we don't want it to be
// stretched or squashed
var dirtTexture = m_Bloom.dirtTexture.value == null ? Texture2D.blackTexture : m_Bloom.dirtTexture.value;
int dirtEnabled = m_Bloom.dirtTexture.value != null && m_Bloom.dirtIntensity.value > 0f ? 1 : 0;
float dirtRatio = (float)dirtTexture.width / (float)dirtTexture.height;
float screenRatio = (float)camera.actualWidth / (float)camera.actualHeight;
var dirtTileOffset = new Vector4(1f, 1f, 0f, 0f);
float dirtIntensity = m_Bloom.dirtIntensity.value * intensity;
if (dirtRatio > screenRatio)
{
    dirtTileOffset.x = screenRatio / dirtRatio;
    dirtTileOffset.z = (1f - dirtTileOffset.x) * 0.5f;
}
else if (screenRatio > dirtRatio)
{
    dirtTileOffset.y = dirtRatio / screenRatio;
    dirtTileOffset.w = (1f - dirtTileOffset.y) * 0.5f;
}

//下面就是把计算好的参数设置。

cmd.SetComputeTextureParam(uberCS, uberKernel, HDShaderIDs._BloomTexture, m_BloomTexture);
cmd.SetComputeTextureParam(uberCS, uberKernel, HDShaderIDs._BloomDirtTexture, dirtTexture);
cmd.SetComputeVectorParam(uberCS, HDShaderIDs._BloomParams, new Vector4(intensity, dirtIntensity, 1f, dirtEnabled));
cmd.SetComputeVectorParam(uberCS, HDShaderIDs._BloomTint, (Vector4)tint);
cmd.SetComputeVectorParam(uberCS, HDShaderIDs._BloomBicubicParams, new Vector4(bloomSize.x, bloomSize.y, 1f / bloomSize.x, 1f / bloomSize.y));
cmd.SetComputeVectorParam(uberCS, HDShaderIDs._BloomDirtScaleOffset, dirtTileOffset);
```

我们法线这里只是把需要的参数设置给computeShader，实际的计算是在UberPost.compute文件当中的。

 **Chromatic aberration ， Bloom，Vignette，Grading, tonemapping这几个特效是同时在一个Pass当中写入到屏幕当中的。**

前面的内容只是在给这个cs设置参数。在最后合并所有结果。

```c
// Run
var destination = m_Pool.Get(Vector2.one, k_ColorFormat);
cmd.SetComputeTextureParam(cs, kernel, HDShaderIDs._InputTexture, source);
cmd.SetComputeTextureParam(cs, kernel, HDShaderIDs._OutputTexture, destination);
cmd.DispatchCompute(cs, kernel, (camera.actualWidth + 7) / 8, (camera.actualHeight + 7) / 8, camera.computePassCount);

```





