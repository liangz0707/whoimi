主要介绍unity遮挡剔除功能

#1.Unity build-in插件：Umbra

##1.1具体原理

[原文链接](https://blogs.unity3d.com/cn/2013/12/02/occlusion-culling-in-unity-4-3-the-basics/)：

​	Umbra’s occlusion culling process can be roughly divided into two distinct stages. In the editor, Umbra processes the game scene so that visibility queries can be performed in the game runtime, in the player. So first, Umbra needs to take the game scene as its input and **bake** it into a lightweight data structure. During the bake, Umbra first **voxelizes** the scene, then groups the voxels into cells and combines these cells with **portals**. This data, in addition to a few other important bits is referred to as **occlusion data** in Unity.

​	In the runtime, Umbra then performs **software portal rasterization** into a **depth buffer**, against which **object visibility** can be tested. In practice, Unity gives Umbra a camera position, and Umbra gives back a list of visible objects. The visibility queries are always **conservative**, which means that false negatives are never returned. On the other hand, some objects may be deemed visible by Umbra even though in reality they appear not to be.	

## 1.1 编辑阶段

### 1.1.1 **voxelizes** the scene：

​	使用八叉树将整个场景体素化。每一个正方体体素顶点都会对齐到世界坐标。根据场景中所有需要进行剔除的物体（被Culling Area包裹和被设置为Occlude或（和）Occluder的物体）会计算整个八叉树对场景的覆盖范围，然后细分，。

### 1.1.2 groups the voxels into cells：

​	将空的体素Cell合并成一个大的空的正方体。

### 1.1.3 combines these cells with **portals**

​	将所有的Cell合并成一portals，根据Occlusion Culling视图可以看出来，portal实际上是包裹各个Cell边界的长方体。只是包裹Cell正方体的一个面的长方体，而不是包裹整个Cells。现在得到的只是一个场景的划分数据。

### 1.1.4 摄像机

​	前面完成了场景划分，接下来需要进行摄像机可行区域设置，也就是需要接一下occlusion area把所有的摄像机可行区域覆盖，同时需要标记为is view area。只有这些区域才会得到正确的摄像机剪裁结果，同时只会为这些区域计算遮挡。通过实验可以看到结果：设置一个场景，和唯一一个oa，并且标记为is view Volume随着这个取悦变大生成的数据也会越来越大。

​	其他区域一般情况下也可能进行遮挡剔除。当时查询数据还是不准确的，很可能产生不能确定的结果。

### 1.1.5 区域划分：occlusion portal 和occlusion area

op有一个用处，就是统一的控制其包裹的所有occlusion static 物体是否显示。不会印象内部的动态物体。

oa有两个用处：

1. 勾选了 is view area 用来标记是否是摄像机可行区域。
2. 没有勾选is view area 用来包裹一个区域内的非occlusion static 物体，如果这个area被完全遮挡，内部的物体也会被隐藏。

### 1.1.6 异常情况

静态物体和摄像机之间 不能隔着 关闭的op 不然会出错。

编辑阶段就会产生上述内容。

## 1.2 运行阶段

游戏运行阶段就是直接根据portal来计算这个Cell当中有没有内容被遮挡。

具体方法就是计算渲染所有portal后的深度，和每一个cell的深度。判断cell当中的物体是否需要被去掉。