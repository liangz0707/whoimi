# UE4基础源码Pipeline部分

这部分主要记录源代码中和Pipeline相关的部分。

# FSceneRenderer

 * Used as the scope for scene rendering functions.
 * It is initialized in the game thread by FSceneViewFamily::BeginRender, and then passed to the rendering thread.
 * The rendering thread calls Render(), and deletes the scene renderer when it returns.



有两个主要的继承类：FMobileSceneRenderer和FDeferredShadingSceneRenderer。手机不支持Deferred（MRT），所以都是forward渲染。

# FMobileSceneRenderer

* Renderer that implements simple forward shading and associated features.

  

# FDeferredShadingSceneRenderer

* Scene renderer that implements a deferred shading pipeline and associated features.