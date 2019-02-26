## 1 旋向性

两个坐标轴有相同的旋向性：可以通过旋转让两个坐标轴重合。

左右手坐标系有不同的选项性。

## 2 左右手坐标系

拇指、食指、中指分别代表，+x +y +z轴。由此左右手会产生两个不同的坐标系。

## 3 左右手法则

在cross叉乘中，需要通过正旋向来确定结果轴的指向：

**在左右坐标系中，用左手，在右手坐标系中用右手。**

## 4 unity的坐标系

模型空间和世界空间是左手坐标系。

相机空间是右手坐标系。这相机对准了z轴负方向。也就是说z轴坐标的减少意味着场景深度的增加。

## 5. 矩阵操作：

UNITY_MATRIX_P 等价于 [GL](http://resetoter.cn/UnityDoc/ScriptReference/GL.html).GetGPUProjectionMatrix

Camera.worldToCameraMatrix  ：Note that camera space matches OpenGL convention: camera's forward is the negative Z axis. This is different from Unity's convention, where forward is the positive Z axis.

Camera.cameraToWorldMatrix：Note that camera space matches OpenGL convention: camera's forward is the negative Z axis. This is different from Unity's convention, where forward is the positive Z axis.



```
UNITY_MATRIX_M
```

- This is identical to `unity_ObjectToWorld`, also called the Model transform

## 6. unity_XX矩阵

shader当中unity_XX（Camera相关）开头的矩阵不一定是当前摄像机矩阵，在官方文档当中也没有提到。

1. 根据实验unity_WorldToCamera 和 UNITY_MATRIX_V 的区别在于：

   而UNITY_MATRIX_V改变了旋向性，第三行z值取反了。

   也就是说unity_WorldToCamera 没有改变旋向性。

2. 这个时候UNITY_MATRIX_P（glstate_matrix_projection）是根据平台特殊处理的矩阵。

而unity_CameraProjcetion是按照正常的流程的的矩阵。



unity c#的代码中通过 GL.get...接口的到的矩阵等价于UNITY_MATRIX_P（glstate_matrix_projection）：省略旋向性改变。约束y方向、约束z方向。更加复杂。

通过Camera.projectionMatrix得到的是一般投影矩阵unity_CamearaProjection。