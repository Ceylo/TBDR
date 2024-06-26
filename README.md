#  Metal Blend Pipelines

This project is an experiment to try several methods to blend many textures, a problem commonly found in drawing software.
It focuses on using Apple's Metal APIs in various ways: with fragment shaders, compute shaders or higher-level (but still Metal based) APIs like Core Image and Metal Petal.
A big motivation here has been to understand how Core Image could be so fast compared to basic usage of compute shaders, and whether it's possible to do better or as good, but without the downsides of Core Image. I was also curious to try Apple Silicon specific optimizations like tile memory usage.

The solution has to provide correct and pixel accurate image result, color managed and with at least 16f internal pixel format.
The different pipelines get compared in terms of speed and memory usage.

### Dataset and testing conditions

50 textures get blended then the result is displayed. Each one is RGBA, 8bits/sample, 4000x2000 pixels.
This data alone, common for all methods, uses ~1.5GB.
The blend operation is a basic `(background + foreground) / 1.1`.

The results listed later below are generated on a 2021 MacBook Pro 14" with 32GB memory and M1 Pro Soc, 16 GPU cores, running macOS 14.5. Image result is displayed in a full size window on a UHD@144Hz display, making the Metal drawable 3840x1836 pixels.

<div align="center">
  <img width="1070" alt="image" src="https://github.com/Ceylo/Metal-Blend-Pipelines/assets/451334/afc9d421-6eee-4fc2-8ba9-53b09ed0a11b">
  
  *A preview of the testing app*
</div>

## Pipelines

_Note: in all pipelines, the last output texture is the MTKView's drawable itself, so there's no additional command needed to display the result._

### Core Image framework

The simplest approach when you just want to think in terms of image composition graph with good flexilibility on the operations to perform. It is a rather efficient solution since Core Image will merge the operations as much as possible, reducing synchronization, memory latency and memory bandwidth issues. Unfortunately it also has a rather high memory usage, non-debuggable shaders and unpredictable stutters when it decides to optimize the rendering graph, as it involves compiling a new Metal shader at runtime.

The full image composition graph is generated, providing a single `CIImage`. This is then rendered through `CIContext.startTask()` which will encode all the rendering work in the provided `MTLCommandBuffer`.

### Compute (1 encoder, 1 dispatch/layer)

This is the most basic approach with Metal only. For each blend operation, a compute kernel is run, reading 1 pixel from 2 input textures, and writing the blended result to a 3rd output texture. This output is then the first input of the next blend operation, along with another layer's texture, and so on.

A single `MTLComputeCommandEncoder` is used, and each blend is done by a distinct `dispathThreads()` call.

### Render (1 encoder/layer, 1 draw/encoder)

Same as with above compute pipeline, except that a fragment shader is used instead of a compute kernel for each blend operation. Another difference is that one `MTLRenderCommandEncoder` **per blend operation** is used. Using a single `MTLRenderCommandEncoder` and 1 draw/blend gives random artifacts as one is not allowed to read & write to/from the same texture within a single render encoder. This happens as, starting from the 2nd layer to blend, the output texture of the 1st blend is the input of the 2nd blend, but the output on this 2nd blend operation is still the same output texture. The issue doesn't happen with `MTLComputeCommandEncoder`, most likely because the output texture is guaranteed to be fully written before the next pass is executed. This makes compute shaders more reliable but also slower due to the added synchronization.

### Render with tile memory (1 encoder, 1 draw/layer)

This one is a variation of the above render pipeline. However using tile memory implies using memoryless textures, which only "persist" within a single `MTLRenderCommandEncoder`. So we need to find a way not to run into the undefined behavior caused by reading & writing to the same texture. Documentation is lacking on the topic so I used best guess. What I ended up doing is using 2 memoryless textures, and swapping them on each subpass of the render encoder. So on the 2nd blend we read from `memoryless1` and `layer2`, and write to `memoryless2`. On the next blend we read from `memoryless2` and `layer3`, and write to `memoryless1`.
> [!IMPORTANT]
> I could not find yet documentation stating that this approach is well defined behavior, although on all devices I tried this gave correct and stable results.

### Render (1 encoder, 1 draw/layer)

Now successful in using a single `MTLRenderCommandEncoder` to blend the 50 layers, we can now use the same approach but without memoryless textures, to see how much the tile memory will actually affect the results.

### Compute (1 encoder, 1 dispatch/layer, 4 tiles)

After having tried render pipelines, let's get back to something closer to what Core Image is doing: a solution fully based on compute kernels. We found out that `MTLComputeCommandEncoder` allows us to use a single encoder out of the box because it guarantees dependencies between reads and writes of several dispatches. So let's try to allow Metal drivers to hide synchronization latency by running 4 fully independent pipelines, by splitting the output in 4 tiles, and only writing to a common `MTLTexture` in the final blend. This shoud allow the 4 pipeline to run in parallel, and one pipeline to move forward while the other waits on memory writes.

### Monolithic compute (1 encoder, 1 dispatch)

