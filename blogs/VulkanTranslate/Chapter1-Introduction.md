# Chapter 1 Introduction

这篇文档主要是一份说明文档，用来描述Vulkan API。 Vulkan API是符合 C99标准的控制low-level图形计算功能的API。

文档的权威版本在官方网站可以查阅：[官方文档](https://www.khronos.org/registry/vulkan/)

生成官方文档的工程：[文档工程](https://github.com/ KhronosGroup/Vulkan-Docs)

## 1.1 文档约定

Vulkan说明文档用与给API开发人员和应用开发人员提供API使用参考，为两者提供统一的约定。从上下文通常也能推断出面向的受众，某些部分只针对这部分开发人员。例如 Valid Usage章节只提供给应用开发人员使用。任何的需求、禁止、建议和选择都在Normative terminology中说明。

> Note
>
>  extensions 中定义的结构体、枚举类型已经被集成到了Vulkan 1.1的核心库中。这些内容现在和Vulkan1.1的接口等价。和部分影响了Vulkan说明文件、头文件、和XML注册表。