##线性空间、GAMMA空间和HDR、纹理



### sRGB

sRGB是一个纹理的标记，也是一种**数据保存格式**。在对纹理采样和写入纹理内容时产生作用。如果开启sRGB，那么在采样时会自动的对采样得到的值进行sRGB解码，写入时会对写入的值进行sRGB编码，然后再写入。

(When calculating lighting on older hardware restricted to 8 bits per channel for the framebuffer format, using a gamma curve provides more precision in the human-perceivable range. More bits are used in the range where the human eye is the most sensitive.

Even though monitors today are digital, they still take a gamma-encoded signal as input. Image files and video files are explicitly encoded to be in gamma space (meaning they carry gamma-encoded values, not linear intensities). This is the standard; everything is in gamma space.) 所以视频的录制结果，PS，SP制作内容的保存一般应该都是sRGB格式。



unity中如果使用**Gamma空间**，sRGB的**设置失效**。

(When using Gamma color space, no conversions are done of any kind, and this setting is not used.)

sRGB (Color Texture)设置的意义：Check this box to specify that the Texture is stored in gamma space. This should always be checked for non-HDR color Textures (such as Albedo and Specular Color)（只有UI和Default会有这个选项, 至于HDR color Textures应该是PS输出的浮点格式纹理）. If the Texture stores information that has a specific meaning, and you need the exact values in the Shader (for example, the smoothness or the metalness), uncheck this box. This box is checked by default.

### Gamma矫正

如果显示设备按照线性亮度显示内容，人眼的感受会觉得很奇怪。

所以显示设备需要显示进过Gamma矫正的内容，以符合人眼感受习惯（More bits are used in the range where the human eye is the most sensitive.）。录制设置完成录制后保存的也是经过Gamma矫正的内容。

**目前，各种设备的输入、保存、输出格式都是在Gamma空间的。**

目前sRGB格式就是Gamma空间的标准。(The accepted standard for gamma space is called sRGB )

**This is the standard; everything is in gamma space.**也就是说一般情况（录制、显示、保存）下都不存在Linear空间。

所以根据纹理内容需要合理的设置纹理的sRGB标记。（如果是法线或者Mask）则Unity 不会进行任何编码解码操作，直接作为数据来处理。针对的只是颜色纹理（Color RenderTexture）.

### 线性空间和Gamma空间

显示、录制、保存都是Gamma空间，但是当纹理颜色参与计算时，为了精确度，需要在线性空间计算。

Unity提供了线性空间处理选项:在每一个渲染过程中，**输入需要进行sRGB的解码，输出时在进行sRGB编码**。（To overcome this, you can set Unity to use an RGB sampler to cross over from gamma to linear sampling. This ensures a linear workflow with all inputs and outputs of a Shader in the correct color space, resulting in a correct outcome.）

设置线性空间方法（go to **Edit** > **Project Settings** > **Player** and open **Player Settings**. Go to **Other Settings** > **Rendering** and change the **Color Space** to **Linear** or **Gamma**, depending on your preference.）

**如果不进行上述设置，Unity就不会进行格式的转化，采样到的就是保存值，输出的内容也会直接写入到纹理上。**



***特例1**:当使用**HDR**的时候颜色校正的时机（只在linear空间有效），Shader结果输出到帧缓存中时并未进行Gamma矫正，而是在更晚的时候进行（这一帧完全渲染结束进行一次Gamma矫正）。如果**Non-HDR** 输出到帧缓存的结果就是Gamma矫正的结果。（When using HDR, rendering is performed in linear space into floating point buffers. These buffers have enough precision not to require conversion to and from gamma space whenever the buffer is accessed. This means that when rendering in linear mode, the framebuffers you use store the colors in linear space. Therefore, all blending and post process effects are implicitly performed in linear space. When the final backbuffer is written to, gamma correction is applied.）

***特例2**:光照贴图，无论颜色空间的设置，光照贴图永远执行的都是**Linear Space的工作流**：The lighting **calculations in the lightmapper are always done in linear space** (see documentation on the [Lighting Window](https://docs.unity3d.com/Manual/GlobalIllumination.html) for more information). **The lightmaps are always stored in gamma space**. This means that the lightmap textures are identical no matter whether you’re in gamma or linear color space.

***特例3**:HDR中帧缓存的内容和颜色空间的关系. **When linear color space is enabled and HDR is not enabled**, a special framebuffer type is used that supports sRGB read and sRGB write (convert from gamma to linear when reading, convert from linear to gamma when writing). When this framebuffer is used for blending or it is bound as a Texture, the values are converted to linear space before being used. When these buffers are written to, the value that is being written is converted from linear space to gamma space. **If you are rendering in linear mode and non-HDR mode**, all post-process effects have their source and target buffers created with sRGB read and write enabled so that post-processing and post-process blending occur in linear space.

***注**：**Gamma工作流**：读入、写出内容时都不进行Gamma矫正：Even though these values are in gamma space, all the Unity Editor’s Shader calculations still treat their inputs as if they were in linear space. To ensure an acceptable final result, the Editor makes an adjustment to deal with the mismatched formats when it writes the Shader outputs to a framebuffer and does not apply gamma correction to the final result.

***注：**一旦打开HDR所有的帧缓存就会从ARGB32(255的整型RGBA)变成ARGBHalf(16位的浮点数)。原本没有HDR时每次帧缓存的读写都会经历sRGB的解码编码。打开HDR后帧缓存的读写不再进行sRGB的编解码，而是在这一帧完全处理完成之后，从HDR转回正常颜色空间后在进行sRGB的编解码，ReadPixel也就是从这里读取纹理的。实际上我们可以在渲染过程中读取colorbuffer，例如渲染到RT当中，这个使用可以读取到没有经过编码颜色数据。



### 纹理颜色空间(sRGB选项控制)

不同颜色空间的工作流只针对Non-Hdr的Color Texture 在Texture的Inspector当中可以看到。sprite和Default有sRGB的选项，其他没有的不会受到颜色空间设置的影响。