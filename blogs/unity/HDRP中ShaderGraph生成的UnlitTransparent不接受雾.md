# HDRP中ShaderGraph生成的UnlitTransparent不接受雾

## 问题：

在开发UV云的过程中，使用了ShaderGraph来创建Shader。

发现使用Unlit节点的Transparent模式，会出现物体不接收雾的情况（即使打开了Recive Fog）。

但是Lit节点的Transparent模式是正常的。

## 解决方案：

通过阅读代码发现：

雾的计算在文件AtmosphericScattering.hlsl中：

```c
float4 EvaluateAtmosphericScattering(PositionInputs posInput, float3 V)
{
    // TODO: do not recompute this, but rather pass it directly.
    float fragDist = distance(posInput.positionWS, GetCurrentViewPosition());

    ...
}
```

这里最重要的内容是posInput.positionWS。出错的原因就是posInput.positionWS当中没有值。

所以导致计算雾深度的时候结果是0。

ShaderGraph生成的代码当中，在Vertex片段中VertMesh函数（所有的HDRP内置shader都走这个shader函数计算vertex信息）已经计算了世界坐标，但是没有被传递到Fragment当中。因为ShaderGraph当中判断他不需要世界坐标。

VertMesh函数位于VertMesh.hlsl当中：

```c

VaryingsMeshType VertMesh(AttributesMesh input)
{
    VaryingsMeshType output;
	...
    // This return the camera relative position (if enable)
    float3 positionRWS = TransformObjectToWorld(input.positionOS);
    ...
    
#ifdef TESSELLATION_ON
    output.positionRWS = positionRWS;
	...
#else
    #ifdef VARYINGS_NEED_POSITION_WS
    output.positionRWS = positionRWS;
    #endif
    output.positionCS = TransformWorldToHClip(positionRWS);
	...
#endif
}
```

下面是ShaderGraph生成的传递函数和结构体，法线并没有世界坐标：

```c
        // Generated Type: VaryingsMeshToPS
        struct VaryingsMeshToPS {
            float4 positionCS : SV_Position;
            #if UNITY_ANY_INSTANCING_ENABLED
            uint instanceID : CUSTOM_INSTANCE_ID;
            #endif // UNITY_ANY_INSTANCING_ENABLED
        };
        struct PackedVaryingsMeshToPS {
            float4 positionCS : SV_Position; // unpacked
            #if UNITY_ANY_INSTANCING_ENABLED
            uint instanceID : CUSTOM_INSTANCE_ID; // unpacked
            #endif // UNITY_ANY_INSTANCING_ENABLED
        };
        PackedVaryingsMeshToPS PackVaryingsMeshToPS(VaryingsMeshToPS input)
        {
            PackedVaryingsMeshToPS output;
            output.positionCS = input.positionCS;
            #if UNITY_ANY_INSTANCING_ENABLED
            output.instanceID = input.instanceID;
            #endif // UNITY_ANY_INSTANCING_ENABLED
            return output;
        }
        VaryingsMeshToPS UnpackVaryingsMeshToPS(PackedVaryingsMeshToPS input)
        {
            VaryingsMeshToPS output;
            output.positionCS = input.positionCS;
            #if UNITY_ANY_INSTANCING_ENABLED
            output.instanceID = input.instanceID;
            #endif // UNITY_ANY_INSTANCING_ENABLED
            return output;
        }
```

在ShaderGraph当中使用一次世界坐标：

![201975-114401](img/201975-114401.jpg)

然后在观察生成的代码，出现了世界坐标，再看场景雾就正常了：

```c
        // Generated Type: VaryingsMeshToPS
        struct VaryingsMeshToPS {
            float4 positionCS : SV_Position;
            float3 positionRWS; // optional
            #if UNITY_ANY_INSTANCING_ENABLED
            uint instanceID : CUSTOM_INSTANCE_ID;
            #endif // UNITY_ANY_INSTANCING_ENABLED
        };
        struct PackedVaryingsMeshToPS {
            float3 interp00 : TEXCOORD0; // auto-packed
            float4 positionCS : SV_Position; // unpacked
            #if UNITY_ANY_INSTANCING_ENABLED
            uint instanceID : CUSTOM_INSTANCE_ID; // unpacked
            #endif // UNITY_ANY_INSTANCING_ENABLED
        };
        PackedVaryingsMeshToPS PackVaryingsMeshToPS(VaryingsMeshToPS input)
        {
            PackedVaryingsMeshToPS output;
            output.positionCS = input.positionCS;
            output.interp00.xyz = input.positionRWS;
            #if UNITY_ANY_INSTANCING_ENABLED
            output.instanceID = input.instanceID;
            #endif // UNITY_ANY_INSTANCING_ENABLED
            return output;
        }
        VaryingsMeshToPS UnpackVaryingsMeshToPS(PackedVaryingsMeshToPS input)
        {
            VaryingsMeshToPS output;
            output.positionCS = input.positionCS;
            output.positionRWS = input.interp00.xyz;
            #if UNITY_ANY_INSTANCING_ENABLED
            output.instanceID = input.instanceID;
            #endif // UNITY_ANY_INSTANCING_ENABLED
            return output;
        }
```

## 注意

shader代码hlsl的包含调用关系比较复杂。