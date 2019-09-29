# 关于Cuda\Shader中if语句执行原理

目前大部分人认为：shader当中动态if判断性能很低，只是用来做条件判断的，要避免使用。实际上if语句是可以用来做性能优化的。

## 参考资料

主要来自于几个文档：

0.  http://haifux.org/lectures/267/Introduction-to-GPUs.pdf 直接说明了GPU分支是如何工作的。

1.  第一个是GPU Gems 2当中提到的流控制[Chapter 34. GPU Flow-Control Idioms](https://developer.nvidia.com/gpugems/GPUGems2/gpugems2_chapter34.html)

里面提到了对于Z-cull来说分支和warp分布的关系。

2.  和GPU分支优化相关的文章：[Reducing_branch_divergence_in_GPU_programs]，上面也提到了分支发散，并且提到了：

    In the presence of a datadependent branch that causes diﬀerent threads in the same warp to follow diﬀerent paths (also known as branch divergence), the warp serially executes each branch path taken, disabling threads that are not on that path.

    说明如果一个Warp没有进行分支发散，是所有warp只走一个分支的。

3.  手机部分GPU架构主要是高通和AMD，目前找到了高通的资料：[Qualcomm® Snapdragon™ Mobile Platform OpenCL General Programming and Optimization]([file:///C:/Users/liangzhe/Desktop/80-nb295-11_a.pdf](file:///C:/Users/liangzhe/Desktop/80-nb295-11_a.pdf))

    这里面提到了避免分支发散（Avoid branch divergence）可以从文本中推测出，手机的GPU架构和PC是一样的，有同Warp优化功能，并不会两个分支都走。

4.  《GPU编程与优化》当中提到没有分支的Warp可以省略空跑分支。

## 例子

更直观的解释参考[GPU分支语句性能](GPU分支语句性能.md)