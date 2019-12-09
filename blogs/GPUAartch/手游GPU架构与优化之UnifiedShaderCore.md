# 手游GPU架构与优化之UnifiedShaderCore

在手游优化的过程中发现Metal当中出现了VSShder饱和PSShader等待的情况。

这主要是由于当前的渲染是VS密集型的。

但是在最新的GPU中已经使用了Unified Shader Cores。VS和PS应该可以进行统一的线程分配。

为了了解这个问题进一步研究了手机架构。

下面是ARM的图片，首先可以发现UTGARD基础架构的GPU还是Separate Shader Cores.也就是说会出现互相等待的情况。

![img](https://images.anandtech.com/doci/10375/4.%20Tech%20Day%20Bifrost%20FINAL-04_575px.png)

使用ARM Mali芯片的主要是 联发科、华为海思、三星。

开始使用USA(Unified Shading Architecture a hardware design by which all shader processing units of a piece of graphics hardware are capable of handling any type of shading tasks. )的：

ATI架构在Radeon HD2000(PC)

Nvidia从GeForce 8系列(PC) 使用Tesla架构

高通Adreno 200 series  [wiki介绍](https://en.wikipedia.org/wiki/Adreno)

Mali Midgrad

PowerVR SGX







