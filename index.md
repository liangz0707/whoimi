
[===== English Part=====](#-english-part)

[===== Chinese Part=====](#-chinese-part)

 [Real Time Rendering](#real-time-rendering)

 [UE4 使用经验](#ue4-使用经验)

 [UE4渲染源代码阅读](#ue4渲染源代码阅读)

 [Unity使用经验](#unity使用经验)

 [Unity 渲染效果](#unity-渲染效果)

 [美术制作方案](#美术制作方案)

 [Unity HDRP 源代码阅读](#unity-hdrp-源代码阅读)

 [Shader 编程](#shader-编程)

 [GPU硬件及架构相关](#gpu硬件及架构相关)

 [杂项](#杂项)

 [我的动态](#我的动态)

# ===== English Part=====

[UE4_Performance_Bottlenecks](blogs/UE4/UE4_Performance_Bottlenecks.md) 

[Building Better 3D Meshes and Textures](blogs/UE4/Building_Better_3D_Meshes_and_Textures.md) 

# ===== Chinese Part=====

## Real Time Rendering

[【精】图像动态范围](blogs/RTR/图像动态范围.md) ：游戏中，Gamma矫正来源、tonemapping来源与动态范围的正确解释。

[【精】GPU渲染优化技术:Inside Geometry Instancing](blogs/RTR/GPU渲染优化技术Inside_Geometry_Instancing.md) 

[主机渲染和优化技术概要](blogs/RTR/主机渲染和优化技术概要.md) 

[Temporal Anti-Aliasing原理与Unity实现](blogs/RTR/TAA.md)

[HDR与ToneMapping理论](blogs/RTR/HDR与ToneMapping理论.md)

[体积云光照部分相关理论](blogs/RTR/体积云.md)

[法线空间转换的问题与解决方案推导](blogs/RTR/法线空间转换.md)

[渲染管线](blogs/RTR/渲染管线.md) 

[BRDF基础原理与推导](blogs/RTR/表面反射模型.md) 

[BRDF常用高光数学模型：G、F、D](blogs/RTR/BRDFSpecular.md)

[蒙特卡洛积分理论](blogs/RTR/蒙特卡洛方法.md)

[重要度采样理论](blogs/RTR/重要度采样.md) 

[IBL光照理论与应用](blogs/RTR/IBL光照理论与应用.md) 

[球面谐波理论与应用](blogs/RTR/球面谐波理论与应用.md) 待完善

[Gamma矫正](blogs/RTR/Gamma矫正.md) 

[Early Z](blogs/RTR/EarlyZ.md)

[手机Tiled-BasedRender和GPU带宽](blogs/RTR/手机Tiled-BasedRender和GPU带宽.md)

[不同图形API剪裁空间说明](blogs/RTR/不同图形API剪裁空间说明.md)

[LOD计算策略](blogs/RTR/LOD策略.md)

## UE4 使用经验

[UE4基础光照模型](blogs/UE4/ue4光照模型.md) 

[UE4 Lightmap结构和编码](blogs/UE4/UnrealLightmap结构和编码.md) 

## UE4渲染源代码阅读

下面两个学习笔记没什么章法，回头看很混乱、不深入，所以尝试换一个角度重新整理

[UE4渲染源码基础Pipeline](blogs/UE4RenderingSource/UE4渲染源码基础Pipeline.md)

[UE4渲染源码基础结构](blogs/UE4RenderingSource/UE4渲染源码基础结构.md)

UE4代码学习：

[[UE代码学习]初探UE4代码层](blogs/UE4RenderingSource/[UE代码学习]初探UE4代码层.md)

[[UE代码学习]UE4Shader代码初探](blogs/UE4RenderingSource/[UE代码学习]UE4Shader代码初探.md)

[[UE代码学习]UE4渲染管线代码初探](blogs/UE4RenderingSource/[UE代码学习]UE4渲染管线代码初探.md) 正在学习

## Unity使用经验

[Unity Surface Shader阴影相关宏的说明](blogs/unity/fullforwardshadows.md)

[HDRP深度纹理读取与使用](blogs/unity/HDRP深度纹理.md)

[HDRP一些特殊功能：天空球编程和新Shader介绍](blogs/unity/Hdrp一些特殊功能.md)

[Unity线性空间和Gamma空间](blogs/unity/Linear And Gamma.md)

[自定义渲染目标](blogs/unity/SetRenderTarget.md)

[遮挡剔除简介](blogs/unity/Occluded Culling.md)

[Shander当中的坐标变换和对应矩阵说明](blogs/unity/Shader当中的坐标变化.md)

[HDRP Shader当中的坐标系变换矩阵](blogs/unity/HDRPShader当中的坐标系变换矩阵.md)

[深度相关内容](blogs/unity/深度信息.md)

[渲染过程中buffer的变化](blogs/unity/渲染过程中buffer的变化.md)

[Shaderd打包中遇到的问题](blogs/unity/Shader打包.md)

[在Shader中通过深度图恢复世界坐标](blogs/unity/世界坐标恢复.md)

[Enlighten原理与烘焙效率提升](blogs/unity/EnlightenBaking.md)

[UV云被远剪裁面Clip以及Shadow被近剪裁面Clip](blogs/unity/UV云被裁减问题.md)

[HDRP中ShaderGraph生成的UnlitTransparent不接受雾](blogs/unity/HDRP中ShaderGraph生成的UnlitTransparent不接受雾.md)

[Unity贴图压缩](blogs/unity/Unity贴图压缩.md)

[Houdini无法导出fbx格式的UV.z通道](blogs/unity/Houdini导出fbxUVz通道.md)

[UnityTextureStreaming](blogs/unity/UnityTextureStreaming.md)

## 图形API

[Vulkan Shader编译与加载过程](blogs/GraphicAPI/Vulkan_Shader编译与加载过程.md)  待补充Shader变种

[Vulkan-A-Specification中文摘录](blogs/VulkanTranslate/Vulkan-A-Specification中文翻译版.md)  

## 资源管理

[FBX文件格式](blogs/3DBasic/FBXFileFormat.md)

## Unity 渲染效果

[Unity中实现Depth Peeling](blogs/RTR/DepthPeeling.md)

[修改UnityHDRP管线来实现怪物猎人世界中宠物毛发效果](blogs/UnityEffect/修改UnityHDRP管线来实现怪物猎人世界中宠物毛发效果.md)

[TODO:shell模型实现毛发渲染](blogs/RTR/毛发方案分析.md) 

[在Unity中实现的《Horizon: Zero Dawn》体积云](gallerys/volumecloud.md)

[噪声与布朗分型的效果探索](gallerys/noisework.md)

[SDF在unity渲染当中的应用探索](blogs/UnityEffect/在Polygon渲染当中使用SDF效果.md)

[PostProcessing V3](blogs/post/PostprocessingV3.md)

[PPV3 HDRP自定义后处理](blogs/post/HDRP自定义后处理.md)

[ColorGrading介绍](blogs/post/ColorGrading.md)  （入门级的ColorGrading简单介绍）

[Unity SRP实现SDF渲染基础框架](blogs/SDF/SDFRenderFrameWork.md)

## 美术制作方案

[后处理ColorGrading](blogs/Art/后处理ColorGrading.md) （中级的ColorGrading说明）

[HDR ColorGrading技术方案](blogs/Art/HDRColorGrading技术方案.md)（高级的ColorGrading和HDR后期处理技术方案）

[ColorGrading中LUT的生成与使用](blogs/Art/ColorGrading中LUT的生成与使用.md)

[材质制作数值标准总结](blogs/Art/材质制作数值标准总结.md)

[材质制作参数F0](blogs/Art/材质制作参数F0.md)

[BRDF与色彩与明暗](blogs/Art/BRDF与色彩与明暗.md)

[CubeMap生成与使用](blogs/HDRPartist/CubeMap生成.md)

[HDRP基础Shader](blogs/HDRPartist/HDRP基础Shader.md)

[Shader Graph](blogs/HDRPartist/Shader Graph.md)

[物理摄像机光照配置](blogs/HDRPartist/unity物理摄相机.md)

[在Substance Painter当中实现Unity HDRP的光照模型](blogs/HDRPartist/SubstanceUseUnityShading.md)

[场景美术资源优化思路](blogs/HDRPartist/场景美术资源优化思路.md)

[Density Volume 3D纹理制作方案](blogs/HDRPartist/Density Volume 3D纹理制作方案.md)

[法线贴图旋转](blogs/HDRPartist/法线贴图旋转.md)

[从模型到打光流程](gallerys/gallery.md)

[Unity24小时基于物理的光照效果](gallerys/unity24h.md)

[米哈游原神细节实现猜测.md](blogs/GameAnalysis/米哈游原神细节实现猜测.md)

[HDRP场景反射](blogs/HDRPartist/场景反射策略.md)

## Unity HDRP 源代码阅读

下面这部分是HDRP6.*版本的内容。

[1.HDRP渲染管线总结](blogs/HDRPsource/new1.HDRP渲染管线总结.md)

[2.HDRP Shader开发说明](blogs/HDRPsource/new2.HDRPShader开发说明.md)

[3.阴影部分代码](blogs/HDRPsource/new3.HDRP Shadow.md)

[HDRPShader内置矩阵备查](blogs/HDRPsource/HDRPShader内置矩阵备查.md)

下面这部分内容主要总结自HDRP4.*以及之前的版本，很多地方不是很准确，并且有很大变动。

[SPR官方介绍](blogs/unity/SPR官方介绍.md)

[1.代码结构](blogs/HDRPsource/1.代码结构.md)

[2.ShaderPass说明](blogs/HDRPsource/2.ShaderPass说明.md)

[3.渲染参数](blogs/HDRPsource/3.渲染参数.md)

[4.LitShader分析](4.LitShader分析.md)

[5.UnityHDRP光源配置](blogs/HDRPsource/5.UnityHDRP光源配置.md)

[6.HDRPDecal](blogs/HDRPsource/6.HDRPDecal.md)

[7.光源配置相关参数](blogs/HDRPsource/7.光源配置相关参数.md)

[8.LitShader](blogs/HDRPsource/8.LitShader.md)

[9.OcclusionProbe](blogs/HDRPsource/9.OcclusionProbe.md)

[10.Lighting设置](blogs/HDRPsource/10.Lighting设置.md)

[HDRP中HairShader代码结构](blogs/unity/HDRP中HairShader代码结构.md)

[HDRP中HairShader代码结构2](blogs/unity/HDRP中HairShader代码结构2.md)

## Shader 编程

[GPU浮点数的精确度注意事项](blogs/Shader/渲染过程中各种浮点数的精确度.md) 从R11G11B10的格式出发，解释图像精确度的原理

[Shader函数重载的问题](blogs/Shader/Shader函数重载的问题.md) 

[不同图形API矩阵行列主序及其影响](blogs/Shader/不同图形API矩阵行列主序及其影响.md) 

[ShaderModel与ES版本和GPU特性的关系](blogs/Shader/ShaderModel与ES版本和GPU特性的关系.md) 

## GPU硬件及架构相关

[【重要】谈谈那些early depth testing技术](blogs/GPUAartch/谈谈那些early_depth_testing技术.md) （草拟）

[HDR硬件层Framebuffer格式标准与渲染阶段的关系](blogs/GPUAartch/HDR硬件层Framebuffer格式标准与渲染阶段的关系.md)

[手游GPU架构与优化之UnifiedShaderCore](blogs/GPUAartch/手游GPU架构与优化之UnifiedShaderCore.md)

[纹理缓存](blogs/GPUAartch/纹理缓存.md)

[GPU架构基础](blogs/GPUAartch/GPU架构.md)

[GPU分支语句性能](blogs/GPUAartch/GPU分支语句性能.md)

[关于Cuda、Shader中if语句执行原理](blogs/GPUAartch/关于Cuda、Shader中if语句执行原理.md)

[GPU架构类型整合](blogs/GPUAartch/GPU架构类型整合.md)

[TeraScale架构浅析](blogs/GPUAartch/TeraScale架构浅析.md)

[TODO:GCN架构浅析与优化](blogs/GPUAartch/GCN架构浅析.md)

[手机图形架构浅析与优化](blogs/GPUAartch/手机GPU架构与优化.md)

## 杂项

[TA工作内容](blogs/HDRPartist/TA工作内容.md)

[游戏行业](blogs/game/game_index.md)

[caffe](blogs/caffe_reader/caffe_index.md)

[技术](blogs/tech/tech_index.md)

[语言学习](blogs/language/lang_index.md)

## 我的动态

[LearningRoadMap](LearningRoadMap.md)

