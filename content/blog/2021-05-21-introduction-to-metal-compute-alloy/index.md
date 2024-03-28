+++
title = "Introduction to Metal Compute: Alloy"
date = 2021-05-21

[taxonomies]
categories = ["Tech"]
tags = ["iOS", "Metal"]
+++

Hello everyone and welcome to the fifth chapter of Introduction to Metal Compute! We did a lot of things in the previous parts. We created a simple image editing app that can open, preview, adjust and export images. To do that, we wrote an image-editing Metal shader kernel, created an encoder for it, learned, how to convert images to textures and passed the data to the GPU while dispatching the commands to it. This article aims to encourage you to use a more "Swifty" way of writing metal-related code. Also, we will migrate from `UIImageView` to `CAMetalLayer` for previewing the result.

## Alloy

Vanilla Metal provides access to work with the device's GPU. It practically does not add any abstraction and allows you to work in the same paradigm in which the hardware works. Being a low-level API, on one hand, Metal provides an ability to have fine-grained control over the hardware, and on the other hand, it introduces a little bit of complexity and redundancy in some cases. While writing metal pipelines, we operate such concepts as devices, command queues, command buffers, command encoders, libraries, functions and more. Some of these objects are created once and can be reused, while others need to be initialized on every kernel dispatch. Some of them need to be initialized with their corresponding descriptors and some are not. In some places, the API is throwable, and in others returns optionals. In general, it feels like Metal was written for Objective-C users without any extra adaptation for Swift.

