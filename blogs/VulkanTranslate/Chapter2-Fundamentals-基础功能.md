# Chapter2 Fundamentals 基础功能

​	这个章节主要介绍了基础概念：包括Vulkan架构、扩展模型、API语法、管线配置、数字表示方法、状态和状态查询、以及不同类型的对象和Shader。这部分提供了一个框架用于解释说明文档剩下部分中更加具体的命令和表现。

## 2.1 主机和设备环境

Vulkan说明文件假设并且需要：主机设备满足下面的要求以支持Vulkan的实现：

* 主机必须支持8,16,32,64位的有符号以及无符号整型，并且可以按照字节（byte）寻址。

* 主机必须支持32-和64-位的浮点数类型，并且满足后续章节的范围和精度的要求。
* 这些数据的字节顺序和表示方式必须和物理设备一致。

> Note
>
> 因为各种类型的数据和结构体主机和物理设备都需要访问，所以两端都需要高效的访问以便保证灵活写入和高性能。

## 2.2 Execution Model 执行模式

这一节说明了Vulkan系统的执行模式。

​	Vulkan会暴露出一个或者多个device，每个device会暴露出一个或多个能够异步工作的queue。同一个device支持的各个队列会划分到不同的family。每一个family支持一种或多种功能，并且应该包含一个或多个有相同特性的queue。同一个family当中queue可以认为是互相兼容的。为某个queue所生产的任务，可以在其所属family中的任何一个queue上执行。本说明文档定义了queue可能具备的四个类型的功能：Graphic，Compute，Transfer 和Sparse memory management。

> Note
>
> 一个device可能报告出多个相似queue的family，而不是报告出这些fafmily的一个或多个的多个成员。这表明，虽然这些family的queue具有类似的能力，但他们之间并不直接相容。（没有很明白。）

设备内存是通过应用程序明确的管理的。每一个设备都可以声明一个或者多个堆用于区分不同区域的内存。内存堆要么是设备的本地内存要么是主机的本地内存，但是对于设备总是可见的。进一步的内存堆的细节是通过在堆上的可用内存类型暴露出来的。例如可用的内存类型有：

* device local：和device直接物理连接的设备内存。
* device local, host visible: 设备的local内存对于主机可见。
* host local ,host  visible : 是在主机的内存对设备和主机都可见。

在其它的架构上，可能有一种堆满足各种用处。

一个Vulkan 应用程序通过提交command buffer来控制所有设备，command buffer能够通过Vulkan库函数的调用记录设备命令。Command buffer的内容依赖与具体的实现和应用程序无关。一旦构造了Comman buffer，就可以进行提交一次或者多次到队列中来执行。多个Command buffer能够通过多线程并行的构建。

Command buffer 提交到不同的队列能够并行的执行，执行顺序彼此没有关联。Command buffer提交到同一个队列表示了一个提交顺序。Command buffer在设备中执行的结果也是异步返回到主机的。一旦Command buffer提交到一个队列，控制流会立即返回到应用程序。应用程序需要关注同步设备和主机、同步各个队列。

### 2.2.1 队列操作 Queue Operation

Vulkan队列提供了一个执行设备引擎的接口。引擎执行的命令在运行之前记录到Command Buffer当中。这些命令伴随queue submission命令提交到一个队列中，进行批量(batches)执行。一旦命令提交到队列，这些命令将会开始并在脱离程序控制的状态下完整的执行，执行的顺序依赖于一些隐式和显式的排序约束。

任务一般会通过队列提交命令提交到队列，这些命令一般形如`vkQueue*`(例如：`vkQueueSubmit`, `vkQueueBindSparse`)，并且可以接受一组等待信号量（信号被激活时才会开始执行）和一组激发的信号量（人物完成或会激活对应的信号）。信号量的处理和任务本身都属于队列操作。

在不同队列上的队列操作并没有隐式的顺序限制，可能以任何顺序执行。不同队列间同步可以通过semaphores（通常用于GPU-GPU指令同步）和fences（通常用语CPU-GPU同步）控制。

提交到一个队列中的Command buffer的执行顺序遵循提交顺序和隐式顺序的约束，其他情况则可能同时执行或者乱序执行。针对单个队列的其他类型的批处理和队列提交（例如：稀疏内存绑定），彼此之间没有隐式顺序约束。附加的批处理和队列提交显式顺序约束可以通过信号量semaphores（通常用于GPU-GPU指令同步）和fences（通常用语CPU-GPU同步）控制。

