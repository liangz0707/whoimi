# 谈谈那些early_depth_testing技术（草拟）



## Early Z（Immediate Mode Renderers (IMRs) ：Nvidia ： GeForces and Radeons）



在OPengl 4.5 强制开启下面是原文。

## Optimization

Early fragment tests, as an optimization, exist to prevent unnecessary executions of the [Fragment Shader](https://www.khronos.org/opengl/wiki/Fragment_Shader). If a fragment will be discarded based on the [Depth Test](https://www.khronos.org/opengl/wiki/Depth_Test) (due perhaps to being behind other geometry), it saves performance to avoid executing the fragment shader. There is specialized hardware that makes this particularly efficient in many GPUs.

The most effective way to use early depth test hardware is to run a depth-only pre-processing pass. This means to render all available geometry, using minimal shaders and a rendering pipeline that [only writes to the depth buffer](https://www.khronos.org/opengl/wiki/Write_Mask). The [Vertex Shader](https://www.khronos.org/opengl/wiki/Vertex_Shader)should do nothing more than transform positions, and the [Fragment Shader does not even need to exist](https://www.khronos.org/opengl/wiki/Fragment_Shader#Optional).

This provides the best performance gain if the fragment shader is expensive, or if you intend to use multiple passes across the geometry.

### Limitations

The [OpenGL Specification](https://www.khronos.org/opengl/wiki/OpenGL_Specification) states that these operations happens after fragment processing. However, a specification only defines apparent behavior, so the implementation is only required to behave "as if" it happened afterwards.

Therefore, an implementation is free to apply early fragment tests if the [Fragment Shader](https://www.khronos.org/opengl/wiki/Fragment_Shader) being used does not do anything that would impact the results of those tests. So if a fragment shader writes to gl_FragDepth, thus changing the fragment's depth value, then early testing cannot take place, since the test must use the new computed value.

**Note:** Do recall that if a fragment shader writes to gl_FragDepth, even conditionally, it must write to it at least once on all codepaths.

There can be other hardware-based limitations as well. For example, some hardware will not execute an early depth test if the (deprecated) alpha test is active, as these use the same hardware on that platform. Because this is a hardware-based optimization, OpenGL has no direct controls that will tell you if early depth testing will happen.

Similarly, if the [fragment shader discards the fragment with the discard keyword](https://www.khronos.org/opengl/wiki/Fragment_Shader#Special_operations), this will almost always turn off early depth tests on some hardware. Note that even *conditional* use of discard will mean that the FS will turn off early depth tests.

**Note:** All of the above limitations apply only to early testing as an optimization. They do not apply to anything below.

## Explicit specification

|                    |                                                              |      |
| :----------------- | ------------------------------------------------------------ | ---- |
| Core in version    | 4.6                                                          |      |
| Core since version | 4.2                                                          |      |
| Core ARB extension | [ARB_shader_image_load_store](http://www.opengl.org/registry/specs/ARB/shader_image_load_store.txt) |      |

More recent hardware can force early depth tests, using a special fragment shader layout qualifier: 

```
layout(early_fragment_tests) in;
```

This will also perform early stencil tests.

There is a caveat with this. This feature *cannot* be used to violate the sanctity of the depth test. When this is activated, any writes to gl_FragDepth will be *ignored*. The value written to the depth buffer will be exactly what was tested *against* the depth buffer: the fragment's depth computed through rasterization.

This feature exists to ensure proper behavior when using [Image Load Store](https://www.khronos.org/opengl/wiki/Image_Load_Store) or other [incoherent memory writing](https://www.khronos.org/opengl/wiki/Incoherent_Memory_Access). Without turning this on, fragments that fail the depth test would still perform their Image Load/Store operations, since the fragment shader that performed those operations successfully executed. However, with early fragment tests, those tests were run before the fragment shader. So this ensures that image load/store operations will only happen on fragments that pass the depth test.

Note that enabling this feature has consequences for [the results of a discarded fragment.](https://www.khronos.org/opengl/wiki/Fragment_Discarding)

## HSR（TBDR：Imagination：PowerVR）



1. 通常在TBDR(iphone , PowerVR)架构的手机GPU当中，有HSR技术，所以不需要Early-Z。并且HSR比Early-Z的效果更好，完全没有OverDraw。



## Forward Pixel Kill（TBD：ARM Mali GPUs from Mali-T62X and T678 onwards ）

在Opengl ES3.1 当中规定了 Early z的说明。

**Early Z + FPK.**

材料1:https://community.arm.com/developer/tools-software/graphics/b/blog/posts/killing-pixels---a-new-optimization-for-shading-on-arm-mali-gpus

材料2：Forward Pixel Kill专利书

材料3：Arm® Mali™ GPU Best Practices

重要的一句话：

**All Mali GPUs since the Mali-T620 GPU includes the FPK optimization. FPK provides automatic hidden surface removal of fragments that are occluded, but early-zs testing does not kill. This is due to the use of a back-to-front render order for opaque geometry. However, do not rely on the FPK optimization alone. An early-zs test is always more energy-efficient, consistent, and works on older Mali GPUs that do not include hidden surface removal.**



联想 ：**FPK对CLIP像素是否有效?**



## Qualcomm Adreno 

在Opengl ES3.1 当中规定了 Early z的说明。

**Early Z rejection** 

Adreno 3xx and 4x can reject occluded pixels at up to four times the drawn pixel fill rate。To get maximum benefit from this feature, QTI recommends drawing a scene with primitives  sorted out from front-to-back; i.e., near-to-far. This ensures that the Z-reject rate is higher for the  far primitives, which is useful for applications that have high-depth complexity.