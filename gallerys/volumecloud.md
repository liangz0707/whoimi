# 地平线（Horizon: Zero Dawn）体积云实现

通过体渲染实现的云。

参考了《地平线》游戏中的算法，在unity当中使用ComputeShader实现。

实现技术包括ToneMapping，RayCasting两部分内容，目前Jitter部分还没有和TAA结合起来，走样效果比较严重。主要复现了核心算法。

性能为2ms在PC上(1070)，需要进一步优化。

目前Unity中实现的效果：

![我的云](img/我的云.png)

地平线文献当中的效果：

![地平线的云](img/地平线的云.png)

