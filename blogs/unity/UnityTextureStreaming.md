# Texture Streaming

[官方文档](https://docs.unity3d.com/Manual/TextureStreaming.html)

Texture Streaming 主要为了解决：

​	当导入贴图的时候，会生成mipmap。导致贴图和mipmap加起来占用的空间变大。

例如：大部分物体离得比较远，使用了贴图的LOD4。但是这个时候LOD0-3都会进入内存。

Texture Streaming的作用就是在使用贴图的时候，只向内存传递需要的mipmap级别的贴图，降低内存占用，不过需要额外的CPU计算。

