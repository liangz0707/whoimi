链接：[Scriptable Render Pipeline Overview](https://blogs.unity3d.com/cn/2018/01/31/srp-overview/)

# Scriptable Render Pipeline Overview

	>unity2018当中介绍了可编程的渲染管线（SRP）。SPR可以通过c#来控制unity中渲染的配置和实现。在了解用户可以定制的渲染管线之前，需要先知道我们所谈论的渲染管线是什么。

##什么是渲染管线

​	渲染管线是一个概括性的术语，用来概括将对象（Object）绘制到屏幕上的一系列的技术。这是一个高度概括的概念，包括了：

- 剪裁 
- 对象渲染
- 后处理

  ​除了这些抽象的概念之外，他们本身的功能还可以根据用户的需求进一步划分。

  ​例如渲染对象可以通过以下方式实现：

- Multi-pass rendering：每个pass渲染一个对象一个光照

- Single-pass：每个对象使用一个pass

- Deffered：将表面属性渲染到G-buffer，然后在屏幕空间实现光照

  当我们使用SRP的时候，就需要决定使用哪一种。不同的方式有不同的优缺点。

## 渲染入口点（Rendering entry point）

​	使用SRP时，你需要定义一个类用来控制渲染；渲染管线就是通过这个类来创建的。入口点是对"Render"方法的一次调用，每一次渲染都有不同的渲染上下文（Render Context，下文解释）和需要渲染的摄像机。

```c#
public class BasicPipeInstance : RenderPipeline
{
//context 渲染上下文 ，cameras 摄像机列表
   public override void Render(ScriptableRenderContext context, Camera[] cameras){}
}
```

## 渲染管线上下文（Render Pipeline Context）

​	SPR（脚本化渲染管线）渲染使用了延迟执行的概念。首先建立一系列的命令，然后统一执行。通过ScriptableRenderContext类来建立这一系列命令。当用一系列操作填充了这个上下文对象之后，可以 通过“submit"方法，来一次性提交所有drawcall。

​	例子：使用CommandBuffer清空渲染目标：

```c#
// Create a new command buffer that can be used
// to issue commands to the render context
var cmd = new CommandBuffer();

// issue a clear render target command
cmd.ClearRenderTarget(true, false, Color.green);

// queue the command buffer
context.ExecuteCommandBuffer(cmd);
```

## 剪裁Culling

剪裁用来确定哪些内容需要渲染的屏幕上。

Unity中剪裁包含：

- 视锥体剪裁：用摄像机的剪裁平面。
- 遮挡剪裁：物体遮挡关系

渲染开始时，需要决定哪些内容需要被渲染。这需要使用摄像机的透视图执行剪裁操作。剪裁结果是一个对象列表和光照列表。这些内容在接下来的渲染管线当中使用。

### 在SRP当中执行剪裁

在SRP当中你通常需要摄像机的透视来完成渲染。unity内置的渲染也是使用同一个摄像机。SRP提供了一系列剪裁API。

一般的剪裁流程：

```c#
// Create an structure to hold the culling paramaters
ScriptableCullingParameters cullingParams;

//Populate the culling paramaters from the camera
if (!CullResults.GetCullingParameters(camera, stereoEnabled, out cullingParams))
    continue;

// if you like you can modify the culling paramaters here
cullingParams.isOrthographic = true;

// Create a structure to hold the cull results
CullResults cullResults = new CullResults();

// Perform the culling operation
CullResults.Cull(ref cullingParams, context, ref cullResults);
```

之后可以进行下一步渲染。

## Drawing 绘制

现在的到了剪裁结果，可以进行进一步渲染。

这里有很多种配置方法，有下面内容决定：

- 目标平台

- 特殊的效果

- 项目类型

  具体的例子。

- HDR和LDR

- Linear和Gamma

- MSAA和后处理AA

- PBR材质和一般材质

- 有无光照

- 光照技术

- 阴影基础

### 过滤：桶和层

不同的对象有不同的类型，可以是透明不透明等。Unity使用队列的概念来决定对象渲染的时机。这些队列构成了桶，渲染对象将会保存在桶中（类似桶排序）。当我们使用SRP时，我们可以决定哪个范围的桶可以被渲染。

除了桶之外，标准的分层也可以用于过滤。

例子：

```c#
// Get the opaque rendering filter settings
var opaqueRange = new FilterRenderersSettings();

//Set the range to be the opaque queues
opaqueRange.renderQueueRange = new RenderQueueRange()
{
    min = 0,
    max = (int)UnityEngine.Rendering.RenderQueue.GeometryLast,
};

//Include all layers
opaqueRange.layerMask = ~0;
```

### 绘制设置：物体如何被绘制

之前的过滤和剪裁都是决定了哪些内容需要被渲染。接下来决定这些物体如何被渲染。SRP提供了一系列的操作，对通过过滤的物体进行渲染配置。配置这些数据的结构是：DrawRenderSettings。这个结构允许以下内容的配置：

- Sorting - 物体绘制顺序。
- 每个渲染标志 - 传递内部参数。
- 渲染标志 - 动态合并实例化技术等等算法
- Shander Pass -当前的draw使用哪一个pass

```c#
// Create the draw render settings
// note that it takes a shader pass name
var drs = new DrawRendererSettings(Camera.current, new ShaderPassName("Opaque"));
 
// enable instancing for the draw call
drs.flags = DrawRendererFlags.EnableInstancing;
 
// pass light probe and lightmap data to each renderer
drs.rendererConfiguration = RendererConfiguration.PerObjectLightProbe | RendererConfiguration.PerObjectLightmaps;
 
// sort the objects like normal opaque objects
drs.sorting.flags = SortFlags.CommonOpaque;
```

### 绘制

现在完成了：

- 剪裁规则
- 过滤规则
- 绘制规则

我们需要执行一个draw call。一个draw call就是对上下文的一次调用。在SRP中通常不渲染独立的网格，除非你需要一次性绘制很多。这减小了开销，同时保证了CPU的快速执行。

发布draw call

```c#
// draw all of the renderers
context.DrawRenderers(cullResults.visibleRenderers, ref drs, opaqueRange);
 
// submit the context, this will execute all of the queued up commands.
context.Submit();
```

这会把对象渲染到当前绑定的渲染目标。你可以使用一个command buffer来切换渲染目标。