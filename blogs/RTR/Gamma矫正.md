# Gamma矫正

之前总结了UnityGamma矫正相关的内容。

这里进行更加详细的解释。

主要的参考资料：

线性空间：[GPU Gems:Chapter 24. The Importance of Being Linear](https://developer.nvidia.com/gpugems/GPUGems3/gpugems3_ch24.html)

HDR：[HDR标准](https://www.eizo.com.cn/global/library/management/ins-and-outs-of-hdr/index1.html)

## Gamma存在的原因

**一切的起因：CRT显示器对于颜色值得的响应并不是线性的** + **人眼对暗部更明显**。

1.  阴极射线管在将电压转换成光强的过程中并不表现出线性关系，也就是说，计算机当中保存的0.5的颜色值，输出到CRT显示器上会变成0.25左右，整个颜色会变暗。即使在后来使用了LCD显示器，也需要进行Gamma矫正。Gamma矫正会将图像亮。$$ C_{Gamma} = C^{1/2.2}$$.
2.  Gamma矫正会将图像变亮的同时，还会让数值的更大的范围来描述颜色的暗部。恰巧人眼对暗部更加敏感，所以我会直接将Gamma矫正后的图像进行保存。再到后来就是，拍摄设备+存储+显示全部都是用Gamma矫正的图像。

## 显示器与Gamma

**根据上面提到的，一般的CRT显示器和LCD显示器都会需要一个Gamma矫正才能正常显示。**

**在使用HDR显示器上，同样会有一条Gamma曲线（与CRT不同的是他的分布在整个亮度区间，而不是0到1），这个是为了能够配合人类的视觉。**

人类眼睛所能感知的最暗和最亮物体之间的差异范围（动态范围）通常为  $$ 10^{12} $$ ，传统的SDR范围是 $$  10^{3} $$ ,HDR的范围 $$ 10^{6} $$ . 

为了正确显示HDR图像，仅仅提高亮度是不够的 - 以与人类视力相匹配的方式显示色彩和色调至关重要。 色彩和色调受每个输入和输出设备具有的称为伽玛的输入 - 输出特性的影响。

## 渲染与Gamma

渲染过程中，涉及到两个问题：

1.  如何使用sRGB图像。
2.  如何输出正确的图像格式。

上面两个问题是独立的，互相之间没有关系，下面进行分别讨论。

### sRGB图像的使用

一般图像都保存在sRGB空间，我们计算光照必定要在线性空间下。

有两种方式得到：

1.   一种是通过图像API支持，例如：opengl在设置纹理是可以吧纹理标记，这样Shader在采样的时候就会自动获取到一个线性空间的颜色。这部分需要API的支持，对于手机来说一个Opengl ES 3.0 和Vulken 都支持。
2.  另一种方式是采样之后计算平方。

在Gems里面提到，对于手动计算，可能在mipmap和filtering步骤有一些问题。

### 输出正确的图像

这部分就需要判断设备类型：

如果支持FrameBuffer设置成sRGB那么就可以直接输出线性颜色。任何被Shader返回的值都会在保存之间进行Gamma矫正。如果开启了线性混合，那么已经保存的值会被返回到线性空间进行混合，然后在进行Gamma矫正之后保存。Alpha的值不会参与上面的操作操作。

如果不支持sRGB类型的Frame Buffer。就需要进行手动矫正。但是这个时候Blend操作我们是无法控制的，所以Blend的结果是用Gamma空间直接混合的错误结果：

```c
float3 finalCol = do_all_lighting_and_shading();
float pixelAlpha = compute_pixel_alpha();
return float4(pow(finalCol, 1.0 / 2.2), pixelAlpha);
// Or (cheaper, but assuming gamma of 2.0 rather than 2.2)
return float4( sqrt( finalCol ), pixelAlpha );
```

如果使用了HDR显示器，那么可以不做这个操作。

### 中间结果

只要记住如果用了sRGB的FrameBuffer，会自动编码解码，计算时统一即可。

### HDR

如果使用了HDR，那么可以参考Unity的制作方案。

如果没有开启HDR，原本颜色的叠加都会使用Gamma矫正。这是因为精确度低，经过Gamma编码可以让更大的区间保存暗部。在大部分8bit的颜色格式中，大部分图形API都提供了sRGB版本，就是为了尽可能提升分辨率。

开启HDR表示可以使用超过255的颜色，此时所有的帧缓存会变成ARGB32.或者ARGBHalf。此时分辨率足够高，所有的线性颜色进行不断的叠加，不在需要特殊编码，而是在这一帧完全处理完成之后，在进行下面的处理。

当所有的混合和或处理计算结束，Gamma矫正才会再次执行。



















