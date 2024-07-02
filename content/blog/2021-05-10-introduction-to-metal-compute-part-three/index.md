+++
title = "Kernel Encoder"
date = 2021-05-10

[taxonomies]
categories = ["Introduction to Metal Compute"]
tags = ["iOS", "Metal"]
+++

## CPU Side: Encoder

Now it's time to write the CPU side of the metal pipeline. First, let's take a quick brief on how the GPU work scheduling is organized.

To get the GPU to perform work on your behalf, you need to send commands to it. There are three types of commands: [`render`](https://developer.apple.com/documentation/metal/mtlrendercommandencoder), [`compute`](https://developer.apple.com/documentation/metal/mtlcomputecommandencoder) and [`blit`](https://developer.apple.com/documentation/metal/mtlblitcommandencoder). Compute command is what we need to schedule our adjustments shader for execution.

The objects that you operate while creating a command for a GPU are:

- [`device`](https://developer.apple.com/documentation/metal/mtldevice) - software interface to a GPU. The device is able to create command queues, shader libraries, and pipeline states and allocate resources (heaps, buffers and textures). Created once by calling [`MTLCreateSystemDefaultDevice()`](https://developer.apple.com/documentation/metal/1433401-mtlcreatesystemdefaultdevice).
- [`library`](https://developer.apple.com/documentation/metal/mtllibrary) - an object that contains compiled shaders. Created once by `device` by calling [`makeDefaultLibrary()`](https://developer.apple.com/documentation/metal/mtldevice/1433380-makedefaultlibrary).
- [`function`](https://developer.apple.com/documentation/metal/mtlfunction) - an object that specifies which shader function a Metal pipeline calls when the GPU executes commands that specify that pipeline. Created once by the `library` by calling the [`makeFunction(name:)`](https://developer.apple.com/documentation/metal/mtllibrary/1515524-makefunction).
- [`pipeline state`](https://developer.apple.com/documentation/metal/mtlcomputepipelinestate) - an object used to refer to a compiled function. Created once by `device` by calling [`makeComputePipelineState(function:)`](https://developer.apple.com/documentation/metal/mtldevice/1433395-makecomputepipelinestate).
- [`command queue`](https://developer.apple.com/documentation/metal/mtlcommandqueue) - an object that queues an ordered list of command buffers for a device to execute. Created once by device by calling [`makeCommandQueue()`](https://developer.apple.com/documentation/metal/mtldevice/1433388-makecommandqueue).
- [`command buffer`](https://developer.apple.com/documentation/metal/mtlcommandbuffer) - a lightweight container that stores encoded commands for the GPU to execute. Created by `command queue` on each command dispatch by calling [`makeCommandBuffer()`](https://developer.apple.com/documentation/metal/mtlcommandqueue/1508686-makecommandbuffer).
- [`command encoder`](https://developer.apple.com/documentation/metal/mtlcommandencoder) - a lightweight object used to encode commands in a command buffer. Created by `command buffer` on each command dispatch by calling [`makeComputeCommandEncoder()`](https://developer.apple.com/documentation/metal/mtlcommandbuffer/1443044-makecomputecommandencoder).

The hierarchy of creation of the objects is depicted below:

{{
    figure(
        src="metal-objects-hierarchy.png",
        alt="metal-objects-hierarchy",
        caption=""
    )
}}

Now let's make an empty swift file `Adjustments.swift`.

{{
    figure(
        src="new-adjustments-file.png",
        alt="new-adjustments-file",
        caption=""
    )
}}

### Adjustments

We are going to create an `Adjustments` class which will be responsible for encoding the work to GPU and passing all necessary data to it: temperature and tint in our case. Following Metal's paradigm of precompilation of the instructions once and quickly reusing them in runtime, `Adjustments` will store the pipeline state as its property.

```swift
import Metal

final class Adjustments {

}
```

Create `temperature` and `tint` properties. These values will be modified by the UI and then sent to the kernel while encoding.

```swift
var temperature: Float = .zero
var tint: Float = .zero
```

Create the dispatch flag and the pipeline state. These values need to be initialized once and stored to use them while encoding.

```swift
private var deviceSupportsNonuniformThreadgroups: Bool
private let pipelineState: MTLComputePipelineState
```

The constructor of `Adjustments` class takes a metal library as an argument. The library is used further to initialize a function for the pipeline state. As one library can contain multiple functions, it's a good practice to initialize the library once and then reuse it. So we're going to create and store the library outside of the class.

```swift
init(library: MTLLibrary) throws {

}
```

From this point, we are going to fill the constructor following step-by-step instructions.

### Constructor

Different iPhones have different hardware (including GPU) that supports different sets of features. To initialise `deviceSupportsNonuniformThreadgroups`` property correctly we need to look find such a feature in the [Metal Feature Set Table](https://developer.apple.com/metal/Metal-Feature-Set-Tables.pdf) and find a corresponding feature set that describes the type of hardware that supports it.

{{
    figure(
        src="feature-table.png",
        alt="feature-table",
        caption=""
    )
}}

```swift
self.deviceSupportsNonuniformThreadgroups = library.device.supportsFeatureSet(.iOS_GPUFamily4_v1)
```

Initialise function constants object and set the `deviceSupportsNonuniformThreadgroups` value to it. The index is set to 0, the same as it was declared in the shaders.

```swift
let constantValues = MTLFunctionConstantValues()
constantValues.setConstantValue(&self.deviceSupportsNonuniformThreadgroups,
                                type: .bool,
                                index: 0)
```

Create a function from a library with and previously initialised FC. The name of the function is the same as in the shaders.

```swift
let function = try library.makeFunction(name: "adjustments",
                                        constantValues: constantValues)
```

The final step is the pipeline state creation. At this point, the shaders will be compiled into GPU instructions and the passed FC will be used to determine if a boundary check will be among them.

```swift
self.pipelineState = try library.device.makeComputePipelineState(function: function)
```

The result should look like this:

```swift
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
```

### Encoding Function

Next, we're going to write the encoding of the kernel. The main thing that we need here to do is use the command buffer's encoder to encode all necessary resources and instructions to the GPU.

{{
    figure(
        src="command-buffer.png",
        alt="command-buffer",
        caption=""
    )
}}

Below the class constructor, add the encoding function.

```swift
func encode(source: MTLTexture,
            destination: MTLTexture,
            in commandBuffer: MTLCommandBuffer) {

}
```

Now let's fill it. Create a `command encoder`. This lightweight object is used to encode everything in a `command` buffer`.

```swift
guard let encoder = commandBuffer.makeComputeCommandEncoder()
else { return }
```

Set `source` and `destination` textures at the same indices we used in the shaders.

```swift
encoder.setTexture(source,
                   index: 0)
encoder.setTexture(destination,
                   index: 1)
```

Set `tint` and `temperature` values. Given that the data that we send to GPU is just two float values, which is not much in size, we use recommended in such cases [`setBytes`](https://developer.apple.com/documentation/metal/mtlcomputecommandencoder/1443159-setbytes) function. If the data is large, we'd create an [`MTLBuffer`](https://developer.apple.com/documentation/metal/mtlbuffer) for it and used [`setBuffer`](https://developer.apple.com/documentation/metal/mtlcomputecommandencoder/1443126-setbuffer) instead. The indices are the same as in the kernel's arguments.

```swift
encoder.setBytes(&self.temperature,
                 length: MemoryLayout<Float>.stride,
                 index: 0)
encoder.setBytes(&self.tint,
                 length: MemoryLayout<Float>.stride,
                 index: 1)
```

Calculate the size of the grid and the threadgroups. The grid size should be the same as the texture's so each thread can work on its pixel. Speaking about the threadgroup size, we need it to be as much as possible to maximize the work parallelization. The calculation of threadgroup size is based on two properties of the pipeline state: [`maxTotalThreadsPerThreadgroup`](https://developer.apple.com/documentation/metal/mtlcomputepipelinestate/1414927-maxtotalthreadsperthreadgroup) and [`threadExecutionWidth`](https://developer.apple.com/documentation/metal/mtlcomputepipelinestate/1414911-threadexecutionwidth). The first defines the maximum number of threads that can be in a single threadgroup and the second is equal to the width of the SIMD group and defines the number of threads to execute in parallel on the GPU.

```swift
let gridSize = MTLSize(width: source.width,
                       height: source.height,
                       depth: 1)
let threadGroupWidth = self.pipelineState.threadExecutionWidth
let threadGroupHeight = self.pipelineState.maxTotalThreadsPerThreadgroup / threadGroupWidth
let threadGroupSize = MTLSize(width: threadGroupWidth,
                              height: threadGroupHeight,
                              depth: 1)
```

Set the pipeline state which contains precompiled instructions of our adjustments kernel.

```swift
encoder.setComputePipelineState(self.pipelineState)
```

If the device supports non-uniform threadgroups, we allow Metal to calculate the number of them and generate smaller threadgroups along the edges of the grid. If the device doesn't support this feature, we calculate the number of threadgroups by hand to overlap the size of the texture.

```swift
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
```

After all encoding is done, we call [`endEncoding()`](https://developer.apple.com/documentation/metal/mtlcommandencoder/1458038-endencoding). Without calling this function the command buffer won't know that it is ready to dispatch the commands to the GPU.

```swift
encoder.endEncoding()
```

Here's the final encoding function:

```swift
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
```

Excellent! Now we have a compute kernel and the corresponding encoder for it. In the [next part](@/blog/2021-05-14-introduction-to-metal-compute-part-four/index.md), we are going to write [`UIImage`](https://developer.apple.com/documentation/uikit/uiimage) to [`MTLTexture`](https://developer.apple.com/documentation/metal/mtltexture) conversion to pass the textures to the encoder, create a command queue and dispatch the kernel 👍.