在semaphores或fences被激发之前，可以确定的是：之前被提交的队列操作已经完成了，且随后的队列操作可以继续进行内存的写入。等待一个被激活的semaphore或 fence可以保证之前的写入的内容是可用的，并且是对之后的命令可见（也就是说等待的semaphore或 fence完成后，接下来就能获取之前写入的内容。）

对于Command Buffer是没有顺序约束的，包括相同或者不同类型的Primary Command Buffer的batches或者submissions，以及Primary Command Buffer和Second Command Buffer之间。也就是说，在任何的 semaphore或者fence操作之间，提交一系列的Command Buffer（包括Second Command Buffer） 和在一个单独的Primary Command Buffer中记录这些命令然后提交是一样的（除非当前的状态被重置了）。想要显示的控制这些顺序，可以通过显示信号量。

在同一个Command Buffer当中的命令遵循一些隐式的顺序约束，但是只约束了一部分操作。

> ​	Note
>
> 提交到一个队列当中的任务有很大的自由度进行重叠执行。这是由于Vulkan本身的管线和并行策略决定的。

记录在Command Buffer当中的命令包括：执行操作（例如：draw，dispatch，clear，copy，query/timestamp， begin/end subpass，）状态设置操作（例如：bind pipeline，descriptor sets, buffer, set dynamic state,push constants, set render pass），同步操作（例如：set/wait events, pipeline barrier, render pass/subpass dependencies）一些命令执行多次。状态设置会更新CommandBuffer当前的状态。一些命令的执行基于当前的状态。

设备端的队列执行和主机端队列提交是异步执行的。当命令缓冲区被提交到队列后控制流马上就退回到应用程序了。 应用程序必须按需求在主机端和设备端进行同步操作。



## 2.3 对象模型 Object Model

devices，queue和其他的实例都是通过Vulkan object表示的。在API级别，所有的对象都通过handle来引用。有两种类型的handle：可分发的与不可分发的。*可分发的* handle是不可见类型数据的指针。这个指针可被layers使用，被当作拦截API命令的一部分， 每一个API命令都接受一个可分发类型的handle作为第一参数。 每一个不可分发类型的对象必须在其生命周期内有唯一一个handle值。

_不可分发的_handle类型是64位整型类型，其含义是Vulkan实现决定的，他可能会把对象信息直接包含到handle里，而非通过指向一个数据结构。不可分发类型的对象，对于不同的类型或者同一个类型可能没有唯一的handle值（handle可能重复）。 如果handle值不唯一，那么销毁任何一个handle时：要保证不能造成其他类型的、相同handle值的对象变得无效，也要保证如果用同一个handle创建了多个对象，并且创建次数多余销毁次数，也不能让这个销毁操作导致有相同handle值的对象失效。

所有通过`VkDevice` （用 `VkDevice`作为第一个参数 )的命令创建的对象都是该设备私有的，必须不能被其他设备使用。

### 2.3.1 对象的生命周期Object Lifetime

对象都是通过形如`vkCreate*` and `vkAllocate*` 的命令创建或者分配的。 一旦一个对象被创建或者分配，它的结构就被认为是不变的，即使对象的内容可以自由改动。 对象通过形如`vkDestroy*` 和 `vkFree`* 的命令来销毁或释放。

被分配（而不是创建）的对象从一个已存在的池子对象或者内存堆中获取资源，当被释放时会把资源归还给该池子或者堆。对象的创建和销毁在运行时通常是低频操作， 对象的分配或者释放可能是高频的。对象池帮助调节分配和释放的性能提升。

应用程序有责任跟踪Vulkan对象的生命周期，并且不能销毁正在被使用的对象。

属于应用程序端的内存在提交给Vulkan命令的时候会即刻被接管。在Vulkan命令执行结束之后一定会返还给应用程序。应用程序可以在需要占用这些内存的命令返回之后，再修改和释放这些内存。

下面的对象会在传递给Vulkan命令之后被消耗掉，并且不能再被由它创建的对象访问。这种对象在任何用到他们的API命令执行期间不能被销毁。

• VkShaderModule
• VkPipelineCache



