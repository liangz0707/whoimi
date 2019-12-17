# 不同图形API矩阵行列主序及其影响

矩阵的行主序和列主序影响了空间变换是矩阵乘法的使用方式。

Opengl的矩阵使用列主序，(参考资料](https://www.khronos.org/opengl/wiki/Data_Type_(GLSL))。

-   mat*n*x*m*: A matrix with *n* columns and *m* rows. OpenGL uses column-major matrices, which is standard for mathematics users. Example: mat3x4.
-   mat*n*: A matrix with *n* columns and *n* rows. Shorthand for mat*n*x*n*

例子：

```c
mat3 theMatrix;
theMatrix[1] = vec3(3.0, 3.0, 3.0); // Sets the second column to all 3.0s
theMatrix[2][0] = 16.0; // Sets the first entry of the third column to 16.0.
```

D3D使用的是行主序：

```c
matrix <float, 2, 2> fMatrix = { 0.0f, 0.1, // row 1
                                 2.1f, 2.2f // row 2
                               };
```

具体使用矩阵时，要注意CPU端buffer的定义方式和矩阵乘法统一即可。

使用不同的图形API时，CPU设置Buffer：

假如View矩阵在CPU端是行主序，直接Copy到Buffer，那么D3D是正常的，但是Opengl就变成了列主序。此时，View空间变换：D3D需要进行mul（V,vector）,Opengl就是需要mul（vector，V）。

使用不同的图形API时，Shader当中构造矩阵：

需要注意opengl的m01,指的是第一列第二行。

































































