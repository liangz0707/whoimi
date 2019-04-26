# HDRP自定义后处理

在HDRP当中， commandbuffer无法通过AddCommandBuffer的方式被摄像机调用，所以需要使用别的方法。

目前有两种方式可以插入自己的渲染内容：**重写渲染流程**和**扩展PPV3代码**。

## 重写渲染流程

通过重写HDAdditionalCameraData的CunstomRender方法。

重写后的CustomRender方法会在HDRenderPipeline当中被调用。

调用方式如下：

```c
protected override void Render(ScriptableRenderContext renderContext, Camera[] cameras)
{
    ...
	// Culling loop
    foreach (var camera in cameras)
    {
        if (camera == null)
            continue;
        
        ...
        // 如果当前摄像机有additionalCameraData类型的脚本，并且脚本当中的CustomRender被重写，那么就会走自定义的摄像机流程。
        if (additionalCameraData != null && additionalCameraData.hasCustomRender)
        {
            skipRequest = true;
            // Execute custom render
            additionalCameraData.ExecuteCustomRender(renderContext, hdCamera);
        }
        
        ...
    }
    ...
}
```

目前这个方法可以用于自定义整个摄像机的渲染过程。

## 扩展PPV3代码

目前要添加自定义的后处理，就需要扩展PPV3的代码。

通过阅读PPV3的源代码，我们可以仿照他的代码进行自定义的后处理。

### PPV3后处理原理

下面简单的解释一下新版PPV3实现的过程。

PostProcessSystem.cs是整个后处理的处理过程。

其中下面的方法是在HDRenderPipeline当中调用的：

```c
public void Render(CommandBuffer cmd, HDCamera camera, BlueNoise blueNoise, RTHandle colorBuffer, RTHandle afterPostProcessTexture, RTHandle lightingBuffer, RenderTargetIdentifier finalRT, RTHandle depthBuffer, bool flipY)
{}

```

也就是说后处理直接被HDRP的渲染流程集成了，无法再通过独立的代码执行。调用位置如下。

```c
 ...
     // Set the depth buffer to the main one to avoid missing out on transparent depth for post process.
     cmd.SetGlobalTexture(HDShaderIDs._CameraDepthTexture, m_SharedRTManager.GetDepthStencilBuffer());

    // Post-processes output straight to the backbuffer
    m_PostProcessSystem.Render(
        cmd: cmd,
        camera: hdCamera,
        blueNoise: m_BlueNoise,
        colorBuffer: m_CameraColorBuffer,
        afterPostProcessTexture: GetAfterPostProcessOffScreenBuffer(),
        lightingBuffer: null,
        finalRT: destination,
        depthBuffer: m_SharedRTManager.GetDepthStencilBuffer(),
        flipY: flipInPostProcesses
                );

    StopStereoRendering(cmd, renderContext, hdCamera.camera);
}

```

在后处理的Render方法当中，主要通过CommandBuffer进行渲染控制，具体的渲染方式有3种：

```c
1.cmd.DispatchCompute // 通过ComputeShader直接写入对应像素。
2.HDUtils.DrawFullScreen(); 
// 通过绘制全屏的颜色，底层原理是cmd.DrawProcedural
3.cmd.Blit
```

理论上可以使用CommandBuffer的所有方法，具体还没有测试，我们可以根据需要使用不同的方式，进行后处理。

### 自定义后处理

在后处理的Render方法中可以搜索下面的注释：

```c
// TODO: User effects go here
```

我猜测这个是Unity给自定义后处理留的位置。

我们可以插入代码：

```c
// TODO: User effects go here
{
    // 开启Profiler
    using (new ProfilingSample(cmd, "UserEffect"))
    {
        // 从缓存池中获取一个渲染RT：destination。
        var destination = m_Pool.Get(Vector2.one, k_ColorFormat);
        
        // 设置材质参数：m_CustomMaterial需要提前设置好，是我们的后处理材质。
        m_CustomMaterial.SetTexture(HDShaderIDs._InputTexture, source);
        
        // 调用工具方法将结果绘制到destination当中。
        HDUtils.DrawFullScreen(cmd, camera, m_CustomMaterial, destination);
        
        // 将渲染目标destination回收到缓存池，并把结果还原到source用于接下来的计算。
        PoolSource(ref source, destination);
    }
}
```

其中source是上下文已经设置好的输入贴图，会在接下来继续处理。

所以我们要做的就是操作source，并且输出source。

这个过程相当于：

```c
void Render(RT source, RT destination)
{
	cmd.Blit(source, destination,m_CustomMaterial);
}
```

注：

1. 另外还需要判断Scene视图等操作。

2. 同时还要注意DrawProcedural的Vertex部分要特殊处理。