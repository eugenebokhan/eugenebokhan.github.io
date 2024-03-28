+++
title = "Introduction to Metal Compute: MTLSwift"
date = 2021-07-10

[taxonomies]
categories = ["Tech"]
tags = ["iOS", "Metal"]
+++

In this chapter, I will introduce you to another cool tool I use every day called [MTLSwift](https://github.com/s1ddok/mtlswift). What is MTLSwift? You might think it is something "Swifty" on the one hand and Metal-related on the other. And you will be right because this tool generates kernel encoders in Swift using Metal shaders. Same as [Alloy](https://github.com/s1ddok/Alloy), MTLSwift was also created by my colleague [Andrey Volodin](http://twitter.com/s1ddok) and co-maintained by me.

Before we dive into how this tool works, let's understand how the idea of encoder code generation was born.

If you look a the shaders and the encoding code, you will see a correlation between the arguments and names used in the shaders code and the values passed and encoded on the CPU side.

{{
    figure(
        src="correlation.png",
        alt="correlation",
        caption=""
    )
}}

This pattern repeats every time you create a kernel function and the encoder for it. Also, if you look at the kernel encoder, you will see that each one of them has the same structure: a pipeline state property, a constructor, taking a library as an argument, and the encoding logic, which takes textures, buffers, or small values as arguments. Such a simple structure of kernel encoder makes it easy to create a new encoder on the one hand. On the other hand, it contributes to the tendency when a developer starts copy-pasting the encoders and reusing them with modifications. And such behavior might become a source of bugs.

Given that, if you somehow extract the information from the kernel sources about the name of the kernel function, its arguments, and function constants, you may be able to create a generator of bug-free encoders. The first approach that may come to mind is to parse the sources and create a logic for understanding keyword operators, etc. However, creating a source code parser from scratch is not an easy task. What if we could use a Metal compiler to help us with that?

The Metal compiler itself is a modified version of Apple's [Clang](https://en.wikipedia.org/wiki/Clang). Clang is a C language family front end for LLVM. LLVM's front end is responsible for parsing the source code, breaking it up into pieces according to a grammatical structure, and checking it for errors. As a result, the front end outputs an Abstract Syntax Tree (AST). The latter is a structured representation, which can be used for different purposes such as creating a symbol table, performing type checking, and finally generating code.

For example, if we dump AST from a simplified version of our adjustments kernel:

```cpp
constant bool deviceSupportsNonuniformThreadgroups [[ function_constant(0) ]];

// MARK: - Adjustments

kernel void adjustments(texture2d<float, access::read> source [[ texture(0) ]],
                        texture2d<float, access::write> destination [[ texture(1) ]],
                        constant float& temperature [[ buffer(0) ]],
                        constant float& tint [[ buffer(1) ]],
                        uint2 position [[ thread_position_in_grid ]]) {

}
```

with the help of command:

```bash
xcrun -sdk iphoneos metal -Xclang -ast-dump -E -fno-color-diagnostics Shaders.metal
```

we will get such AST:

```cpp
|-UsingDirectiveDecl 0x7fa0d3dcc998 <Shaders.metal:3:1, col:17> col:17 Namespace 0x7fa0d4818710 'metal'
|-VarDecl 0x7fa0d3dcca18 <line:6:1, col:15> col:15 deviceSupportsNonuniformThreadgroups 'const constant bool'
| `-MetalFunctionConstantAttr 0x7fa0d3dcca78 <col:55, col:74>
|   `-IntegerLiteral 0x7fa0d3dcc9e8 <col:73> 'int' 0
`-FunctionDecl 0x7fa0d3dcd578 <line:10:1, line:16:1> line:10:13 adjustments 'void (texture2d<float, access::read>, texture2d<float, access::write>, const constant float &, const constant float &, uint2)'
  |-ParmVarDecl 0x7fa0d3dccda0 <col:25, col:56> col:56 source 'texture2d<float, access::read>':'metal::texture2d<float, metal::access::read, void>'
  | `-MetalTextureIndexAttr 0x7fa0d3dcce00 <col:66, col:75>
  |   `-IntegerLiteral 0x7fa0d3dccd38 <col:74> 'int' 0
  |-ParmVarDecl 0x7fa0d3dcd120 <line:11:25, col:57> col:57 destination 'texture2d<float, access::write>':'metal::texture2d<float, metal::access::write, void>'
  | `-MetalTextureIndexAttr 0x7fa0d3dcd180 <col:72, col:81>
  |   `-IntegerLiteral 0x7fa0d3dcd0b8 <col:80> 'int' 1
  |-ParmVarDecl 0x7fa0d3dcd230 <line:12:25, col:41> col:41 temperature 'const constant float &'
  | `-MetalBufferIndexAttr 0x7fa0d3dcd290 <col:56, col:64>
  |   `-IntegerLiteral 0x7fa0d3dcd1c8 <col:63> 'int' 0
  |-ParmVarDecl 0x7fa0d3dcd310 <line:13:25, col:41> col:41 tint 'const constant float &'
  | `-MetalBufferIndexAttr 0x7fa0d3dcd370 <col:49, col:57>
  |   `-IntegerLiteral 0x7fa0d3dcd2d8 <col:56> 'int' 1
  |-ParmVarDecl 0x7fa0d3dcd3c8 <line:14:25, col:31> col:31 position 'uint2':'unsigned int __attribute__((ext_vector_type(2)))'
  | `-MetalThreadPosGridAttr 0x7fa0d3dcd428 <col:43>
  |-CompoundStmt 0x7fa0d877da18 <col:71, line:16:1>
  `-MetalKernelAttr 0x7fa0d3dcd640 <line:10:1>
```

As we can see, AST has a node-based structure, which can be easily parsed. Internally MTLSwift calls the Metal compiler to output such AST, parses it, creates intermediate node-based representation, and extracts all needed information. To help MTLSwift get the info about how we want to dispatch the kernel and also add extra info about the kernel function arguments, some custom annotations were introduced. Let's take a look at them.

### Customising code generation

Every custom annotation starts with `mtlswift:`. The program uses this declaration prefix to identify the start of a declaration. It must be written in a docstring way right before the kernel.

```cpp
/// mtlswift: ...
kernel void exampleKernel(...
```

- `dispatch:`

    A dispatch type to use. All dispatch types have to be followed by either a constant amount of threads via literals (like `X, Y, Z`), specifying a target texture to cover via `over:` argument, or stating that amount of threads will be provided by the user by using `provided`. You can see all of the examples in each section, but you can choose the combination yourself.

  - `even`

    Dispatch threadgroups of a uniform threadgroup size. `Width`, `height`, and `depth` describe the grid size.

  - `exact`

    Dispatch threads with threadgroups of non-uniform size.

  - `optimal(function_constant_index)`

    Uses `exact` type if GPU supports non-uniform threadgroup size and `over` if it doesn't. This declaration requires a boolean function constant index to be passed to decide what dispatch type to use.

  - `none`

    The dispatch type is used by default. In this case, the user has to dispatch the kernel manually after calling `encode` method

- `threadgroupSize:`

    Specify the threadgroup size.

  - `X, Y, Z`

    Allows to specify constant X, Y and Z dimensions for threadgroup size.

  - `max`

    This parameter sets the pipeline state's [`max2dThreadgroupSize`](https://github.com/s1ddok/Alloy/blob/b82aa3fde347a81eef9551be7ffc28eec2b93bca/Alloy/MTLComputePipelineState%2BThreads.swift#L24).

  - `executionWidth`

    This parameter sets the pipeline state's [`executionWidthThreadgroupSize`](https://github.com/s1ddok/Alloy/blob/b82aa3fde347a81eef9551be7ffc28eec2b93bca/Alloy/MTLComputePipelineState%2BThreads.swift#L12).

  - `provided`

    In this case, the user has to pass the threadgroup size and an argument to `encode(...` function.

- `swiftParameterType:`

    The type of the buffers passed to the kernel.

- `swiftParameterName:`

    The name of the buffers passed to the kernel.

- `swiftName:`

    Encoder's name in generated Swift code. Must be followed by a valid Swift identifier.

- `accessLevel:`

    Specifies the access visibility of the encoder. Must be followed by either `public`, `open`, `internal`, `private` or `fileprivate`. `internal` is the default.

### Adjustments

Ok, let's update our shaders code to support `MTLSwift`. First, in `Shaders.metal` after

```cpp
using namespace metal;
```

add the following code:

```cpp
namespace mtlswift {}
```

This new line is an entry point of the MTLSwift AST parser.
Now let's add custom annotations before the kernel:

```cpp
/// mtlswift:dispatch:optimal(0):over:destination
```

This annotation tells that the encoder will dispatch a grid of the same dimension as the destination texture with non-uniform threadgroups branching function constant set at 0.

Next, let's declare the types of tint and temperature values passed to the encoder:

```cpp
/// mtlswift:swiftParameterType:temperature:Float32
/// mtlswift:swiftParameterType:tint:Float32
```

The final version of the shader file should look like this:

```cpp
#include <metal_stdlib>
#include "ColorConversion.h"
using namespace metal;
namespace mtlswift {}

constant bool deviceSupportsNonuniformThreadgroups [[ function_constant(0) ]];

// MARK: - Adjustments

/// mtlswift:dispatch:optimal(0):over:destination
/// mtlswift:swiftParameterType:temperature:Float32
/// mtlswift:swiftParameterType:tint:Float32
kernel void adjustments(texture2d<float, access::read> source [[ texture(0) ]],
                        texture2d<float, access::write> destination [[ texture(1) ]],
                        constant float& temperature [[ buffer(0) ]],
                        constant float& tint [[ buffer(1) ]],
                        uint2 position [[thread_position_in_grid]]) {
    const auto textureSize = ushort2(destination.get_width(),
                                     destination.get_height());
    if (!deviceSupportsNonuniformThreadgroups) {
        if (position.x >= textureSize.x || position.y >= textureSize.y) {
            return;
        }
    }
    const auto sourceValue = source.read(position);
    auto labValue = denormalizeLab(rgb2lab(sourceValue.rgb));
    labValue.b += temperature * 10.0f;
    labValue.g += tint * 10.0f;
    labValue = clipLab(labValue);
    labValue = normalizeLab(labValue);
    const auto resultValue = float4(lab2rgb(labValue), sourceValue.a);
    destination.write(resultValue, position);
}
```

Now let's install MTLSwift.

```bash
git clone https://github.com/s1ddok/mtlswift.git
cd mtlswift
sudo make install
```

To generate the encoder for the kernel in the .metal file, you need to call MTLSwift's `generate` command:

```bash
mtlswift generate part-6/Image\ Editor\ Demo/Shaders/Shaders.metal
```

As a result, you will get `Shaders.metal.swift` file next to the shaders:

```swift
// This file is autogenerated, do not edit it
import Alloy
internal class Adjustments {
  internal let deviceSupportsNonuniformThreadgroups: Bool
  internal let pipelineState: MTLComputePipelineState
  internal init(library: MTLLibrary) throws {
    let constantValues = MTLFunctionConstantValues()
    self.deviceSupportsNonuniformThreadgroups = library.device.supports(feature: .nonUniformThreadgroups)
    constantValues.set(self.deviceSupportsNonuniformThreadgroups, at: 0)
    self.pipelineState = try library.computePipelineState(function: "adjustments", constants: constantValues)
  }
  internal func callAsFunction(source: MTLTexture, destination: MTLTexture, temperature: Float32, tint: Float32, in commandBuffer: MTLCommandBuffer) {
    self.encode(source: source, destination: destination, temperature: temperature, tint: tint, in: commandBuffer)
  }
  internal func callAsFunction(source: MTLTexture, destination: MTLTexture, temperature: Float32, tint: Float32, using encoder: MTLComputeCommandEncoder) {
    self.encode(source: source, destination: destination, temperature: temperature, tint: tint, using: encoder)
  }
  internal func encode(source: MTLTexture, destination: MTLTexture, temperature: Float32, tint: Float32, in commandBuffer: MTLCommandBuffer) {
    commandBuffer.compute { encoder in
      encoder.label = "Adjustments"
      self.encode(source: source, destination: destination, temperature: temperature, tint: tint, using: encoder)
    }
  }
  internal func encode(source: MTLTexture, destination: MTLTexture, temperature: Float32, tint: Float32, using encoder: MTLComputeCommandEncoder) {
    let _threadgroupSize = self.pipelineState.max2dThreadgroupSize
    encoder.setTexture(source, index: 0)
    encoder.setTexture(destination, index: 1)
    encoder.setValue(temperature, at: 0)
    encoder.setValue(tint, at: 1)
    if self.deviceSupportsNonuniformThreadgroups { encoder.dispatch2d(state: self.pipelineState, exactly: destination.size, threadgroupSize: _threadgroupSize) } else { encoder.dispatch2d(state: self.pipelineState, covering: destination.size, threadgroupSize: _threadgroupSize) }
  }
}
```

Import this file to the Xcode project and remove `Adjustments.swift` file.

As the autogenerated `Adjustments` class doesn't encapsulate temperature and tint properties, we need to move them to `ViewController.swift`:

```swift
private var temperature = Float.zero
private var tint = Float.zero
```

Modify the settings in the `commonInit` function:

```swift
self.settings.settings = [
    FloatSetting(name: "Temperature",
                 defaultValue: .zero,
                 min: -1,
                 max: 1) {
    self.temperature = $0
    self.redraw()
},
    FloatSetting(name: "Tint",
                 defaultValue: .zero,
                 min: -1,
                 max: 1) {
    self.tint = $0
    self.redraw()
}
]
```

Inside the redraw function replace the dispatching code:

```swift
private func redraw() {
    guard let source = self.texturePair?.source,
          let destination = self.texturePair?.destination
    else { return }
    DispatchQueue.main.async {
        try? self.context.schedule { commandBuffer in
            self.adjustments(source: source,
                             destination: destination,
                             temperature: self.temperature,
                             tint: self.tint,
                             in: commandBuffer)
            self.textureView.draw(in: commandBuffer)
        }
    }
}
```

That's it! Now you can compile and run the project. From this point, each time you modify the shaders, you don't need to worry about the encoders at all. By calling MTLSwift, you automatically get 50% of the shaders-related job done, which means less code to maintain with fewer bugs to show up.

The final code can be found [here](https://github.com/eugenebokhan/introduction-to-metal-compute/tree/main/part-6).

Thank you for reading, see you next time 👋.
