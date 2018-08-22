在Unity官方文档的介绍中，SetRenderTarget可以设置渲染目标。

1. 屏幕的渲染目标可以通过Graphics.activeColorBuffer和Graphics.activeDepthBuffer获取。

其中Graphics.activeDepthBuffer当中包括了stencilbuffer和depthbuffer。

2. commandbuffer当中同样可以使用BuiltinRenderTextureType.Color和BuiltinRenderTextureType.depth来访问对应的渲染目标，但是**BuiltinRenderTextureType.depth是不包括stencilbuffer**的。
3. 如果想要复用包含stencilbuffer的Graphics.activeDepthBuffer，是不能通过SetRenderTarget和自定义的ColorBuffer组合的，会报错。**因为screen的buffer和rendertarget的buffer不能混合使用**。
4. 如果只需要复用同一张depth，只需要使用BuiltinRenderTextureType.depth和自定义的colorbuffer即可。
5. 如果需要复用同一张depth/stencil，必须要在一开始修改渲染目标为自定义的RenderTarget，例如：SetRenderTarget(selfColorBuffer, selfDepthBuffer) ,同时selfDepthBuffer的深度位数**必须为24或32**.  如果是16 则不支持stencilbuffer。




关于延迟渲染（deferred）渲染目标（RenderTexture）的问题

1. 延迟渲染的RenderTarget目前没有找到方法设置。
2. 延迟渲染的内容可以通过commandbuffer通过BuiltinRenderTextureType.Gbuffer获取出来。
3. BuiltinRenderTextureType.Gbuffer3当中保存了整个渲染流程的结果，包括了Forwardpass。
4. 在渲染结束后，会将BuiltinRenderTextureType.Gbuffer3当中的内容拷贝到最终渲染目标当中。
5. SetRenderBuffers()中的多个渲染目标并不是Gbuffer，而是从shader直接输出的内容。
6. CameraEvent.beforelighting可以看到Gbuffer3组合了无光照内容和自发光内容和全局光照，还没有环境反射，不过在AfterGbuffer时还没有自发光内容。
7. 延迟渲染的环境反射是在完成渲染目标切换之后，才进行渲染的，比天空球还晚。（前向渲染是伴随物体光照同时计算的）



延迟渲染过程中的深度提取

1. 如果使用BuiltinRenderTextureType.Depth无法获取到stencil纹理，在deferred当中无存在，如果使用会报错：built-in renderTexture type3 not found...
2. deferred过程需要使用BuiltinRenderTextureType.ReolvedDepth，不过使用默认深度和自定义 深度大小很容易不匹配
3. 在ResolvedDepth中存在完整的深度信息。在deferred第一步先计算延迟物体深度（也就是Gbuffer填充阶段），然后计算前向物体深度，当在延迟渲染中设置depthbuffer时，代替的就是ResolvedDepth。
4. ResolvedDepth中的Stencil，延迟光照阶段最多会使用高四位，并且进入前向阶段后不会清零。
5. 在延迟渲染阶段，前向物体深度计算时会在stencilbuffer当中以207为mask写入内容。这表示高两位和第四位是都可以用的。并且Stencil写入和深度写入在同一个阶段。


*注：当使用commandbuffer绘制一般的物体时，目前的CameraEvent都是不包含光照信息的，所以绘制出来的内容始终是黑色的。

 