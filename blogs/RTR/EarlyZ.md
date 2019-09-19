# Early Z

## 优化原理

[OpenGL官方文档对EarlyZ的解释](https://www.khronos.org/opengl/wiki/Early_Fragment_Test)

原本的深度测试是在Fragment Shader计算之后的，如果不通过深度测试，这部分Pixel计算就是浪费的。

优化方式就是在Fragment Shader处理之前，先进行深度测试，可以避免过多的Pixel计算。

使用深度检测是方便的（硬件直接支持），这个优化，假设你的Pixel计算没有对深度进行任何修改，也就是说在Fragment Shader之前和之后进行深度检测的结果是一样的。（Therefore, an implementation is free to apply early fragment tests if the [Fragment Shader](https://www.khronos.org/opengl/wiki/Fragment_Shader) being used does not do anything that would impact the results of those tests. ）



## 限制条件

1.  这个是硬件支持的功能，图形API无法决定是否进行。（原文：Because this is a hardware-based optimization, OpenGL has no direct controls that will tell you if early depth testing will happen.）

2.  如果进行了Alpha Clip或者深度写入，会造成EarlyZ失效。（原文：Similarly, if the [fragment shader discards the fragment with the discard keyword](https://www.khronos.org/opengl/wiki/Fragment_Shader#Special_operations), this will almost always turn off early depth tests on some hardware. Note that even *conditional* use of discard will mean that the FS will turn off early depth tests.）

## 优化方式

### Z-Pre-Pass

Early-Z的实现，主要是通过一个Z-pre-pass实现。简单来说：对不透明物体：

1.  首先用简单shader进行渲染，这个shader**不写颜色缓冲区，只写深度缓冲区**。
2.  **关闭深度写入，开启深度测试**，用正常的shader进行渲染。

The most effective way to use early depth test hardware is to run a depth-only pre-processing pass. This means to render all available geometry, using minimal shaders and a rendering pipeline that [only writes to the depth buffer](https://www.khronos.org/opengl/wiki/Write_Mask). The [Vertex Shader](https://www.khronos.org/opengl/wiki/Vertex_Shader) should do nothing more than transform positions, and the [Fragment Shader does not even need to exist](https://www.khronos.org/opengl/wiki/Fragment_Shader#Optional).

This provides the best performance gain if the fragment shader is expensive, or if you intend to use multiple passes across the geometry.

### 类似Unity的渲染排序 Render Sort

我们常用的深度排序（从前往后渲染）渲染不透明物体，就是为了利用Early Z，如果没有Early Z那么深度排序就没有意义（并且还会浪费CPU）。（Unity官方文档原文：Spend lots of CPU cycles to do occlusion culling and better sorting (to take advantage of Early Z-cull).）

## **Alpha Test（Discard）在移动平台消耗较大的原因**

跟上面讲到的Early-Z有关。

1.  正常情况下，不管是否是开启深度写入或者深度测试，这个面片的光栅化之后对应的像素的深度值都可以在Early-Z（Z-Cull）的阶段判断出来。
2.  如果开启了Alpha Test（Discard），导致Early-Z失效。即使被遮挡也一定会进行Pixel计算！ 与之相对的Alpha Blend却可以进行正确的Early计算，所以会有人讨论Alpha Test和Alpha Blend的区别。[CryEngine官方解释](https://docs.cryengine.com/display/SDKDOC2/Rendering+Performance+Guidelines)文档指出，使用Alpha Testing会导致跳过对Opaque物体的优化。Alpha Blend会进行Frame Buffer的w/o操作，开销也比较到。所以他们性能开销的痛点是不一样的。

Unity官方文档的描述：

The fixed-function [AlphaTest](https://docs.unity3d.com/Manual//SL-AlphaTest.html) - or its programmable equivalent, `clip()` - has different performance characteristics on different platforms:

-   Generally you gain a small advantage when using it to remove totally transparent pixels on most platforms.
-   However, on PowerVR GPUs found in iOS and some Android devices, alpha testing is resource-intensive. Do not try to use it for performance optimization on these platforms, as it causes the game to run slower than usual.

## 其他的资料

这个[文章](https://zhuanlan.zhihu.com/p/33127345)提到了像素并行处理的问题，这个问题在使用Unity的Render Sort策略下是有问题的。但是如果使用Z-Pre-Pass策略是没有问题的。

里面提到的更详细的内容都是猜测不是很准确，需要进一步研究一些手机架构。

## 总结

他的核心：Early Z的优化和优化失效的问题。































​	