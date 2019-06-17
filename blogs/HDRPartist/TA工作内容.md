## 性能优化：

* BRDF 计算优化： GGX : F \Geometry\NDF。(手游上比较重要)
* Texture Atlasing 工具：PBR贴图Pack在一起，合成一个大的贴图，Shader种类必须特别小。
* IBL cubemap Normalization：看一下战神的PBR（主要改进的是效果）

## 生产力优化：

* Light probe 自动摆放工具：能走区域(Nav Mesh)，贪心，光照变化。
* Texture Streaming工具：Unity内置支持一些功能。
* GPU LightMapper/ Enlighten 动态设置工具。
* 程序化生成工具，地形生成、物体摆放、模型生成。
* 资源管理工具。

## 标准制定：

* 文档+培训。
* 所见即所得的工具：帮助美术更好的理解PBR制作的结果，查看工具。
* 多个版本的标准光照环境：确认资产和灯光得到正确结果：场景，亮度适中、光源没有色彩倾向、没有强烈明暗对比。为了很好的辨识固有色、色相、光滑度、法线、金属度、AO。只用ACES：不用曝光和其他后处理（就是白平衡场景）。
* 灯光的强度参考：各种常见的值得设置、自动曝光设置方式。
* 把工具嵌入在流程当中。
* 定制的SP环境：定制Shader和定制后处理。



## Shader关联：

* Shader Strip 工具。
* Shader 变体控制。
* Shader Warmup
* Shader Feature