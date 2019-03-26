# 1.UnlitShader使用

1. 目前UnlitGraph支持主要的纹理、向量、颜色、顶点色、屏幕空间位置等各种基础参数的计算。

2. 通过宏来切换不同效果的部分需要手写谢。
3. 同时支持透明和不透明两种模式。
4. **扭曲distort，默认的Unlit shader已经可以使用**，但是UnlitGraph没有提供，LitGraph有这个功能，所以未来应该会提供。

使用方式：HDRP创建er默认材质，Shader改成Hdrp/Unlit，材质选项中Surface Type改成Transparent。此时材质下方出现了新的选项Transprency Input。打开Distortion，给出一张扭曲纹理，然后调节参数就可以看到扭曲效果。

**注意**：扭曲和渲染顺序无关，所有和扭曲材质重叠切没有渲染深度的材质，都会被扭曲。



# 2.LitShader

1. Lit一共有两种扭曲方式：Refraction折射和Distortion变形。LitGraph也可以使用这两个功能

2. 支持折射、扭曲，但是会带上光照。

3. Refraction：透明的Lit物体可以用折射功能（Unlit物体只可以选择是否被折射）。Lit物体如果要折射其他物体就不能被其他的物体折射。不透明物体可以选择是否被折射。可以用来实现类似冰冻怪物的效果。

4. Distortion：Unlit和透明的Lit都可以用。有扭曲的Unlit材质会扭曲任何和他重叠的无深度物体。



# 3.Unlit贴花

1. 默认的功能无法实现Unlit效果。我们需要自己实现Unlit Decal，理论上只要能够正确读取深度就可以渲染，不过需要研究一下渲染输出的buffer、渲染队列、已经新的深度读取方式。

2. **内置贴花支持模型贴花**，一个Decal模型可以贴附在另一个模型上，可以有自己的UV。



# 4.天空球编程

参考链接：[天空球编程](https://github.com/Unity-Technologies/ScriptableRenderPipeline/wiki/Writing-A-Custom-Sky-Renderer)，可以自由实现想要的天空球。



#5.HDRP 自带天空球混合

Unity Volume源代码中有实现HDRI插值的计划，见：TODO注释：

```c
public class CubemapParameter : VolumeParameter<Cubemap>
{
     public CubemapParameter(Cubemap value, bool overrideState = false)
     : base(value, overrideState) {}
     // TODO: Cubemap interpolation
}

public class NoInterpCubemapParameter : VolumeParameter<Cubemap>
{
    public NoInterpCubemapParameter(Cubemap value, bool overrideState = false)
    : base(value, overrideState) {}
}
```

Volume当中的不同天空球都可以实现各自的混合效果，不过Hdri目前没有实现完（如果是其他两种类型的天空球，当前版本已经可以实现插值）。cubemap在混合时实时计算插值，有一部分开销需要考虑。在HDRP6.0.0当中也还是没有实现，不知道Unity是出于什么考虑，一直没有给出Cubemap混合的代码。

目前，**我们实现这部分代码就可以直接使用Volume当中的天空球插值**。



#6.关于特效和渲染队列

之前美术手动调节了材质上的渲染队列来实现不同的特效。现在渲染队列的机制有一些改动，在不熟悉之前最好不要手动设置队列（默认材质都是不能调节的，只能修改**渲染优先级Sort Priority**），通过Shader Graph创建的可以手动设置。

现在，在不透明物体渲染阶段，不会渲染2650以上的物体（看源代码得到的结果，在以前只要有队列，就会按照队列顺序渲染），反之亦然。

HDRP理论上不建议手动调整材质顺序，因为这个顺序不在单纯的是顺序，还涉及到各种特殊的效果。

2650-2850用来渲染被折射的透明物体，这个都是通过设置自动配置的，在不熟悉引擎之前不要修改材质队列。

2900-3100是一把透明物体。

2850到2900之间会出现奇怪的变化，Unity没有给出这部分渲染的是什么。

2000到2500是不透明物体。这个队列以外的不透明物体不会渲染。

2000以下什么都不会渲染（以前都是可以渲染的）。

**现在的状态就是：如果没有把队列和物体类型匹配就会出现不可预知的结果。**

下面是控制渲染队列的代码。

```c
public enum Priority
        {
            Background = UnityEngine.Rendering.RenderQueue.Background,
            Opaque = UnityEngine.Rendering.RenderQueue.Geometry,
            OpaqueAlphaTest = UnityEngine.Rendering.RenderQueue.AlphaTest,
            // Warning: we must not change Geometry last value to stay compatible with occlusion
            OpaqueLast = UnityEngine.Rendering.RenderQueue.GeometryLast,
            // For transparent pass we define a range of 200 value to define the priority
            // Warning: Be sure no range are overlapping
            PreRefractionFirst = 2750 - k_TransparentPriorityQueueRange,
            PreRefraction = 2750,
            PreRefractionLast = 2750 + k_TransparentPriorityQueueRange,
            TransparentFirst = UnityEngine.Rendering.RenderQueue.Transparent - k_TransparentPriorityQueueRange,
            Transparent = UnityEngine.Rendering.RenderQueue.Transparent,
            TransparentLast = UnityEngine.Rendering.RenderQueue.Transparent + k_TransparentPriorityQueueRange,
            Overlay = UnityEngine.Rendering.RenderQueue.Overlay
        }

RenderQueueRange k_RenderQueue_OpaqueNoAlphaTest = 
    new RenderQueueRange { min = (int)Priority.Opaque, 
                          max = (int)Priority.OpaqueAlphaTest - 1 };

RenderQueueRange k_RenderQueue_OpaqueAlphaTest = 
    new RenderQueueRange { min = (int)Priority.OpaqueAlphaTest, 
                          max = (int)Priority.OpaqueLast };

RenderQueueRange k_RenderQueue_AllOpaque = new RenderQueueRange { 
    min = (int)Priority.Opaque,
    max = (int)Priority.OpaqueLast };

RenderQueueRange k_RenderQueue_PreRefraction = new RenderQueueRange { 
    min = (int)Priority.PreRefractionFirst, 
    max = (int)Priority.PreRefractionLast };

RenderQueueRange k_RenderQueue_Transparent = new RenderQueueRange { 
    min = (int)Priority.TransparentFirst, 
    max = (int)Priority.TransparentLast };

RenderQueueRange k_RenderQueue_AllTransparent = new RenderQueueRange { 
    min = (int)Priority.PreRefractionFirst, 
    max = (int)Priority.TransparentLast };

RenderQueueRange k_RenderQueue_All = new RenderQueueRange { min = 0, max = 5000 };
    }
```

