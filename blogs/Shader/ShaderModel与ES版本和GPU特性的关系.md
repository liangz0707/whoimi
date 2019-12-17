# ShaderModel与ES版本和GPU特性的关系

## Opengl版本和功能的对应

[Opengl网站](https://www.khronos.org/opengl/wiki/History_of_OpenGL)



### OpenGL 4.0 (2010)

|                           Addition                           | [Core Extension](https://www.khronos.org/opengl/wiki/Extension#Core_Extensions) |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
|                    Shading language 4.00                     | [ARB_texture_query_lod](http://www.opengl.org/registry/specs/ARB/texture_query_lod.txt), [ARB_gpu_shader5](http://www.opengl.org/registry/specs/ARB/gpu_shader5.txt), [ARB_gpu_shader_fp64](http://www.opengl.org/registry/specs/ARB/gpu_shader_fp64.txt), [ARB_shader_subroutine](http://www.opengl.org/registry/specs/ARB/shader_subroutine.txt), [ARB_texture_gather](http://www.opengl.org/registry/specs/ARB/texture_gather.txt) |
| [Indirect Drawing](https://www.khronos.org/opengl/wiki/Indirect_Drawing), without multidraw | [ARB_draw_indirect](http://www.opengl.org/registry/specs/ARB/draw_indirect.txt) |
|          Request minimum number of fragment inputs           | [ARB_sample_shading](http://www.opengl.org/registry/specs/ARB/sample_shading.txt) |
| [Tessellation](https://www.khronos.org/opengl/wiki/Tessellation), with shader stages | [ARB_tessellation_shader](http://www.opengl.org/registry/specs/ARB/tessellation_shader.txt) |
| [Buffer Texture](https://www.khronos.org/opengl/wiki/Buffer_Texture) formats RGB32F, RGB32I, RGB32UI | [ARB_texture_buffer_object_rgb32](http://www.opengl.org/registry/specs/ARB/texture_buffer_object_rgb32.txt) |
| [Cubemap Array Texture](https://www.khronos.org/opengl/wiki/Cubemap_Array_Texture) | [ARB_texture_cube_map_array](http://www.opengl.org/registry/specs/ARB/texture_cube_map_array.txt) |
| [Transform Feedback](https://www.khronos.org/opengl/wiki/Transform_Feedback) objects and multiple feedback stream output. | [ARB_transform_feedback2](http://www.opengl.org/registry/specs/ARB/transform_feedback2.txt), [ARB_transform_feedback3](http://www.opengl.org/registry/specs/ARB/transform_feedback3.txt) |
|                           Addition                           |                        Promoted from                         |
| [Individual blend equations for each color output](https://www.khronos.org/opengl/wiki/Draw_Buffer_Blend) | [ARB_draw_buffers_blend](http://www.opengl.org/registry/specs/ARB/draw_buffers_blend.txt) |

### OpenGL 3.3 (2010)

|                           Addition                           | [Core Extension](https://www.khronos.org/opengl/wiki/Extension#Core_Extensions) |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
|                    Shading language 3.30                     | [ARB_shader_bit_encoding](http://www.opengl.org/registry/specs/ARB/shader_bit_encoding.txt) |
| [Dual-source blending](https://www.khronos.org/opengl/wiki/Dual_Source_Blending). | [ARB_blend_func_extended](http://www.opengl.org/registry/specs/ARB/blend_func_extended.txt) |
| Shader-defined locations for [attributes](https://www.khronos.org/opengl/wiki/Layout_Vertex_Attribute) and [fragment shader outputs](https://www.khronos.org/opengl/wiki/Layout_Fragment_Output). | [ARB_explicit_attrib_location](http://www.opengl.org/registry/specs/ARB/explicit_attrib_location.txt) |
| Simple boolean [Occlusion Query](https://www.khronos.org/opengl/wiki/Occlusion_Query) | [ARB_occlusion_query2](http://www.opengl.org/registry/specs/ARB/occlusion_query2.txt) |
| [Sampler Objects](https://www.khronos.org/opengl/wiki/Sampler_Object) | [ARB_sampler_objects](http://www.opengl.org/registry/specs/ARB/sampler_objects.txt) |
| A new [image format](https://www.khronos.org/opengl/wiki/Image_Format) for unsigned 10.10.10.2 colors | [ARB_texture_rgb10_a2ui](http://www.opengl.org/registry/specs/ARB/texture_rgb10_a2ui.txt) |
| [Texture swizzle](https://www.khronos.org/opengl/wiki/Texture_Swizzle) | [ARB_texture_swizzle](http://www.opengl.org/registry/specs/ARB/texture_swizzle.txt) |
| [Timer queries](https://www.khronos.org/opengl/wiki/Timer_Query) | [ARB_timer_query](http://www.opengl.org/registry/specs/ARB/timer_query.txt) |
| [Instanced arrays](https://www.khronos.org/opengl/wiki/Instanced_Array) | [ARB_instanced_arrays](http://www.opengl.org/registry/specs/ARB/instanced_arrays.txt) |
| [Vertex attributes 2.10.10.10](https://www.khronos.org/opengl/wiki/Vertex_Format_Type) | [ARB_vertex_type_2_10_10_10_rev](http://www.opengl.org/registry/specs/ARB/vertex_type_2_10_10_10_rev.txt) |

### OpenGL 3.2 (2009)

-   Core and compatibility profiles
-   Shading language 1.50

|                           Addition                           | [Core Extension](https://www.khronos.org/opengl/wiki/Extension#Core_Extensions) |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| [D3D compatible color vertex component ordering](https://www.khronos.org/opengl/wiki/D3D_Vertex_Format_Compatibility) | [ARB_vertex_array_bgra](http://www.opengl.org/registry/specs/ARB/vertex_array_bgra.txt) |
| [Drawing command allowing modification of the base vertex index](https://www.khronos.org/opengl/wiki/Draw_Base_Index) | [ARB_draw_elements_base_vertex](http://www.opengl.org/registry/specs/ARB/draw_elements_base_vertex.txt) |
| [Shader fragment coordinate convention control](https://www.khronos.org/opengl/wiki/Fragment_Shader#System_inputs) | [ARB_fragment_coord_conventions](http://www.opengl.org/registry/specs/ARB/fragment_coord_conventions.txt) |
| [Provoking vertex control](https://www.khronos.org/opengl/wiki/Provoking_Vertex) | [ARB_provoking_vertex](http://www.opengl.org/registry/specs/ARB/provoking_vertex.txt) |
| [Seamless cube map filtering](https://www.khronos.org/opengl/wiki/Seamless_Cubemap) | [ARB_seamless_cube_map](http://www.opengl.org/registry/specs/ARB/seamless_cube_map.txt) |
| Multisampled textures and texture samplers for specific sample locations | [ARB_texture_multisample](http://www.opengl.org/registry/specs/ARB/texture_multisample.txt) |
| Fragment [Depth Clamping](https://www.khronos.org/opengl/wiki/Depth_Clamp) | [ARB_depth_clamp](http://www.opengl.org/registry/specs/ARB/depth_clamp.txt) |
| [Fence sync objects](https://www.khronos.org/opengl/wiki/Sync_Object) | [ARB_sync](http://www.opengl.org/registry/specs/ARB/sync.txt) |
|                           Addition                           |                        Promoted from                         |
| [Geometry Shaders](https://www.khronos.org/opengl/wiki/Geometry_Shader), as well as input/output [Interface Blocks](https://www.khronos.org/opengl/wiki/Interface_Block) | [ARB_geometry_shader4](http://www.opengl.org/registry/specs/ARB/geometry_shader4.txt), heavily modified. |

### OpenGL 3.1 (2009)

-   All features deprecated in OpenGL 3.0 are removed except wide lines
-   Shading language 1.40
-   SNORM texture component formats

|                           Addition                           | [Core Extension](https://www.khronos.org/opengl/wiki/Extension#Core_Extensions) |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| [Uniform Buffer Objects](https://www.khronos.org/opengl/wiki/Uniform_Buffer_Object) | [ARB_uniform_buffer_object](http://www.opengl.org/registry/specs/ARB/uniform_buffer_object.txt) |
|                           Addition                           |                        Promoted from                         |
| [Instanced rendering with a per instance counter accessible to vertex shaders](https://www.khronos.org/opengl/wiki/Instancing) | [ARB_draw_instanced](http://www.opengl.org/registry/specs/ARB/draw_instanced.txt) |
|             Data copying between buffer objects              | [EXT_copy_buffer](http://www.opengl.org/registry/specs/EXT/copy_buffer.txt) |
| [Primitive restart](https://www.khronos.org/opengl/wiki/Primitive_restart) | [NV_primitive_restart](http://www.opengl.org/registry/specs/NV/primitive_restart.txt) |
| [Buffer Textures](https://www.khronos.org/opengl/wiki/Buffer_Texture) | [ARB_texture_buffer_object](http://www.opengl.org/registry/specs/ARB/texture_buffer_object.txt) |
| [Rectangle Textures](https://www.khronos.org/opengl/wiki/Rectangle_Texture) | [ARB_texture_rectangle](http://www.opengl.org/registry/specs/ARB/texture_rectangle.txt) |



## Microsoft对ShaderModel的定义

HLSL的ShaderModel主要表示不同版本的Shader支持的语言、功能特性。（The HLSL shader model is a versioning approach indicating which new features are added to the language. 

应用和游戏可以针对某个版本通用的功能进行开发，硬件和驱动工程师也可以针对这个版本的特性进行支持。



不同的ShaderModel版本：不同的处理阶段（Geometry etc）、约束、处理能力、向下兼容情况。

-   [Shader Model 1](https://msdn.microsoft.com/en-us/library/windows/desktop/bb509654(v=vs.85).aspx). This was the first shader model created in DirectX. It introduced vertex and pixel shaders to the first implementation of the programmable pipeline.
-   [Shader Model 2](https://msdn.microsoft.com/en-us/library/windows/desktop/bb509655(v=vs.85).aspx). Adds new intrinsics and increases limits on registers and instructions.
-   [Shader Model 3](https://msdn.microsoft.com/en-us/library/windows/desktop/bb509656(v=vs.85).aspx). Adds new intrinsics and increases limits on registers and instructions.
-   [Shader Model 4](https://msdn.microsoft.com/en-us/library/windows/desktop/bb509657(v=vs.85).aspx). This is a superset of the capabilities in Shader Model 3, except that Shader Model 4 doesn't support the features in Shader Model 1. It has been designed using a common-shader core that gives a common set of features to all programmable shaders, which are only programmable using HLSL. It adds new shader profiles to target geometry shaders.
-   [Shader Model 5](https://msdn.microsoft.com/en-us/library/windows/desktop/ff471356(v=vs.85).aspx). This is a superset of shader model 4 and adds new resources, compute shaders and tessellation.
-   [Shader Model 5.1](https://msdn.microsoft.com/en-us/library/windows/desktop/dn933277(v=vs.85).aspx). This is functionally very similar to Shader Model 5; the main change is more flexibility in resource selection by allowing indexing of arrays of descriptors from within a shader.
-   [Shader Model 6.0](https://github.com/microsoft/DirectXShaderCompiler/wiki/Shader-Model-6.0). This is a superset of shader model 5.1 with some deprecated language elements and with the addition of wave intrinsics and 64-bit integers for arithmetic.
-   [Shader Model 6.1](https://github.com/microsoft/DirectXShaderCompiler/wiki/Shader-Model-6.1). This is a superset of shader model 6.0 that adds support for SV_ViewID, barycentric semantics and the GetAttributeAtVertex intrinsic.
-   [Shader Model 6.2](https://github.com/microsoft/DirectXShaderCompiler/wiki/Shader-Model-6.2). Adds support for float16 (as opposed to minfloat16) and denorm mode selection.
-   [Shader Model 6.3](https://github.com/microsoft/DirectXShaderCompiler/wiki/Shader-Model-6.3). Adds support for DirectX Raytracing (DXR), including libraries and linking.
-   [Shader Model 6.4](https://github.com/microsoft/DirectXShaderCompiler/wiki/Shader-Model-6.4). Adds low-precision packed dot product intrinsics, and support for library sub-objects to simplify raytracing.



Differences between Direct3D 9 and Direct3D 10:
Direct3D 9 introduced shader models 1, 2, and 3.
Direct3D 10 introduced shader model 4.
Direct3D 10.1 introduced shader model 4.1.





## Unity对不同版本的定义

[https://docs.unity3d.com/Manual/SL-ShaderCompileTargets.html](https://docs.unity3d.com/Manual/SL-ShaderCompileTargets.html)

Here is the list of shader models supported, with roughly increasing set of capabilities (and in some cases higher platform/GPU requirements):

#### #pragma target 2.0

-   Works on all platforms supported by Unity. DX9 shader model 2.0.
-   Limited amount of arithmetic & texture instructions; 8 interpolators; no vertex texture sampling; no derivatives in **fragment shaders**
    ; no explicit **LOD**
     texture sampling.

#### #pragma target 2.5 (default)

-   Almost the same as 3.0 target (see below), except still only has 8 interpolators, and does not have explicit LOD texture sampling.
-   Compiles into DX11 feature level 9.3 on Windows Phone.

#### \#pragma target 3.0

-   DX9 shader model 3.0: derivative instructions, texture LOD sampling, 10 interpolators, more math/texture instructions allowed.
-   Not supported on DX11 feature level 9.x GPUs (e.g. most Windows Phone devices).
-   Might not be fully supported by some OpenGL ES 2.0 devices, depending on driver extensions present and features used.

#### #pragma target 3.5 (or es3.0)

-   OpenGL ES 3.0 capabilities (DX10 SM4.0 on D3D platforms, just without geometry shaders).
-   Not supported on DX11 9.x (WinPhone), OpenGL ES 2.0.
-   Supported on DX11+, OpenGL 3.2+, OpenGL ES 3+, Metal, Vulkan, PS4/XB1 consoles.
-   Native integer operations in shaders, texture arrays and so on.

#### #pragma target 4.0

-   DX11 shader model 4.0.
-   Not supported on DX11 9.x (WinPhone), OpenGL ES 2.0/3.0/3.1, Metal.
-   Supported on DX11+, OpenGL 3.2+, OpenGL ES 3.1+AEP, Vulkan, PS4/XB1 consoles.
-   Has geometry shaders and everything that `es3.0` target has.

#### #pragma target 4.5 (or es3.1)

-   OpenGL ES 3.1 capabilities (DX11 SM5.0 on D3D platforms, just without tessellation shaders).
-   Not supported on DX11 before SM5.0, OpenGL before 4.3 (i.e. Mac), OpenGL ES 2.0/3.0.
-   Supported on DX11+ SM5.0, OpenGL 4.3+, OpenGL ES 3.1, Metal, Vulkan, PS4/XB1 consoles.
-   Has compute shaders, random access texture writes, atomics and so on. No geometry or tessellation shaders.

#### #pragma target 4.6 (or gl4.1)

-   OpenGL 4.1 capabilities (DX11 SM5.0 on D3D platforms, just without compute shaders). This is basically the highest OpenGL level supported by Macs.
-   Not supported on DX11 before SM5.0, OpenGL before 4.1, OpenGL ES 2.0/3.0/3.1, Metal.
-   Supported on DX11+ SM5.0, OpenGL 4.1+, OpenGL ES 3.1+AEP, Vulkan, Metal (without geometry), PS4/XB1 consoles.

#### #pragma target 5.0

-   DX11 shader model 5.0.
-   Not supported on DX11 before SM5.0, OpenGL before 4.3 (i.e. Mac), OpenGL ES 2.0/3.0/3.1, Metal.
-   Supported on DX11+ SM5.0, OpenGL 4.3+, OpenGL ES 3.1+AEP, Vulkan, Metal (without geometry), PS4/XB1 consoles.



























