

# Vulkan Shader编译与加载过程

##  间接

参考了[Vulkan教程](https://vulkan-tutorial.com/Drawing_a_triangle/Graphics_pipeline_basics/Shader_modules)

​	和一般的图形API不同（GL\DX），Vulkan 的shader代码是通过字节码的格式保存的，而非具备可读性的HLSL和GLSL。字节码的格式交过SPIR-V，它设计出来和Vulkan和OpenCL一同使用。可以用来编写图形Shader和计算Shader，这部分专注于图形Shader。

使用字节码的优点是由GPU供应商提供的把Shader代码转换成本地代码的编译器能复杂度更低。

过去使用可读代码的风险，编译器实现灵活，尝试特性实现不同：The past has shown that with human-readable syntax like GLSL, some GPU vendors were rather flexible with their interpretation of the standard. If you happen to write non-trivial shaders with a GPU from one of these vendors, then you'd risk other vendor's drivers rejecting your code due to syntax errors, or worse, your shader running differently because of compiler bugs. With a straightforward bytecode format like SPIR-V that will hopefully be avoided.

我们不需要手写字节码，而是需要用与供应商独立的编译器把GLSL转化成SPIR-V.编译器用来验证Shader代码是完全标准的，并且声称SPIR-V代码集成在程序中。编译器提供了library用来集成在程序中，也可以用现有编译好的程序工具。

我们可以直接使用编译器glslangValidator.exe，不过这里使用glslc工具，因为这个工具提供了includes功能，并且它的参数和gcc和Clang很相似。这些已经集成在了VulkanSDK当中。

后面先展示GLSL代码，然后转换成SPIR-V，并在运行时加载。

GLSL语法：[语法简介](http://nehe.gamedev.net/article/glsl_an_introduction/25007/)

## Shader基本使用

### 编写GLSL代码

首先先编写GLSL代码文件shader.vert：

```c
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) out vec3 fragColor;

vec2 positions[3] = vec2[](
    vec2(0.0, -0.5),
    vec2(0.5, 0.5),
    vec2(-0.5, 0.5)
);

vec3 colors[3] = vec3[](
    vec3(1.0, 0.0, 0.0),
    vec3(0.0, 1.0, 0.0),
    vec3(0.0, 0.0, 1.0)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
    fragColor = colors[gl_VertexIndex];
}
```

### 编译成字节码

代码写完后使用编译器的工具编译成字节码：

```c
C:/VulkanSDK/x.x.x.x/Bin32/glslc.exe shader.vert -o vert.spv
```

之后我们就得到了字节码文件vert.spv。

### 程序运行时加载字节码文件

需要用于渲染是加载字节码文件：

```c
static std::vector<char> readFile(const std::string& filename) {
    std::ifstream file(filename, std::ios::ate | std::ios::binary);

    if (!file.is_open()) {
        throw std::runtime_error("failed to open file!");
    }
    
    size_t fileSize = (size_t) file.tellg();
std::vector<char> buffer(fileSize);
    file.seekg(0);
file.read(buffer.data(), fileSize);
    file.close();

	return buffer;
}
```

### 根据字节码数据穿件Shader Modules

这部分内容就进入了图形API的部分。根据ShaderModule就可以吧这个Shader代码设置成渲染状态的一部分。

## 进一步分析

SPIR-V是二进制的中间吗，优点是不需要考虑供应商特性。缺点是需要考虑如何将shader语言编译成SPIR-V.

最好的方案是：[Khronos' glslang library](https://github.com/KhronosGroup/glslang) 和 [Google's shaderc](https://github.com/google/shaderc).

都是开源的Shader编译库。

使用GLSL到SPIR-V的优点是不需要在程序中发布Shader代码

### 一种GLSL不同的版本

GLSL有很多不同的版本：

-   Modern mobile (GLES3+)
-   Legacy mobile (GLES2)
-   Modern desktop (GL3+)
-   Legacy desktop (GL2)
-   Vulkan GLSL

Vulkan当中有很多和其他变体不兼容的地方

-   Descriptor sets, no such concept in OpenGL
-   Push constants, no such concept in OpenGL, but very similar to "uniform vec4 UsuallyRegisterMappedUniform;"
-   Subpass input attachments, maps well to [Pixel Local Storage on ARM Mali GPUs](https://community.arm.com/graphics/b/blog/posts/pixel-local-storage-on-arm-mali-gpus)
-   gl_InstanceIndex vs gl_InstanceID. Same, but Vulkan GLSL's version InstanceIndex (finally!) adds base instance offset for you



## 变体策略

等待补充



















