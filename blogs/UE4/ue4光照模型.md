# UE4光照模型

## 引文

主要参考文章《Real Shading in Unreal Engine 4》by Brian Karis, Epic Games.

详细介绍了UE4的PBR模型，这里对其中主要的细节总结。

UE4的PBR灵感来自于Disney：

Burley, Brent, “Physically-Based Shading at Disney”, part of “Practical Physically Based Shading
in Film and Game Production”, SIGGRAPH 2012 Course Notes. http://blog.selfshadow.com/
publications/s2012-shading-course/

使用PBR的目的如下,原文：

```c
Real-Time Performance
    • First and foremost, it needs to be efficient to use with many lights visible at a time.
    Reduced Complexity
    • There should be as few parameters as possible. A large array of parameters either results in
    decision paralysis, trial and error, or interconnected properties that require many values to be
    changed for a single intended effect.
    • We need to be able to use image-based lighting and analytic light sources interchangeably, so
    parameters must behave consistently across all light types.

Intuitive Interface
    • We prefer simple-to-understand values, as opposed to physical ones such as index of refraction.
    Perceptually Linear
    • We wish to support layering through masks, but we can only afford to shade once per pixel. This
    means that parameter-blended shading must match blending of the shaded results as closely as
    possible.
    
Easy to Master
    • We would like to avoid the need for technical understanding of dielectrics and conductors, as well
    as minimize the effort required to create basic physically plausible materials.
    Robust
    • It should be difficult to mistakenly create physically implausible materials.
    • All combinations of parameters should be as robust and plausible as possible.
    Expressive
    • Deferred shading limits the number of shading models we can have, so our base shading model
    needs to be descriptive enough to cover 99% of the materials that occur in the real world.
    • All layerable materials need to share the same set of parameters in order to blend between them.
    Flexible
    • Other projects and licensees may not share the same goal of photorealism, so it needs to be
    flexible enough to enable non-photorealistic rendering
```

## 光照模型

UE4的漫反射模型使用Lambertian：

$$ f(l,v) = \frac{c_{diff}}{\pi} $$

$ c_{diff} $是表面的albedo。

UE4的高光同样使用微表面模型：

$$ f(l,v) = \frac{D(h)G(l,v,h)F(v,h)}{4(nl)(nv)} $$

### D项 ：GGX (Trowbridge-Reitz)

$$ \alpha = roughness^2 $$

$$ D_{GGX}(h)=\frac{\alpha^2}{\pi((nh)^2(\alpha^2-1)+1)^2} $$

### G项：Schlick

UE4使用的Schlick-GGX（用在IBL上）

$$k =\frac{\alpha} {2}$$  

$$ \alpha = roughness^2 $$

UE4又根据Disney文章对其做了修改（这个修改只用在解析光源上，IBL在glancing angles会太暗）：

$$k =\frac{(roughness + 1)^2} {8}$$  

$$ G_{GGX}(v) = \frac{nv}{(nv)(1-k)+k} $$

### F项：Schlick’s approximation

$$ F(v,h)=F_0 + (1 - F_0) 2^{-5.55473(vh)-6.98316)(vh)} $$

## IBL

首先需要解决的是辐射度积分，通常使用重要度采样：

$$ \int_H L_i(l)f(l,v)cos(\theta_l)dl  = \frac{1}{N}\sum^{N}_{k=1}\frac{L_i(l_k)f(l_k,v)cos theta _{l_k} }{p(l_k,v)} $$

表示采样半球上的所有光线.k表示反射探针上的一个点，采样多个点，p表示概率。

代码：

```c
float3 ImportanceSampleGGX( float2 Xi, float Roughness , float3 N )
{
    float a = Roughness * Roughness;
    float Phi = 2 * PI * Xi.x;
    float CosTheta = sqrt( (1 - Xi.y) / ( 1 + (a*a - 1) * Xi.y ) );
    float SinTheta = sqrt( 1 - CosTheta * CosTheta );
    float3 H;
    H.x = SinTheta * cos( Phi );
    H.y = SinTheta * sin( Phi );
    H.z = CosTheta;
    float3 UpVector = abs(N.z) < 0.999 ? float3(0,0,1) : float3(1,0,0);
    float3 TangentX = normalize( cross( UpVector , N ) );
    float3 TangentY = cross( N, TangentX );
    // Tangent to world space
    return TangentX * H.x + TangentY * H.y + N * H.z;
}
```

### 重要度采样的推导

这部分内容主要解释什么是重要度采样以及如何推导。

从PDF（概率密度函数：积分为1）到重要度采样函数的过程。

目的：通过求和模拟复杂函数积分，代表就是蒙特卡洛积分。

蒙特卡洛积分是等分采样，重要度采样能够修正蒙特卡洛积分的权重来提升准确度。

重要度采样：引入分布p(x)。

蒙特卡洛计算积分：

$$ \int_a^b f(x) dx = \frac{b-a}{N}\sum_{i=1}^{N}f(x_i)$$

重要度采样积分：

$$ \int_a^b f(x) dx = \int_a^b p(x)\frac{\pi(x)}{p(x)}f(x) dx$$

我们可以将上面的函数当做$ \frac{\pi(x)}{p(x)}f(x)  $在概率p(x)上的期望，则：

可以在$ p(x) $上采样估计期望: 

