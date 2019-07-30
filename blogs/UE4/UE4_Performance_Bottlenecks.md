# UE4 GPU profile

This post describe  Pipeline and Render Bottlenecks of UE4.

## Frame

To display a frame both calculations on the GPU and CPU need to be finished.

After, All the game code finished and all the pixel shaded, we can dispatch a frame to the screen.

The time of cost is the bigger number between GPU and CPU.

### GPU cost

On the GPU, we have parallelism and a pipeline.

Parallel - many core work at same time.  Core are best utilized when working on same task.

Pipeline- frame divided into steps.  Pixel Shader can't continue before it's fed with data from Vertex Shader.

Vertex Shader -> Tesselation -> Geometry -> Pixel Shader.

### Draw call

CPU sends commands - "draw calls "  to control the GPU.

more mesh. more materials ==  More draw calls.

If you want to draw a triangle set with a different material, you have to dispatch a GPU command from CPU to the GPU first, and goes through the driver , and then is translated, and only the transmitted to the GPU.

### Pixel- bound sources of trouble

### Resolution of screen.

Pixel most slowest part of the pipeline.

Depend resolution of screen.

Heavy shader, post processing, translucency

### Quad overDraw

Quad means a block of 4 pixel(2*2)

Many operations on the GPU are done on full quads or even bigger tiles-not single pixels

Small polygon waste GPU time. More small or more thin then triangle the bigger the problem.

Triangle cunt is rarly a problem.

Avoiding small and thin triangle :  Use Lods. Keep polygons bigger and even.

### Triangle Count 

explode after tessllation.

shadow casting need copy a mesh for drawing shadow mask.

### Hard edges ,UV splits,Smooth group

add vertex count.

### Memory-related sources

### Texture Sample

Too many texture sampler use up the bandwidth.

Texture packing and Compression.

### Texture Cache

fetch data from nearby cache.  Need UV continue.

UV  noise is bad for cache.

### Streaming cache

Good for memory. It's not related to the GPU cache.

