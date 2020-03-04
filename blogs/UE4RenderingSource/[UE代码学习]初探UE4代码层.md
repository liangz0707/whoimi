# 初入UE4代码层

##  阅读入口

本篇笔记主要记录了从零开始如何进入UE4的代码层，开始编码和Shader阅读。

首先需要对Shader和渲染过程有一定了解。

最开始需要了解的是两个关键的文件：

**BasePassPixelShader.usf**：基础的Deferred渲染的Shader文件。

**DeferredShadingRenderer.cpp**：控制渲染流程的代码。

当然我们都知道UE4有两套渲染管线：Deferred和针对手机的Mobile，所以对应的文件：

**MobileBasePassPixelShader.usf**

**MobileShadingRenderer.cpp**

上边两组文件的命名和编码有足够的自解释性，可以作为阅读的起始点（**Deferred那一组**）。通过阅读可以了解到

1. UE4基础的渲染过程.
2. UE4Shader的语法、编码方式、组织方式（Shader文件的包含关系、文件结构等）

推荐反复的推敲阅读，虽然暂时无法开始编码工作，但这两个文件可以对应到Unity或者是其他图形渲染引擎的相应部分。

**我也是以这两个文件为基础不断的进行延展的代码阅读。**

下面对这两部分进行导读：[UE4Shader代码入门导读](UE4Shader代码初探.md)和[UE4渲染流程代码入门导读](UE4渲染管线代码初探.md)