$$ E|f| = \frac{1}{N}\sum_{i=1}^{N}\frac{\pi(x_i)}{p(x_i)}f(x_i) $$

则$ \frac{\pi(x_i)}{p(x_i)} $为重要度权重。

所以重要度采样就需要有一个新的分布和分布对应的概率。

#### Importance Sampling techniques for GGX 

针对IBL和ray tracing 没有L方向所以需要这个方法计算。

光照函数：

$$  L_o = L_e + \int L_i f(w_i,w_o) (w_i.w_n) dw_i$$

对于一条光线：

$$ f(l,v) = \frac{D(h)G(l,v,h)F(v,h)}{4(nl)(nv)} $$

我们现在探讨GGX的重要度采样，应为NDF对于整个BRDF有重要的影响，在Ray Tracing当中也是重要的讨论部分。

$$ D_{GGX}(h)=\frac{\alpha^2}{\pi((nh)^2(\alpha^2-1)+1)^2} $$

为了进行重要度采样，我们需要在求D(h)的边缘分布函数（CDF）的倒数。来生成一个微表面法线(因为光线追踪和IBL当中可能没有光线方向，所以需要特殊的方法计算这个h)。

#### 概率论预备知识

概率密度函数：“PDF”（**P**robability **D**ensity **F**unction）

联合概率：P(A,B)

条件概率：P(A|B) = P(A,B) / P(A)

边缘概率：P(A)

事件独立：P(A|B) = P(A,B) / P(B) = P(A)

#### Phong BRDF

Phong模型的PDF：

$$ p(\theta,\phi)= \frac{n + 1}{2\pi}cos^n(\theta)sin(\theta) $$

首先对$ \phi $积分，得到了$ \theta $的边缘密度函数。：

$$ p(\theta)= \int_0^{2\pi}p(\theta,\phi)  d\phi=(n + 1)cos^n(\theta)sin(\theta) $$

然后推导出$ \phi $的条件概率，各项同性的$ p(\phi || \theta)  $结果永远都是这样。刚好是一个单位圆的周长，他们是独立的：

$$ p(\phi | \theta) = \frac{p(\theta,\phi)}{p(\theta)} = \frac{1}{2\pi} = p(\phi )$$

现在我们得到了两个条件概率，分别积分就可以得到具体的概率值。

设 $ P(S_\phi)  $为$ \phi $的概率，则对条件概率积分：

$$ P(S_\phi) = \int_0^{S_\phi} p(\phi) d \phi  = \int_0^{S_\phi} \frac{1}{2\pi}d \phi = \frac{S_\phi}{2\pi}$$

假设 $ P(S_\phi)  $是一个随机变量：

 $$ P(S_\phi)  =\frac{ s_\phi}{2\pi} =\xi_\phi  $$

$$   s_\phi  = 2\pi\xi_\phi $$

设 $ P(S_\theta)  $为$ \theta $的概率，则对条件概率积分：

$$ P(S_\theta)=\int_0^{S_\theta} p(\theta)d\theta =\int_0^{S_\theta} (n + 1)cos^n(\theta)sin(\theta) d\theta = 1 - cos^{1 + n}s_{\theta}$$

$$   s_\theta  = cos^{-1}((1-\xi_\theta)^{\frac{1}{1+n}}) $$

$ \xi  $是一个随机变量。重要度采样的代码：

```c
vec2 importance_sample_phong(vec2 xi)
{
  float phi = 2.0f * PI * xi.x;
  float theta = acos(pow(1.0f - xi.y, 1.0f/(n+1.0f)));
  return vec2(phi, theta);
}
```

### IBL计算

```c
float3 SpecularIBL( float3 SpecularColor , float Roughness , float3 N, float3 V )
{
    float3 SpecularLighting = 0;
    const uint NumSamples = 1024;
    for( uint i = 0; i < NumSamples; i++ )
    {
    	float2 Xi = Hammersley( i, NumSamples );
        // 使用xi分布进行采样。
        float3 H = ImportanceSampleGGX( Xi, Roughness , N );
        float3 L = 2 * dot( V, H ) * H - V;
        float NoV = saturate( dot( N, V ) );
        float NoL = saturate( dot( N, L ) );
        float NoH = saturate( dot( N, H ) );
        float VoH = saturate( dot( V, H ) );
        if( NoL > 0 )
        {
            float3 SampleColor = EnvMap.SampleLevel( EnvMapSampler , L, 0 ).rgb;
            float G = G_Smith( Roughness , NoV, NoL );
            float Fc = pow( 1 - VoH, 5 );
            float3 F = (1 - Fc) * SpecularColor + Fc;
            // Incident light = SampleColor * NoL
            // Microfacet specular = D*G*F / (4*NoL*NoV)
            // pdf = D * NoH / (4 * VoH) xi对应的概率
            // SpecularLighting =  D*G*F / (4*NoL*NoV) /  (pdf)
            // SpecularLighting = NoL* D*G*F / (4*NoL*NoV) * (4 * VoH)  / D * NoH
            // 这里pdf是1所以公式变为 G*F / (4*NoL*NoV)，约分之后得到下面的公式
            // 也就是说P等于1
            SpecularLighting += SampleColor * F * G * VoH / (NoH * NoV);
        }
    }
    // 取平均值
    return SpecularLighting / NumSamples;
}
```

