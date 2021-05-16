---
layout: post
title:  "Introduction to Metal Compute: Part 3"
date:   2021-05-10 15:01:35 +0300
image:  '/images/introduction-to-metal-compute-part-three/icon.png'
tags:
---

## CPU Side: Encoder

Now it's time write the CPU side of the metal pipeline. First let's take a quick brief on how GPU work scheduling is organised.

To get the GPU to perform work on your behalf, you need to send commands to it. There are three types of commands: [`render`](https://developer.apple.com/documentation/metal/mtlrendercommandencoder), [`compute`](https://developer.apple.com/documentation/metal/mtlcomputecommandencoder) and [`blit`](https://developer.apple.com/documentation/metal/mtlblitcommandencoder). Compute command is what we need to schedule our adjustments shader for execution. 

The objects that you operate while creating a command for a GPU are:

- [`device`](https://developer.apple.com/documentation/metal/mtldevice) - software interface to a GPU. Device is able to create command queues, shader libraries, pipeline states and allocate resources (heaps, buffers and textures). Created once by calling [`MTLCreateSystemDefaultDevice()`](https://developer.apple.com/documentation/metal/1433401-mtlcreatesystemdefaultdevice).
- [`library`](https://developer.apple.com/documentation/metal/mtllibrary) - an object that contains compiled shaders. Created once by `device` by calling [`makeDefaultLibrary()`](https://developer.apple.com/documentation/metal/mtldevice/1433380-makedefaultlibrary).
- [`function`](https://developer.apple.com/documentation/metal/mtlfunction) - an object that specifies which shader function a Metal pipeline calls when the GPU executes commands that specify that pipeline. Created once by `library` by calling [`makeFunction(name:)`](https://developer.apple.com/documentation/metal/mtllibrary/1515524-makefunction).
- [`pipeline state`](https://developer.apple.com/documentation/metal/mtlcomputepipelinestate) - an object used to refer to a compiled function. Created once by `device` by calling [`makeComputePipelineState(function:)`](https://developer.apple.com/documentation/metal/mtldevice/1433395-makecomputepipelinestate).
- [`command queue`](https://developer.apple.com/documentation/metal/mtlcommandqueue) - an object that queues an ordered list of command buffers for a device to execute. Created once by device by calling [`makeCommandQueue()`](https://developer.apple.com/documentation/metal/mtldevice/1433388-makecommandqueue).
- [`command buffer`](https://developer.apple.com/documentation/metal/mtlcommandbuffer) - a lightweight container that stores encoded commands for the GPU to execute. Created by `command queue` on each command dispatch by calling [`makeCommandBuffer()`](https://developer.apple.com/documentation/metal/mtlcommandqueue/1508686-makecommandbuffer).
- [`command encoder`](https://developer.apple.com/documentation/metal/mtlcommandencoder) - a lightweight object used to encode commands in command buffer. Created by `command buffer` on each command dispatch by calling [`makeComputeCommandEncoder()`](https://developer.apple.com/documentation/metal/mtlcommandbuffer/1443044-makecomputecommandencoder).

The hierarchy of creation of the objects is the depicted below:

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-three/metal-objects-hierarchy.png"
alt="demo app" 
width="600"
/>
</p>

Now let's make an empty swift file `Adjustments.swift`.

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-three/new-adjustments-file.png"
alt="demo app" 
width="800"
/>
</p>

### Adjustments

We are going to create an `Adjustments` class which will be responsible for encoding the work to GPU and passing all necessary data to it: temperature and tint in our case. Following Metal's paradigm of precompilation of the instructions once and quick reuse them in runtime, `Adjustments` will store the pipeline state as its property.

{% highlight swift %}
import Metal

final class Adjustments {

}
{% endhighlight %}

Create `temperature` and `tint` properties. These values will be modified by the UI and then sent to the kernel while encoding.

{% highlight swift %}
var temperature: Float = .zero
var tint: Float = .zero
{% endhighlight %}

Create the dispatch flag and the pipeline state. These values need to be initialised once and stored to use them while encoding.

{% highlight swift %}
private var deviceSupportsNonuniformThreadgroups: Bool
private let pipelineState: MTLComputePipelineState
{% endhighlight %}

The constructor of `Adjustments` class takes a metal library as an argument. The library is used further to initialise a function for the pipeline state. As one library can contain multiple functions, it's a good practice to initialise library once and then reuse it. So we're going to create and store the library outside of the class.

{% highlight swift %}
init(library: MTLLibrary) throws {

}
{% endhighlight %}

From this point we are going to fill the constructor following step by step instructions.

### Constructor

Different iPhones have different hardware (including GPU) that supports the different set of features. In order to initialise `deviceSupportsNonuniformThreadgroups` property correctly we need to look find such feature in the [Metal Feature Set Table](https://developer.apple.com/metal/Metal-Feature-Set-Tables.pdf) and find a corresponding feature set that describes the type of hardware that supports it.

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-three/feature-table.png"
alt="demo app" 
width="800"
/>
</p>

{% highlight swift %}
self.deviceSupportsNonuniformThreadgroups = library.device.supportsFeatureSet(.iOS_GPUFamily4_v1)
{% endhighlight %}

Initialise function constants object and set the `deviceSupportsNonuniformThreadgroups` value to it. The index is set to 0, the same as it was declared in the shaders.

{% highlight swift %}
let constantValues = MTLFunctionConstantValues()
constantValues.setConstantValue(&self.deviceSupportsNonuniformThreadgroups,
                                type: .bool,
                                index: 0)
{% endhighlight %}

Create a function from a library with and previously initialised FC. The name of the function is the same as in the shaders.

{% highlight swift %}
let function = try library.makeFunction(name: "adjustments",
                                        constantValues: constantValues)
{% endhighlight %}

The final step is the pipeline state creation. At this point the shaders will be compiled into GPU instructions and the passed FC will be used to determine if boundary check will be among them.

{% highlight swift %}
self.pipelineState = try library.device.makeComputePipelineState(function: function)
{% endhighlight %}

The result should look like this:

{% highlight swift %}
import Metal

final class Adjustments {

    var temperature: Float = .zero
    var tint: Float = .zero
    private var deviceSupportsNonuniformThreadgroups: Bool
    private let pipelineState: MTLComputePipelineState
    
    init(library: MTLLibrary) throws {
        self.deviceSupportsNonuniformThreadgroups = library.device.supportsFeatureSet(.iOS_GPUFamily4_v1)
        let constantValues = MTLFunctionConstantValues()
        constantValues.setConstantValue(&self.deviceSupportsNonuniformThreadgroups,
                                        type: .bool,
                                        index: 0)
        let function = try library.makeFunction(name: "adjustments",
                                                constantValues: constantValues)
        self.pipelineState = try library.device.makeComputePipelineState(function: function)
    }
    
}
{% endhighlight %}

### Encoding Function

Next, we're a going to write the encoding of the kernel. The main thing what we need here to do is use command buffer's encoder to encode all necessary resources and instructions to GPU.

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-three/command-buffer.png"
alt="demo app" 
width="500"
/>
</p>

Below the class constructor, add the encoding function.

{% highlight swift %}
func encode(source: MTLTexture,
            destination: MTLTexture,
            in commandBuffer: MTLCommandBuffer) {

}
{% endhighlight %}

Now let's fill it. Create a `command encoder`. This lightweight object is used to encode everything to a `command buffer`.

{% highlight swift %}
guard let encoder = commandBuffer.makeComputeCommandEncoder()
else { return }
{% endhighlight %}

Set `source` and `destination` textures at the same indices we used in the shaders.

{% highlight swift %}
encoder.setTexture(source,
                   index: 0)
encoder.setTexture(destination,
                   index: 1)
{% endhighlight %}

Set `tint` and `temperature` values. Given that the data that we send to GPU is just two float values, which is not much in size, we use recommended in such cases [`setBytes`](https://developer.apple.com/documentation/metal/mtlcomputecommandencoder/1443159-setbytes) function. If the data is large, we'd create an [`MTLBuffer`](https://developer.apple.com/documentation/metal/mtlbuffer) for it and used [`setBuffer`](https://developer.apple.com/documentation/metal/mtlcomputecommandencoder/1443126-setbuffer) instead. The indices are the same as in the kernel's arguments.

{% highlight swift %}
encoder.setBytes(&self.temperature,
                 length: MemoryLayout<Float>.stride,
                 index: 0)
encoder.setBytes(&self.tint,
                 length: MemoryLayout<Float>.stride,
                 index: 1)
{% endhighlight %}

Calculate the size of grid and the threadgroups. The grid size should be the same as the texture's so each thread could work on its own pixel. Speaking about the threadgroup size, we need it to be as much as possible in order to maximise the work parallelisation. The calculation of threadgroup size is based on two properties of pipeline state: [`maxTotalThreadsPerThreadgroup`](https://developer.apple.com/documentation/metal/mtlcomputepipelinestate/1414927-maxtotalthreadsperthreadgroup) and [`threadExecutionWidth`](https://developer.apple.com/documentation/metal/mtlcomputepipelinestate/1414911-threadexecutionwidth). The first defines the maximum number of threads that can be in a single threadgroup and the second is equal to width of SIMD group and defines the number of threads to execute in parallel on the GPU.

{% highlight swift %}
let gridSize = MTLSize(width: source.width,
                       height: source.height,
                       depth: 1)
let threadGroupWidth = self.pipelineState.threadExecutionWidth
let threadGroupHeight = self.pipelineState.maxTotalThreadsPerThreadgroup / threadGroupWidth
let threadGroupSize = MTLSize(width: threadGroupWidth,
                              height: threadGroupHeight,
                              depth: 1)
{% endhighlight %}

Set the pipeline state which contains precompiled instructions of our adjustments kernel.

{% highlight swift %}
encoder.setComputePipelineState(self.pipelineState)
{% endhighlight %}

If the device supports non-uniform threadgroups, we allow Metal to calculate the number of them and to generate smaller threadgroups along the edges of the grid. If the device doesn't support this feature, we calculate the number of threadgroups by hand to overlap the size of the texture. 

{% highlight swift %}
if self.deviceSupportsNonuniformThreadgroups {
    encoder.dispatchThreads(gridSize,
                            threadsPerThreadgroup: threadGroupSize)
} else {
    let threadGroupCount = MTLSize(width: (gridSize.width + threadGroupSize.width - 1) / threadGroupSize.width,
                                   height: (gridSize.height + threadGroupSize.height - 1) / threadGroupSize.height,
                                   depth: 1)
    encoder.dispatchThreadgroups(threadGroupCount,
                                 threadsPerThreadgroup: threadGroupSize)
}
{% endhighlight %}

After all encoding is done, we call [`endEncoding()`](https://developer.apple.com/documentation/metal/mtlcommandencoder/1458038-endencoding). Without calling of this function the command buffer won't know that it is ready to dispatch the commands to GPU.

{% highlight swift %}
encoder.endEncoding()
{% endhighlight %}

Here's the final encoding function:

{% highlight swift %}
func encode(source: MTLTexture,
            destination: MTLTexture,
            in commandBuffer: MTLCommandBuffer) {
    guard let encoder = commandBuffer.makeComputeCommandEncoder()
    else { return }

    encoder.setTexture(source,
                       index: 0)
    encoder.setTexture(destination,
                       index: 1)

    encoder.setBytes(&self.temperature,
                     length: MemoryLayout<Float>.stride,
                     index: 0)
    encoder.setBytes(&self.tint,
                     length: MemoryLayout<Float>.stride,
                     index: 1)

    let gridSize = MTLSize(width: source.width,
                           height: source.height,
                           depth: 1)
    let threadGroupWidth = self.pipelineState.threadExecutionWidth
    let threadGroupHeight = self.pipelineState.maxTotalThreadsPerThreadgroup / threadGroupWidth
    let threadGroupSize = MTLSize(width: threadGroupWidth,
                                  height: threadGroupHeight,
                                  depth: 1)

    encoder.setComputePipelineState(self.pipelineState)
    
    if self.deviceSupportsNonuniformThreadgroups {
        encoder.dispatchThreads(gridSize,
                                threadsPerThreadgroup: threadGroupSize)
    } else {
        let threadGroupCount = MTLSize(width: (gridSize.width + threadGroupSize.width - 1) / threadGroupSize.width,
                                       height: (gridSize.height + threadGroupSize.height - 1) / threadGroupSize.height,
                                       depth: 1)
        encoder.dispatchThreadgroups(threadGroupCount,
                                     threadsPerThreadgroup: threadGroupSize)
    }
    
    encoder.endEncoding()
}
{% endhighlight %}

Excellent! Now we have a compute kernel and the corresponding encoder for it. In the next part we are going to write [`UIImage`](https://developer.apple.com/documentation/uikit/uiimage) to [`MTLTexture`](https://developer.apple.com/documentation/metal/mtltexture) conversion to pass the textures to the encoder, create a command queue and dispatch the kernel 👍.