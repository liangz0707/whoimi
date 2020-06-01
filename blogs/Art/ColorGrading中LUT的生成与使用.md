# ColorGrading中LUT的生成与使用

场景的LUT贴图有两种生成方式：

1.  通常DaVinCi生成的是.cube文件，需要用cube文件转换。
2.  直接在PS当中调整。

## PS直接调整

对于用ps直接调整来说，只要把参考图片和Natural LUT一起调整即可，最终Natural LUT的结果就是我们要的LUT。

## cube文件转换

打开PS，加载默认的LUT文件（就是netrual lut，不对图像产生影响的lut）。

![1590739624304](ColorGrading中LUT的生成与使用/1590739624304.png)

确定颜色模式为RGB

![1590739649754](ColorGrading中LUT的生成与使用/1590739649754.png)

开启颜色查找功能：

![1590739676967](ColorGrading中LUT的生成与使用/1590739676967.png)

使用默认选项点击确认，得到lut图层：

![1590739706978](ColorGrading中LUT的生成与使用/1590739706978.png)

![1590739721058](ColorGrading中LUT的生成与使用/1590739721058.png)

下面是LUT的图层控制面板：

![1590739883952](ColorGrading中LUT的生成与使用/1590739883952.png)

配置完成之后就得到了修改好的lut，保存成非压缩的png即可：

![1590739972583](ColorGrading中LUT的生成与使用/1590739972583.png)



## 下面是两种Natural Lut

64*64![neutral-lut](ColorGrading中LUT的生成与使用/neutral-lut.png)



32*32

![reshade-neutral-lut-768x24](ColorGrading中LUT的生成与使用/reshade-neutral-lut-768x24.png)