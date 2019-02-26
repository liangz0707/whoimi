# Shader打包

总结shader打包过程当中遇到的shader重复，编译时间过长的问题。

和shader相关的**开销**和以下几个方面有关：

* Other/ShaderLab的大小：编译大小
* Asset/Shader的大小和重复出现情况：资源当中shader数量
* Shader的编译时间：编译时间

shader 的打包**策略**主要有一下几种：

* Shader单独打包
* Shader不打包，跟随依赖打入对其依赖资源的包：
* 放置到Always Include中

下面描述上面2、3两种打包策略可能会造成的**问题**：

1.Always Include会造成**大量的Shader编译时间**(在build阶段），会编译所有的shader变体。**ShaderLab很大。**

2.Shader不打包可能会造成，Shader可能会存在多个版本，**Asset/Shader**数量变多。但是一个材质也只会启用一两个变体（shader_feature），所以ShaderLab不会太大。

不同shader打包策略的细节情况：

1. 如果Always include （无论shader是否打包）则在**打AB包时**会编译所有变体。在**build场景时**也会编译所有变体（无论场景是否使用这个shader）。

   1.1 如果这个shader打了AB包，但是没有加载这个AB包。则**Asset/Shader**中只有一份，**ShaderLab**包含了所有变体。

   1.2 如果这个shader打了AB包，同时加载了AB包（确实使用了，比如在材质中使用），则**Asset/Shader**和**ShaderLab**都会翻一倍。

   1.3 如果没有打包，但是加载了对这个shader有依赖的内容，则有几个依赖会翻几倍（**这个shader的全部变体都会翻倍**）。**结论**：a.AlwayInclude会使shader_feature编程multicompile全部编译，b.同时不管依赖情况如何，所有变体都会编译。

2. 如果不用Always Include，如果不对shader打包，则每一个依赖都会生成一份shader的**Asset/Shader**，同时根据各自shader_feature的启用情况，生成各自所需要的**ShaderLab**的。

3. 如果不用Always Include，如果对shader打包，则只生成一份shader的**Asset/Shader**，他所需要的所有生效的shader_feature决定一份总的**ShaderLab**。

## 关于FallBack依赖

1. 如果Shader有多份（无论什么原因），FallBack Shader如果不打包，则FallBack Shader必定有多份。
2. 无论Shader在AB包中存在了多少份（单独打一个包或被依赖资源打入多个包），如果FallBack Shader打包，则AB包中的这个Shader使用的FallBackShader只有一份，但是如果这个Shader出现在场景中，则Shader会多一份，FallBack Shader会多一份。

在实际打包过程中需要注意的是，我们无法对内置shader进行打包，只能使用AlwaysInclude所以需要。尤其是依赖的Shader版本，是否在工程中。

## 关于Dependency依赖

和FallBack一致



**注：**

1. 场景当中DIsable的GameObject对应的Material的材质一样会被编译进入场景。

2. **多个独立打包的shader的同一个Fallback会产生多份。**放在工程目录任何位置FallBack都会生效。





# 地形shdaer的特殊依赖

1.如果把默认地形Shader放入AB包，不需要进行设置则可以自动使用本地的Shader

​	针对Nature/Terrain/... shader

2.如果AlwayInclude也可以达到一样的效果。

3.一般Shader的FallBack直接放在工程里即可，但是地形Shader的Dependency 的Hidde/***无法Alwaysinclude，单独打包也不会被引用。

4.注意啊 如果Nature已经放入工程，则工程内的地形只会使用这个，在使用AlwayInlcude包括内置的Natureshadr就没有意义了。