Since above approach didn't help, let's try to get even closer to what Core Image is doing: by blending all layers within a single compute kernel. This is not flexible at all but it'll still be interesting to know how this affects speed.

### Compute with threadgroup memory/imageblocks (1? encoder, 1? dispatch/layer)

_TODO_

### [Metal Petal framework](https://github.com/MetalPetal/MetalPetal)

This is a third-party framework very similar to Apple's Core Image. However it relies on fragment shaders in addition to compute kernels, and claims to be faster than Core Image.
The pipeline setup is almost identical to the one in Core Image, except for a few points:
- We don't need to (un)flip the output image
- The API makes it safer to work with unpremultiplied alpha textures
- We can't tell it to use our already existing `MTLCommandBuffer`. It will always create its own, from its own `MTLCommandQueue`. This prevents us from controlling how the Metal Petal shaders will be executed in the middle of our own Metal pipeline.

## Results
Now that we know what we're comparing, let's see results!

<img width="808" alt="Frame times" src="https://github.com/Ceylo/Metal-Blend-Pipelines/assets/451334/45172207-46fe-4541-962b-55b5a4041f47">
<img width="804" alt="Memory usage" src="https://github.com/Ceylo/Metal-Blend-Pipelines/assets/451334/fe17909b-ac3c-4a19-98ce-24e774e03a33">
<img width="1037" alt="Results table" src="https://github.com/Ceylo/Metal-Blend-Pipelines/assets/451334/189762f5-3cae-4812-ae8a-c5b7a6660ee6">

## Comments
So there is a lot of information packed in there:
- The first thing that you can notice is that the Core Image pipeline is a good candidate but not the best if your main concern is speed. And if you care about memory usage, it's the worst by far.
- The naive compute kernels behave pretty badly, and splitting the work into 4 independent branches didn't help.
- On the other hand, the monolithic compute kernel, which is supposed to do the same as Core Image, is actually 30% faster than Apple's implementation! Obviously the biggest downside is that everything is hardcoded and in a real app you'd want to be able to dynamically switch the blend operations or the number of layers involved. On this matter, stitchable functions should give identical speed, all while removing these two constraints, BUT removing the ability to debug such stitched compute kernels (at least as of Xcode 15.4).
- Now the much more interesting results in my opinion are with render commands. The solution with an encoder/layer is not so interesting: slower than Core Image and Metal Petal. On the other hand, the solution with a single encoder performs better than all other solutions, and it's still debuggable, and fully dynamic! 🎉 Its main downside is the slightly more tricky workflow in the fragment shader that forces you to do ping pong between two color attachments. A nice bonus is that the variation with tile memory is the one using the smallest amount of memory, along with the monolithic kernel. That one doesn't need any intermediate texture, while the render pipeline does, but since it's taking advantage of memoryless textures, it's not adding up on the app memory usage. And if we want to keep compatibility with Intel Macs, we just need to change one line of code and switch to a regular `.private` texture type.
- One more thing not visible in above results is that some pipelines will automatically skip computations for hidden fragments while some other won't. Basically all render pipelines that use a single encoder, or compute pipelines that use a single dispatch benefit from this. Same for Core Image, but NOT Metal Petal. This can be a nice speed boost "for free" if you render contents that's bigger than what you can display.

## Stability issues
Now there's one more thing that I noticed while profiling all this: some pipelines were giving very stable times while others weren't.
<div align="center">
  <img width="238" alt="Results table" src="https://github.com/Ceylo/Metal-Blend-Pipelines/assets/451334/b7246ae3-c4dd-43c6-91b1-dc63bd10e925">
  
  *An unstable pipeline*
</div>
If we check the behavior in Instruments with Metal System Trace, we can see the following:
<img width="953" alt="image" src="https://github.com/Ceylo/Metal-Blend-Pipelines/assets/451334/3c5f7d45-9d28-4410-a8c5-6001af4b1195">

Purple is window composition from macOS's WindowServer process. Orange is the work for a frame, and green the work for another frame. The key part is that we see the vertex function for two frames being executed at the very beginning. Once geometry for the orange frame is done, fragment function starts, then same for the green frame. This means that we start generating both frames almost at the same time, but the orange frame will be ready for display in a much shorter delay than the green frame. This can become even worse when Core Animation decides that it should add a 3rd frame in the pipeline to do triple-buffering.

So I decided to add synchronization so that the driver properly finishes one command buffer before starting the next one. And this gave a stable framerate for a minor cost in fps (or actually an improvement in the case of the monolithic compute kernel).
<img width="914" alt="image" src="https://github.com/Ceylo/Metal-Blend-Pipelines/assets/451334/623ba6df-256f-44a5-ab99-5a7ed8f4db06">
<div align="center">
  <img width="238" alt="image" src="https://github.com/Ceylo/Metal-Blend-Pipelines/assets/451334/bf072dca-16ac-4aff-8629-14e09d8206fc">
  
  *A stable pipeline*
</div>
So now you know what "sequential command buffers" means in above results.

## Resources

[WWDC21: Create image processing apps powered by Apple Silicon](https://developer.apple.com/wwdc21/10153)