With all these thoughts in mind, [Alloy](https://github.com/s1ddok/Alloy/) was born. This framework's purpose is to simplify Metal development on Swift and make the code cleaner and consistent without changing the main paradigm of low-level control over how things work. It provides a nano-tiny layer over vanilla Metal API, that hides the majority of redundant explicity in the Metal code, while not limiting flexibility a bit. Originally Alloy was created in 2018 by my colleague [Andrey Volodin](https://twitter.com/s1ddok) and what concerns me, I was one of the main contributors to it for a couple of years.

To make code more consistent, the Alloys API is designed to be throwable in those places where vanilla Metal is either throwable or returns optionals. A lot of extensions were added to the device, texture, command queue and other classes to reduce the number of repeating boilerplate code.

### Device

The device has been upgraded and by using it you can:

- allocate a heap without a descriptor:

```swift
let heap = try device.heap(size: 512,
                           storageMode: .shared,
                           cpuCacheMode: .defaultCache)
```

- create a texture with a few lines of code:

```swift
let texture = try device.texture(width: 512,
                                 height: 512,
                                 pixelFormat: .bgra8Unorm,
                                 usage: [.shaderRead, .shaderWrite])
```

- allocate a buffer with a value:

```swift
let someValue = SIMD4<Float>(repeating: 1)
let buffer = try device.buffer(with: someValue,
                               options: .storageModeShared)
```

... and more!

### Command Queue

We extended the command queue with two convenience functions that allow you to:

- dispatch a command buffer in an async manner:

```swift
commandQueue.schedule { commandBuffer in
    // encoding logic
}
```

- dispatch a command buffer synchronously:

```swift
commandQueue.scheduleAndWait { commandBuffer in
    // encoding logic
}
```

### Command Buffer

Now encoding the commands to the command buffer can be done by calling just one function. Also, you don't need to worry about committing to the work. You can easily encode:

- a compute command:

```swift
commandBuffer.compute { computeCommandEncoder in
    // compute command encoding logic
}
```

- a render command:

```swift
commandBuffer.render(descriptor: MTLRenderPassDescriptor) { renderCommandEncoder in
    // render command encoding logic
}
```

- a blit command:

```swift
commandBuffer.blit { blitCommandEncoder
    // blit command encoding logic
}
```

### Compute Command Encoder

Remember how you passed data to shaders via a compute command encoder? If you needed to pass any value, you needed to calculate its size in bytes and pass a reference to the value. Now you can just call:

```swift
let someValue = SIMD4<Float>(repeating: 1)
encoder.setValue(someValue, at: 0)
```

or if you need to pass an array:

```swift
let someArray = [Float](repeating: 1, count: 256)
encoder.setValue(someArray, at: 0)
```

Also, you can set the textures just by calling:

```swift
encoder.setTextures(textureOne, textureTwo)
```

and buffers:

```swift
encoder.setBuffers(bufferOne, bufferTwo)
```

One of the key things is that now you don't need to write threadgrop size computations by hand and the code can be reduced just to:

```swift
if self.deviceSupportsNonuniformThreadgroups {
    encoder.dispatch2d(state: pipelineState,
                       exactly: size)
} else {
    encoder.dispatch2d(state: pipelineState,
                       covering: size)
}
```

### Texture

What concerns textures, now you can create images from them by calling:

```swift
let image = try texture.figure(colorSpace: .displayP3Space)
```

and pixel buffers:

```swift
let pixelBuffer = texture.pixelBuffer
```

Now it is easy to get `size`, `region` and a `descriptor` of a texture as well as to create its empty copy:

```swift
let textureTwo = try textureOne.matchingTexture(usage: [.shaderRead, .shaderWrite],
                                                storage: .shared)
```

### Context

The only new concept that Alloy introduces is `MTLContext`. The context is an object that is designed to be injected across the app. Internally, the context holds references to such objects that remain the same over the whole metal pipeline lifecycle (device, command queue, library cache and texture loader) and provides a convenience API to maintain it. With the help of context, you can:

- create a texture form `CGImage`:

```swift
let texture = try self.context.texture(from: CGImage,
                                       srgb: Bool?,
                                       usage: MTLTextureUsage,
                                       generateMipmaps: Bool)
```

- create a shaders library for a given bundle:

```swift
let library = try self.context.library(for: Bundle)
```

- do everything that a device and command queue can.

Also, it is important to notice that this framework provides a set of handwritten utility kernels that are commonly used in image processing:

- BitonicSort;
- LookUpTable;
- MaskGuidedBlur;
- Normalisation;
- RGBAToYCbCr;
- YCbCrToRGBA;
- TextureAffineCrop;
- TextureCopy;
- TextureMask;
- TextureMax;
- TextureMean;
- TextureMin;
- TextureMultiplyAdd;
- TextureResize;
- TextureWeightedMix;
  
and more!

And yet we have covered just a little part of all the extensions that Alloy has. Alloy is a production-ready tool and it is used in the development of all [Prisma](https://prisma-ai.com)'s apps. I highly recommend you give this framework a try, and I am sure, you won't return to vanilla Metal anymore 🙂.

## Demo App

Let's migrate our existing codebase to Alloy and see it in action. First, add it as a dependency to the project.

{{
    figure(
        src="add-alloy-package.png",
        alt="add-alloy-package",
        caption=""
    )
}}

### Adjustments

Navigate to `Adjustments.swift` and replace `import Metal` with `import Alloy`.

Now let's modify the class constructor. The first thing is `deviceSupportsNonuniformThreadgroups`. Make this property a constant by replacing `var` with `let` and init it this way:

```swift
self.deviceSupportsNonuniformThreadgroups = library.device.supports(feature: .nonUniformThreadgroups)
```

The function constants can be created much cleaner:

```swift
let constantValues = MTLFunctionConstantValues()
constantValues.set(self.deviceSupportsNonuniformThreadgroups, at: 0)
```

To create a pipeline state now you don't need to init with a function. Instead, pass the function name directly to the pipeline state constructor:

```swift
self.pipelineState = try library.computePipelineState(function: "adjustments",
                                                      constants: constantValues)
```

The final variant of the `Adjustments` init looks like this:

```swift
init(library: MTLLibrary) throws {
    self.deviceSupportsNonuniformThreadgroups = library.device.supports(feature: .nonUniformThreadgroups)
    let constantValues = MTLFunctionConstantValues()
    constantValues.set(self.deviceSupportsNonuniformThreadgroups, at: 0)
    self.pipelineState = try library.computePipelineState(function: "adjustments",
                                                          constants: constantValues)
}
```

Next, stop is the encoding function. Currently, it looks large, and explicit and takes about 39 lines of code. Thanks to Alloy's extensions over the command buffer and command encoder, we can significantly reduce the amount of the code. Replace the encoding with the following:

```swift
func encode(source: MTLTexture,
            destination: MTLTexture,
            in commandBuffer: MTLCommandBuffer) {
    commandBuffer.compute { encoder in
    // ...
    }
}
```

As you can see, now you don't need to create an encoder by hand, instead we are using Swift's closures which look much cleaner. Now let's add the encoding logic:

- set the label

    ```swift
    encoder.label = "Adjustments"
    ```

- set the textures:

    ```swift
    encoder.setTextures(source, destination)
    ```

- set the floats:

    ```swift
    encoder.setValue(self.temperature, at: 0)
    encoder.setValue(self.tint, at: 1)
    ```

- dispatch the command:

    ```swift
    if self.deviceSupportsNonuniformThreadgroups {
        encoder.dispatch2d(state: self.pipelineState,
                           exactly: destination.size)
    } else {
        encoder.dispatch2d(state: self.pipelineState,
                           covering: destination.size)
    }
    ```

The result function looks like this:

```swift
func encode(source: MTLTexture,
            destination: MTLTexture,
            in commandBuffer: MTLCommandBuffer) {
    commandBuffer.compute { encoder in
        encoder.label = "Adjustments"
        encoder.setTextures(source, destination)
        encoder.setValue(self.temperature, at: 0)
        encoder.setValue(self.tint, at: 1)
        if self.deviceSupportsNonuniformThreadgroups {
            encoder.dispatch2d(state: self.pipelineState,
                               exactly: destination.size)
        } else {
            encoder.dispatch2d(state: self.pipelineState,
                               covering: destination.size)
        }
    }
}
```

and it takes only 16 lines of code! Note that we don't need to worry about the `encoder.endEncoding()`, because Alloy does it under the hood and reduces the amount of redundant code just by changing the way you call it.

### ViewController

Next, delete the `TextureManager` file. All its logic now can be replaced by Alloy's `MTLContext`. Navigate to the `ViewController`. Import Alloy and replace the `device`, the `commandQueue` and the `textureManager` properties with:

```swift
private let context: MTLContext
```

Now, the `ViewContoller`'s constructor should be modified:

```swift
init(context: MTLContext) throws {
    self.context = context
    self.adjustments = try .init(library: context.library(for: .main))
    self.imageView = .init()
    super.init(nibName: nil, bundle: nil)
    self.commonInit()
}
```

as well as the calling of this constructor in `SceneDelegate.swift`:

```swift
guard let windowScene = (scene as? UIWindowScene),
      let vc = try? ViewController(context: .init())
else { return }
```

Note, how easy it is to initialize Adjustments with just one line of code.

Next, the `handlePickedImage`'s texture creation logic needs to be replaced with:

```swift
guard let cgImage = image.cgImage,
      let source = try? self.context.texture(from: cgImage,
                                             srgb: false,
                                             usage: .shaderRead),
      let destination = try? source.matchingTexture(usage: [.shaderRead, .shaderWrite])
else { return }
```

The creation of the textures from images is now super easy.

Knowing that the context holds a command queue inside and the textures now can output images, we can delete the command queue property from the class and replace the `redraw` function with the following:

```swift
private func redraw() {
    guard let source = self.texturePair?.source,
          let destination = self.texturePair?.destination
    else { return }

    try? self.context.schedule { commandBuffer in
        self.adjustments.encode(source: source,
                                destination: destination,
                                in: commandBuffer)
        commandBuffer.addScheduledHandler { _ in
            DispatchQueue.main.async {
                self.imageView.image = try? destination.figure()
            }
        }
    }
}
```

Now it looks much better and cleaner. The bonus is that overall we reduced the number of lines of code by 122, which is very cool 🎉.

## Texture View

The creation of an image by every slider change is extremely inefficient: each time the system allocates memory, copies the texture bytes to it, creates an image, sets it to the image view and draws the layer to the screen. The better approach is just to render the texture directly to `CAMetalLayer`. To do that we will use [Texture View](https://github.com/eugenebokhan/texture-view). Internally, this little framework renders two triangles with a texture stretched over them.

{{
    figure(
        src="rendering.png",
        alt="rendering",
        caption=""
    )
}}

Let's add it as a dependency.

{{
    figure(
        src="add-alloy-package.png",
        alt="add-alloy-package",
        caption=""
    )
}}

 Import `TextureView` in the `ViewController`. Replace the image view with a texture view:

- the property:

```swift
private let textureView: TextureView
```

- in the `ViewController`'s constructor:

```swift
self.textureView = try .init(device: context.device)
```

- in the `commonInit` function:

```swift
// texture view
self.textureView.textureContentMode = .aspectFit
self.textureView.layer.cornerRadius = 10
self.textureView.layer.masksToBounds = true
self.view.addSubview(self.textureView)
self.textureView.backgroundColor = .tertiarySystemFill
self.textureView.snp.makeConstraints {
    $0.left.right.equalToSuperview().inset(20)
    $0.top.equalTo(self.view.safeAreaLayoutGuide).inset(20)
    $0.bottom.equalTo(settingsView.snp.top).inset(-20)
}
```

Replace the getting result image logic of `share` function with the following:

```swift
guard let destination = self.texturePair?.destination,
      let image = try? destination.figure()
else { return }
```

In the `handlePickedImage` replace the image view code with:

```swift
self.textureView.texture = destination
```

And finally, the redraw function should look like this:

```swift
private func redraw() {
    guard let source = self.texturePair?.source,
          let destination = self.texturePair?.destination
    else { return }
    DispatchQueue.main.async {
        try? self.context.schedule { commandBuffer in
            self.adjustments.encode(source: source,
                                    destination: destination,
                                    in: commandBuffer)
            self.textureView.draw(in: commandBuffer)
        }
    }
}
```

Nice! Now we have an image editing app with an end-to-end Metal pipeline 🤘. The source code of the result project can be found [here](https://github.com/eugenebokhan/introduction-to-metal-compute/tree/main/part-5). In the next Chapter, we will learn how to use another cool tool for kernel encoder code generation.
