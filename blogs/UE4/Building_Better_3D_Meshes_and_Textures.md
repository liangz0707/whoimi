# Building Better 3D Meshes and Textures

Meshes And Textures  are basic assets. It's important to control the asset detail to get better game performance.

## Naming Conventions

First of all, I want to discuss  is Naming Conventions . It's Great you can know what they are when you read the asset list by reading their name.

There are examples. We can prefix with a couple of letters to  help us  quickly identify what type of the data is.

SM_Rock_00 - **S**tatic **M**esh for a **Rock** variation **00**

T_Rock_00_BC - **B**ase **C**olor **T**exture for a **Rock** variation **00**

SKM_RockBun_00 - **Sk**eletal **M**esh

## Texture Creation

We prefer to: (GPU like it)

Textures should always be a power of 2.

Textures do not have to be square, just use a power of 2. 

If you go with power of two texture the advantage is that Engine will generate **mip map chain** for you that will really help in terms of performance and quality.

We cant bring a non power of 2 texture but  Engine will do some like padding internally so that like square, so you end up with wasting space  

**UI** stuff non power of 2 is not so bad, because those are alway at full resolution.

We have a very flexible in the type of texture like:

```c
PNG- Embed Alpha Support
PSD- Embed Alpha Support
TGA- Embed Alpha Support
BMP
FLOAT
PCX
IPG
EXR
DDS - Cubemap Texture (32 Bit / Channels 8.8.8.8. ARGB 32 bppm unsigned)
HDR - Cubemap Texture (LongLat Unwrap)
```

JPEG is compressed ,So you will let go some quality. because Engine will recompress over JPEG.

## Texture Import

Drag into editor Or Use import Panel.

## Alpha Import

  In terms of storage, splitting the alpha from the texture make that we can use different size of alpha. if we have limited space.

There are something need be known

LOD bias.

Channel Packing.

Srgb.

## Mip Mapping

Base Version is full resolution, other are smaller.

Mip Gen Setting:  Filtering.

LOD bias.

Texture Group: Pre-configure texture based on their usage.

 ## Texture Compression

Main modes are：

DXT1 -  No Alpha

DXT5 -  Alpha

these are tile-based compression.

There ara some special mode for HDR texture.

bc7 -  compression time is longer but you have higher quality over the DXT1 and DXT5.

## System Unit

UE4 use Centimeters for unity of measurement.

Make sure set DCC unit to centimeters

## Triangle Counts

Do not model small detials.

You should constantly check the triangle count.

Engine can deal with a lot of polygon ,but we should prepare scene  properly for that.

## Material ID

As you model your object , you can specify which polygons use which material in your mesh.   Engine treat material id as separate objects. GPU rendering this robot is equivalent to rendering two objects.

Material ID in unity is corresponding to  submeshes.

























