# 介绍caffe的安装过程

> 首先需要从github上clone最新的代码

> 官方提到的安装方式有两种：cmake和make两种方式

> 建议使用官方方式，因为使用example的时候不太一样

[Github Caffe](https://github.com/BVLC/caffe)


[Caffe 官方网站](http://caffe.berkeleyvision.org/)

在安装的过程当中遇到了两种问题

1. (30 vs.0)这个问题主要是由于cuda版本和显卡驱动的版本不匹配。通过官方下载最新的显卡驱动就能解决
2. (8 vs .0)这个问题主要是由于，显卡无法容纳batchsize，调小就可以了。
3. 还有可能提及到libopencv*的错误，这个需要是由于没能够找到合适的opencv，看情况重新编译，或者制定目录即可。

[back to list](index.md)




