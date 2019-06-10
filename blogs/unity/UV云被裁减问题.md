# UV云或阴影被远近剪裁面Clip的问题

在制作大世界的过程中，美术需要自己设计云的摆放，这样的云实际上就是一个摆放在天上的大透明面片。

所有物体的渲染都需要经历一个硬件级别的剪裁，也就是ClipSpace的作用。但是考虑到场景中视距只有几百米，所以远剪裁面可能为500。这就导致云被clip掉了。

硬件对顶点数据进行剪裁的依据也是clipspace坐标：

在Unity当中：

```c
v2f vert (appdata v)
{
    v2f o;
    o.vertex = UnityObjectToClipPos(v.vertex);
    return o;
}
```

clipspace.z 通常用来进行剪裁、进行深度测试、写入深度。同时依赖于clipspace.w.

clipspace.xy 通常用来用作透视变换、视口变换、光栅化。同时依赖于clipspace.w.

当渲染透明物体的时候，clipspace.z实际上就只用做了深度测试和剪裁。

uv云不希望被剪裁掉，所以可以修改clipspace.z，让他在远近剪裁面之间。

又由于uv云还是需要进行正确的深度测试，不能挡住不透明物体，所以需要放到远剪裁面上。

```c
v2f vert (appdata v)
{
    v2f o;
    o.vertex = UnityObjectToClipPos(v.vertex);
    o.vertex.z =  1;
    return o;
}
```

o.vertex.z 的远近剪裁面实际上和具体的图形接口有关，D3D和Opengl的远剪裁面值是不一样的。所以不能简单的设置成1.

Unity当中就可以通过：

```c
#if UNITY_REVERSED_Z
	// 远处比较大
#else
	// 近处深度大
#endif

// 远近剪裁面 API相关设置：
D3D11
#define UNITY_REVERSED_Z 1
#define UNITY_NEAR_CLIP_VALUE (1.0)
#define UNITY_RAW_FAR_CLIP_VALUE (0.0)

GL
#define UNITY_REVERSED_Z 0
#define UNITY_NEAR_CLIP_VALUE (-1.0)
#define UNITY_RAW_FAR_CLIP_VALUE (1.0)

PSSL
#define UNITY_REVERSED_Z 1
#define UNITY_NEAR_CLIP_VALUE (1.0)
#define UNITY_RAW_FAR_CLIP_VALUE (0.0)

// 把clip限制在近剪裁面：
#if defined(UNITY_REVERSED_Z)
    float clamped = min(clipPos.z, clipPos.w*UNITY_NEAR_CLIP_VALUE);// clipPos.z<1
#else
    float clamped = max(clipPos.z, clipPos.w*UNITY_NEAR_CLIP_VALUE);// clipPos.z>-1
#endif

// 把clip限制在远：
#if defined(UNITY_REVERSED_Z)
    float clamped = max(clipPos.z, UNITY_RAW_FAR_CLIP_VALUE);// clipPos.z>0
#else
    float clamped = min(clipPos.z, UNITY_RAW_FAR_CLIP_VALUE);// clipPos.z<1
#endif
```


