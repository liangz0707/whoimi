## 优化性能：

* BRDF 计算优化： GGX : F \Geometry\NDF。手游上比较重要。

* 材质优化：PBR贴图Pack在一起。

* IBL cubemap Normalization：看一下战神的PBR。



## 优化生产力工具：

* Texture Atlasing 工具 ->  Shader种类必须特别小。

* Light probe 自动摆放工具：能走区域(Nav Mesh)，贪心，光照变化。
* Texture Streaming工具。
* GPU LightMapper/ Enlighten 动态设置工具。



## 标准：

文档+培训。

所见即所得的工具：帮助美术更好的理解PBR制作的结果。

多个版本的标准光照确认资产和灯光得到正确结果：场景，亮度适中、光源没有色彩倾向、没有强烈明暗对比。为了很好的辨识固有色、色相、光滑度、法线、金属度、AO。

只用ACES：不用曝光。

灯光的内容强度标准：各种常见的值得设置。自动曝光。

HDR的具体分析、和能量守恒分析、基于物理的光照。double LDR？

把工具嵌入在流程当中。

定制的SP环境：定制Shader和定制后处理。



## Shader关联：

Shader Strip 

Shader Vatrient

Shader Warmup

Shader Feature