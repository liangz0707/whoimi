# BRDF常用高光项模型

在基础的BRDF当中，我们推到出了基础的BRDF公式结构：

$$f = \frac{DGF}{4 (nl)(nv)}$$

D项：法线分布函数用来描述，法线方向在表面上的分布情况，积分结果为1。

G项：用来描述Shadow和masking项。用来描述不同的几何结构有多少光线被遮挡。

F项：用来描述在不同材质，不同角度上，反射光线所占的比例。

下面来分析不同的的实现，主要从[博客](<https://graphicrants.blogspot.com/2013/08/specular-brdf-reference.html>)翻译而来：

首先，UE4和unity HDRP当中对于roughness的计算都有一些特别。

例如UE，当中参与计算的roughness为：

$$ \alpha = roughness^2 $$

unity HDRP当中使用的是smoothness，最终参与计算的roughness为:

$$ \alpha = (1 - smothness)^2 $$

## 法线分布函数NDF

法线分布函数主要用来描述高光分布，也是用来描述微表面朝向的分布。所以他的半球积分为1：

$$ \int_\Omega D(h)(n⋅h)d w_i = 1$$ 

其中m为半角，并且每一项都有一个$ \frac{1}{\pi \alpha^2} $ 因子。

### Blinn-Phong

$$ D_{Blinn}(h) = \frac{1}{\pi \alpha^2} (nh)^{\frac{2}{\alpha^2} -2} $$

### Beckmann

$$ D_{Beckmann}(h)= \frac{1}{\pi \alpha^2 (nh)^4}\exp (\frac{(nh)^2 -1}{\alpha^2(nh)^2}) $$

### GGX (Trowbridge-Reitz)

$$ D_{GGX}(h)=\frac{\alpha^2}{\pi((nh)^2(\alpha^2-1)+1)^2} $$

### GGX Anisotropic

$$ D_{GGXaniso}(h)=\frac{1}{\pi\alpha_x\alpha_y} \frac{1}{(\frac{(xh)^2}{\alpha_x^2} +\frac{(yh)^2}{\alpha_y^2}+(nh)^2)^2} $$

## 几何结构项Geometric Shadowing

几何结构项描述了微表面的对于光线的遮挡关系，他是一个对光线的削减比例，所以不需要满足积分为1。对光线遮挡的比例依赖于粗糙度和微表面分布（NDF）。

### Implicit

$$  G_{Implicit}(l,v,h) = (nl)(nv)$$

 ### Neumann 

$$  G_{Neumann}(l,v,h) = \frac{(nl)(nv)}{max(nl,nv)}$$

### Cook-Torrance

$$ G_{Cook-Torrance}(l,v,h)=min(1, \frac{2(nh)(nv)}{vh}, \frac{2(nh)(nl)}{vh} )$$

### Kelemen 

$$ G_{Kelemen } (l,v,h)=\frac{(nl)(nv)}{(vh)^2}$$

## Smith方法描述的几何结构项

Smith方法根据不同的法线分布函数NDF，来构造对应的结构遮挡。Smith把遮挡拆分成了两项：灯光和视线，他们使用同一个函数。

$$ G(l,v,h)=G_1(l)G_1(v) $$

下面根据不同的NDF来给出对应的$ G_1 $

### Blinn-Phong

没有对应的$ G_1 $,建议使用Beckmann的$ G_1 $。

### **Beckmann** 

$$ c=\frac{nv}{\alpha\sqrt{1-(nv)^2}} $$

$$ G_{Beckmann}(v)= $$

$$ if c < 1.6 :  \frac{3.524c+2.181c^2}{1+2.276c+2.577c^2} $$

$$ else:  1 $$

### GGX

$$  G_{GGX}(v) = \frac{2(nv)}{(nv)+\sqrt{\alpha^2+(1-\alpha^2)(nv)^2}}  $$

### Schlick-Beckmann

$$k = \alpha \sqrt{\frac{2}{\pi}}$$

$$ G_{Schlick}(v) = \frac{nv}{(nv)(1-k)+k} $$

### Schlick-GGX

$$k =\frac{\alpha} {2}$$

$$ G_{GGX}(v) = \frac{nv}{(nv)(1-k)+k} $$

## 菲涅尔Fresnel

菲涅尔主要用来描述在不同的入射角度上，给出一个折射率（index of refraction），计算反射和折射的比例。

菲涅尔的规律是：入射角越大，光线反射比例越大。

通常我们使用的输入参数不是折射率（IoR），而是垂直入射时的反射比$ F_0 $，并且通常隐含了如果入射角度接近90度，那么反射比会趋近于1($ F_{90} = 1$)。

### None

$$  F_{None}(vh) = F_0$$

### Schlick

$$ F_{Schlick}(v,h)=F_0+(1-F_0)(1-(vh))^5 $$ 

目前Unity使用的就是这种。

### Cook-Torrance

$$ \eta = \frac{1+\sqrt{F_0}}{1-\sqrt{F_0}}$$

$$ c=vh $$

$$ g = \sqrt{\eta^2 + c^2 -1} $$

$$ F_{Cook-Torrance}(vh)=\frac{1}{2}(\frac{g-c}{g+c})^2(1+(\frac{(g+c)c-1}{(g-c)c+1})^2) $$

## 优化

由于brdf计算过于复杂，brdf通常需要优化，优化方案通常有以下几种：

1. 将brdf作为一个整体进行约分化简。
2. 使用预计算，把计算结果存储在贴图当中。
3. 简化brdf公式，使用brdf公式的lod策略。

## References

[1] Hoffman 2013, 

"Background: Physics and Math of Shading"

[2] Blinn 1977, "Models of light reflection for computer synthesized pictures"

[3] Beckmann 1963, "The scattering of electromagnetic waves from rough surfaces"

[4] Walter et al. 2007, 

"Microfacet models for refraction through rough surfaces"

[5] Burley 2012, 

"Physically-Based Shading at Disney"

[6] Neumann et al. 1999, 

"Compact metallic reflectance models"

[7] Kelemen 2001, 

"A microfacet based coupled specular-matte brdf model with importance sampling"

[8] Smith 1967, "Geometrical shadowing of a random rough surface"

[9] Schlick 1994, 

"An Inexpensive BRDF Model for Physically-Based Rendering"

[10] Karis 2013, 

"Real Shading in Unreal Engine 4"

[11] Cook and Torrance 1982, 

"A Reflectance Model for Computer Graphics"

[12] Reed 2013, 

"How Is the NDF Really Defined?"