# 一个三角形在GPU中的一生

这篇笔记总结自[Life of a triangle - NVIDIA's logical pipeline](https://developer.nvidia.com/content/life-triangle-nvidias-logical-pipeline) 和知乎文章[渲染优化-从GPU的结构谈起](https://blog.uwa4d.com/archives/USparkle_GPU.html)。

主要记录了GPU硬件是如何详细的工作的。另外附加了一些笔记和自己的理解。

## 前言

先进的Fermi架构已经发布十年了（2010年左右发布），现在是时候更新它的基础图形架构了。

Fermi是Nvidia的第一个实现了完全可扩展的图形引擎（fully scalable graphics engine的概念请查阅wiki，有说明），在Kepler和Maxwell架构当中也能找到他的影子。

下面的文章，特别是下面的“精简的管道知识”图片，应该作为各种学习材料的。这篇文章主要从图形的角度讨论GPU是如何工作的，尤其是一些原则，比如：着色程序代码是如何执行的，对于计算来说是一样的。

## GPU管线架构

下面是一张涵盖了整个GPU架构的图片，后面会详细分开讲解。

![NVIDIA's logical pipeline](%E4%B8%80%E4%B8%AA%E4%B8%89%E8%A7%92%E5%BD%A2%E5%9C%A8GPU%E4%B8%AD%E7%9A%84%E4%B8%80%E7%94%9F/fermipipeline.png)

## gpu是超级并行工作分发器

在图形中，我们必须处理数据放大（很少的数据提交会产生很大的变化的工作负载：指的是整个图形流水线处理一个很少的渲染模型）。他的复杂性来源于：

> * 每个drawcall可以生成不同数量的三角形。
> * 裁剪后的顶点数量与三角形发生变化。
> * 经过背面和深度剔除后，并不是所有的三角形都需要屏幕上的像素。
> * 一个屏幕尺寸大小的三角形的意味着它需要数百万像素或者根本不需要（被挡住不渲染）。

因此，现代gpu让它们的基本类型(三角形、线、点)遵循**逻辑管道**，而不是**物理管道**。在G80的统一架构(DX9、ps3、xbox360)之前的日子里，流水线在芯片上表现为不同的**硬件模块**，工作一个接一个地通过这些硬件模块串行执行。（**不同的物理硬件处理不同的数据**）。G80能够让顶点和片段着色器重用一些硬件单元，但是它仍然有很多用于图元/栅格化等阶段的硬件，是串行执行的。有了Fermi，管道变得完全并行化了，这意味着通过重用芯片上的多个引擎（不同的功能组件）实现了**逻辑管道**(我们后面要讲的三角形处理步骤)。

假设我们有两个三角形：A 和B。他们的不同部分可能在不同的逻辑阶段。A已经完成了空间转换操作需要进行光栅化。那么，A的一部分可能已经完成了Pixel-shader的执行，一部分可能被z-cull剪裁掉了，还有一部分可已经写入到了frame buffer，还有一部分可能在等待执行。除此之外，我们可以拉取三角形B的顶点数据。因此，虽然每个三角形都必须经过所有的逻辑步骤，但它们可能处在其生命周期的不同步骤中。这些工作会被划分成任务甚至子任务，然后并行执行。每个任务都被调度到可用的资源中，这并不局限于特定类型的任务。

## GPU architecture：基础硬件结构

下图是Maxwell GPU架构，他和Fermi架构基本类似。

* **Giga Thread Engine** ：有一个Giga线程引擎（**Giga Thread Engine** ）来管理所有正在进行的工作。
* **Graphics Processing Cluster**:GPU被划分为多个GPCs(图形处理集群：**Graphics Processing Cluster**)
* 每个都有多个SMs(流多处理器：**Streaming Multiprocessor**)和一个栅格引擎（**Raster Engine**）。
* **Crossbar**:在这个过程中有很多的互连，最明显的是一个**Crossbar** ，它允许各种任务在GPCs或其他功能单元(例如：ROP(**render output unit**))之间迁移。
* **Shader**是在SMs上完成的。SMs它包含许多核心**Cores** ，为线程执行数学操作（**核心之间以SIMD的方式运行**）。一个线程既可以是vertex shader 也可以是fragment shader.
* 这些核心和其他单元由**Warp Schedulers**驱动，**Warp Schedulers**以32个线程为一组进行管理，并将执行的指令交给**Dispatch Units**。
* **代码逻辑**由调度器**Warp Schedulers**处理，而不是在核心**Cores** 内部。**Cores** 只会看到调度程序中的类似“**用寄存器4235和寄存器4234求和，并存储在4230中**”的内容。（也就是说**Cores**一次只能执行一条Shader指令）。
* **一个核心本身是相当愚蠢的，相比之下，一个CPU的核心是相当聪明的。**GPU将智能提升到更高的层次，它可以执行一个整体(或者多个)的工作。(SIMD)
* 在GPU上，这些单元中的数量(每个GPC有多少 SMs，有多少GPC ..)取决于芯片配置本身。
* SMs的设计本身(cores的数量，指令单元，schedulers…)也随着时间一代一代的变化(见第一张图片)，并保证芯片高效运行，它们可以从高端台式电脑扩展到笔记本电脑和移动设备。

![fermipipeline_maxwell_gpu](%E4%B8%80%E4%B8%AA%E4%B8%89%E8%A7%92%E5%BD%A2%E5%9C%A8GPU%E4%B8%AD%E7%9A%84%E4%B8%80%E7%94%9F/fermipipeline_maxwell_gpu.png)

## The logical pipeline 逻辑管线

这里开始介绍上面的物理硬件是如何通过逻辑管线配合起来的。

说明：假设**drawcall**引用了**index buffer**和**vertex buffer**，这些**index buffer**和**vertex buffer**已经被数据填满了，并且位于**GPU**的**DRAM**中，并且只使用了**vertexshader** 和 **pixelshader**。



1. 程序在图形api(DX or GL).中进行提交一个drawcall。这个drawcall将会在随后到达驱动程序（**driver**）。驱动程序会执行一系列验证确保操作合法，然后将指令**以GPU可以读取的编码方式**插入**pushbuffer**。在此时可能产生CPU端瓶颈，这就是为什么程序员需要合理的利用api和技术来利用gpu。（通过GPU功能降低drawcall）。



2. 经过一段时间或显式的调用"flush" 指令后，驱动程序（**driver**）已在**push buffer**中缓冲了足够的任务并且发送给GPU进行处理（有操作系统的参与）。





