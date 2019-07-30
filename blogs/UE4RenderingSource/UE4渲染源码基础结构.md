# UE4 Rendering SourceCode

开始阅读UE4 渲染相关的引擎源码。

源代码目录：

\Source\Runtime\Renderer\

**这部分主要记录一些基本的数据类型的代码。**

## FSceneView

* A projection from scene space into a 2D screen region.

将场景空间转换到屏幕空间的类，**直接对应了一个View（一个Camera）**。

内部保存了ViewMatrix 和ProjectionMatrix。

一个屏幕上可以用有多个View。

和Camera相关的参数设置等，这里应该就是对应一个Camera。

一些参数，都适合具体摄像机相关的内容：

```c
/** Actual field of view and that desired by the camera originally */
float FOV;
FLinearColor BackgroundColor;
FLinearColor OverlayColor;
/** Color scale multiplier used during post processing */
FLinearColor ColorScale;

/** Maximum number of shadow cascades to render with. */
int32 MaxShadowCascades;

FViewMatrices ViewMatrices;

/** Variables used to determine the view matrix */
FVector		ViewLocation;
FRotator	ViewRotation;
FQuat		BaseHmdOrientation;
FVector		BaseHmdLocation;
float		WorldToMetersScale;

// normally the same as ViewMatrices unless "r.Shadow.FreezeCamera" is activated
FViewMatrices ShadowViewMatrices;
```

## FViewMatrices

**和摄像机相关的矩阵**：

ViewMatrix和ProjectionMatrix以及相关的数据和操作：

```c
//构造函数
	FViewMatrices()
	{
		ProjectionMatrix.SetIdentity();
		ViewMatrix.SetIdentity();
		HMDViewMatrixNoRoll.SetIdentity();
		TranslatedViewMatrix.SetIdentity();
		TranslatedViewProjectionMatrix.SetIdentity();
		InvTranslatedViewProjectionMatrix.SetIdentity();
		PreViewTranslation = FVector::ZeroVector;
		ViewOrigin = FVector::ZeroVector;
		ProjectionScale = FVector2D::ZeroVector;
		TemporalAAProjectionJitter = FVector2D::ZeroVector;
		ScreenScale = 1.f;
	}
```

## FSceneViewFamily

* A set of views into a scene which only have different view transforms and owner actors.

**保存所有的FSceneView，对应一个RenderTarget**.

```c
TArray<const FSceneView*> Views;
```

一个FSceneViewFamily当中的物体会渲染到同一个RenderTarget上，类似Unity的多个View。

```c
/** The render target which the views are being rendered to. */
const FRenderTarget* RenderTarget;
```

要绘制的内容保存在FSceneInterface当中：

```c
/** The scene being viewed. */
FSceneInterface* Scene;
```

其他和一个视口相关的内容：自动曝光。Gamma矫正、截屏内容等。

```c
/** Gamma correction used when rendering this family. Default is 1.0 */
float GammaCorrection;

/** Editor setting to allow designers to override the automatic expose. 0:Automatic, following indices: -4 .. +4 */
FExposureSettings ExposureSettings;
/** 
* Which component of the scene rendering should be output to the final render target.
* If SCS_FinalColorLDR this indicates do nothing.
*/
ESceneCaptureSource SceneCaptureSource;

/** When enabled, the scene capture will composite into the render target instead of overwriting its contents. */
ESceneCaptureCompositeMode SceneCaptureCompositeMode;
```

## FSceneInterface

 * An interface to the private scene manager implementation of a scene.  Use GetRendererModule().AllocateScene to create.
 * The scene

**场景和场景管理的接口，这部分没有实现代码只有接口。**

把UPrimitiveComponent加入FSceneInterface，这里应该是maintread和renderthread的一个交互位置。

UPrimitiveComponent的增删改查，UPrimitiveComponent应该就是场景当中的渲染对象：

```c

virtual void AddPrimitive(UPrimitiveComponent* Primitive) = 0;

virtual void RemovePrimitive(UPrimitiveComponent* Primitive) = 0;

virtual void ReleasePrimitive(UPrimitiveComponent* Primitive) = 0;

virtual bool GetPreviousLocalToWorld(const FPrimitiveSceneInfo* PrimitiveSceneInfo, FMatrix& OutPreviousLocalToWorld) const { return false; }

```

灯光相关的管理：

```c
virtual void AddLight(ULightComponent* Light) = 0;

virtual void RemoveLight(ULightComponent* Light) = 0;

virtual void AddInvisibleLight(ULightComponent* Light) = 0;
virtual void SetSkyLight(FSkyLightSceneProxy* Light) = 0;
virtual void DisableSkyLight(FSkyLightSceneProxy* Light) = 0;
```

Decal、反射探针、雾、风场、SpeedTree、特效等的管理。

从开头声明的类可以看出他管理的内容。

```c
// 渲染相关内容
class AWorldSettings;
class FAtmosphericFogSceneInfo;
class FMaterial;
class FMaterialShaderMap;
class FPrimitiveSceneInfo;
class FRenderResource;
class FRenderTarget;
class FSkyLightSceneProxy;
class FTexture;
class FVertexFactory;
// 场景资源
class UDecalComponent;
class ULightComponent;
class UPlanarReflectionComponent;
class UPrimitiveComponent;
class UReflectionCaptureComponent;
class USkyLightComponent;
class UStaticMeshComponent;
class UTextureCube;
```

## UPrimitiveComponent 略

 * PrimitiveComponents are SceneComponents that contain or generate some sort of geometry, generally to be rendered or used as collision data.
 * There are several subclasses for the various types of geometry, but the most common by far are the ShapeComponents (Capsule, Sphere, Box), StaticMeshComponent, and SkeletalMeshComponent.
 * ShapeComponents generate geometry that is used for collision detection but are not rendered, while StaticMeshComponents and SkeletalMeshComponents contain pre-built geometry that is rendered, but can also be used for collision detection.

描述用于渲染或者碰撞计算的组件，描述多种几何体类型：ShapeComponents 、StaticMeshComponent、SkeletalMeshComponent。

ShapeComponents 主要用于碰撞但是不渲染。

StaticMeshComponents and SkeletalMeshComponents 既可以用于碰撞也可以用于渲染。



## FRenderResource

* A rendering resource which is owned by the rendering thread.

渲染线程拥有的资源。

和RHI有关的初始化等内容。

具体的实现包括：

### FTexture

* A textures resource.

### FTextureReference

* A texture reference resource.

### FVertexBuffer

* A vertex buffer resource

### FNullColorVertexBuffer

* A vertex buffer with a single color component.  This is used on meshes that don't have a color component to keep from needing a separate vertex factory to handle this case.

### FIndexBuffer

* An index buffer resource.

### 和FrenderResource无关的

FGlobalDynamicVertexBuffer

FGlobalDynamicIndexBuffer

### FVertexFactory

* Encapsulates a vertex data source which can be linked into a vertex shader.

