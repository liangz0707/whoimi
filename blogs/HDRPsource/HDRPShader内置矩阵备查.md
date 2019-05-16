HDRP当中设置矩阵的代码：

HDCamera.cs

```c
{
    bool taaEnabled = m_frameSettings.IsEnabled(FrameSettingsField.Postprocess)
        && antialiasing == AntialiasingMode.TemporalAntialiasing
        && camera.cameraType == CameraType.Game;

    cmd.SetGlobalMatrix(HDShaderIDs._ViewMatrix,                viewMatrix);
    cmd.SetGlobalMatrix(HDShaderIDs._InvViewMatrix,             viewMatrix.inverse);
    cmd.SetGlobalMatrix(HDShaderIDs._ProjMatrix,                projMatrix);
    cmd.SetGlobalMatrix(HDShaderIDs._InvProjMatrix,             projMatrix.inverse);
    cmd.SetGlobalMatrix(HDShaderIDs._ViewProjMatrix,            viewProjMatrix);
    cmd.SetGlobalMatrix(HDShaderIDs._InvViewProjMatrix,         viewProjMatrix.inverse);
    cmd.SetGlobalMatrix(HDShaderIDs._NonJitteredViewProjMatrix, nonJitteredViewProjMatrix);
    cmd.SetGlobalMatrix(HDShaderIDs._PrevViewProjMatrix,        prevViewProjMatrix);
    cmd.SetGlobalMatrix(HDShaderIDs._CameraViewProjMatrix,      viewProjMatrix);
    cmd.SetGlobalVector(HDShaderIDs._WorldSpaceCameraPos,       worldSpaceCameraPos);
    cmd.SetGlobalVector(HDShaderIDs._PrevCamPosRWS,             prevWorldSpaceCameraPos);
    cmd.SetGlobalVector(HDShaderIDs._ScreenSize,                screenSize);
    cmd.SetGlobalVector(HDShaderIDs._ScreenToTargetScale,       doubleBufferedViewportScale);
    cmd.SetGlobalVector(HDShaderIDs._ScreenToTargetScaleHistory, doubleBufferedViewportScaleHistory);
    cmd.SetGlobalVector(HDShaderIDs._ZBufferParams,             zBufferParams);
    cmd.SetGlobalVector(HDShaderIDs._ProjectionParams,          projectionParams);
    cmd.SetGlobalVector(HDShaderIDs.unity_OrthoParams,          unity_OrthoParams);
    cmd.SetGlobalVector(HDShaderIDs._ScreenParams,              screenParams);
    cmd.SetGlobalVector(HDShaderIDs._TaaFrameInfo,              new Vector4(taaFrameRotation.x, taaFrameRotation.y, taaFrameIndex, taaEnabled ? 1 : 0));
    cmd.SetGlobalVector(HDShaderIDs._TaaJitterStrength,         taaJitter);
    cmd.SetGlobalVectorArray(HDShaderIDs._FrustumPlanes,        frustumPlaneEquations);
```

矩阵在Shader当中的代码:

ShaderVariables.hlsl

```c

// Define that before including all the sub systems ShaderVariablesXXX.hlsl files in order to include constant buffer properties.
#define SHADER_VARIABLES_INCLUDE_CB

// Important: please use macros or functions to access the CBuffer data.
// The member names and data layout can (and will) change!
CBUFFER_START(UnityGlobal)
    // ================================
    //     PER VIEW CONSTANTS
    // ================================
    // TODO: all affine matrices should be 3x4.
    float4x4 _ViewMatrix;
float4x4 _InvViewMatrix;
float4x4 _ProjMatrix;
float4x4 _InvProjMatrix;
float4x4 _ViewProjMatrix;
float4x4 _CameraViewProjMatrix;
float4x4 _InvViewProjMatrix;
float4x4 _NonJitteredViewProjMatrix;
float4x4 _PrevViewProjMatrix;       // non-jittered

float4 _TextureWidthScaling; // 0.5 for SinglePassDoubleWide (stereo) and 1.0 otherwise

// TODO: put commonly used vars together (below), and then sort them by the frequency of use (descending).
// Note: a matrix is 4 * 4 * 4 = 64 bytes (1x cache line), so no need to sort those.
#ifndef USING_STEREO_MATRICES
float3 _WorldSpaceCameraPos;
float  _Pad0;
float3 _PrevCamPosRWS;
float  _Pad1;
#endif
float4 _ScreenSize;                 // { w, h, 1 / w, 1 / h }

// Those two uniforms are specific to the RTHandle system
float4 _ScreenToTargetScale;        // { w / RTHandle.maxWidth, h / RTHandle.maxHeight } : xy = currFrame, zw = prevFrame
float4 _ScreenToTargetScaleHistory; // Same as above but the RTHandle handle size is that of the history buffer

```

上面矩阵的宏定义:

ShaderVariablesMatrixDefsHDCamera.hlsl

```c
#ifdef UNITY_SHADER_VARIABLES_MATRIX_DEFS_LEGACY_UNITY_INCLUDED
    #error Mixing HDCamera and legacy Unity matrix definitions
#endif

#ifndef UNITY_SHADER_VARIABLES_MATRIX_DEFS_HDCAMERA_INCLUDED
#define UNITY_SHADER_VARIABLES_MATRIX_DEFS_HDCAMERA_INCLUDED

#if defined(USING_STEREO_MATRICES)

#define UNITY_MATRIX_V     _ViewMatrixStereo[unity_StereoEyeIndex]
#define UNITY_MATRIX_I_V   _InvViewMatrixStereo[unity_StereoEyeIndex]
#define UNITY_MATRIX_P     OptimizeProjectionMatrix(_ProjMatrixStereo[unity_StereoEyeIndex])
#define UNITY_MATRIX_I_P   _InvProjMatrixStereo[unity_StereoEyeIndex]
#define UNITY_MATRIX_VP    _ViewProjMatrixStereo[unity_StereoEyeIndex]
#define UNITY_MATRIX_I_VP  _InvViewProjMatrixStereo[unity_StereoEyeIndex]
#define UNITY_MATRIX_UNJITTERED_VP _ViewProjMatrixStereo[unity_StereoEyeIndex] // Since VR doesn't need to add jitter, just use normal VP matrix
#define UNITY_MATRIX_PREV_VP _PrevViewProjMatrixStereo[unity_StereoEyeIndex]

#else

#define UNITY_MATRIX_V     _ViewMatrix
#define UNITY_MATRIX_I_V   _InvViewMatrix
#define UNITY_MATRIX_P     OptimizeProjectionMatrix(_ProjMatrix)
#define UNITY_MATRIX_I_P   _InvProjMatrix
#define UNITY_MATRIX_VP    _ViewProjMatrix
#define UNITY_MATRIX_I_VP  _InvViewProjMatrix
#define UNITY_MATRIX_UNJITTERED_VP _NonJitteredViewProjMatrix
#define UNITY_MATRIX_PREV_VP _PrevViewProjMatrix

#endif // USING_STEREO_MATRICES

#endif // UNITY_SHADER_VARIABLES_MATRIX_DEFS_HDCAMERA_INCLUDED
```

