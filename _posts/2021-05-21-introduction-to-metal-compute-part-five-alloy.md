---
layout: post
title:  "Introduction to Metal Compute: Alloy"
date:   2021-05-21 15:01:35 +0300
image:  '/images/introduction-to-metal-compute-alloy/icon.png'
tags:   iOS Metal
---

Hello everyone and welcome the fifth chapter of Introduction to Metal Compute! We made a lot things in the previous parts. We created a simple image editing app that is able to open, preview, adjust and export images. To do that, we wrote an image editing Metal shader kernel, created an encoder for it, learned, how to convert images to textures and pass the data to the GPU while dispatching the commands to it. The aim of this article is to encourage you to use more "Swifty" way of writing Metal related code. Also, we will migrate from `UIImageView` to `CAMetalLayer` for previewing the result.

## Alloy

Vanilla Metal provides an access to work with device's GPU. It practically does not add any abstraction and allows you to work in the same paradigm in which the hardware works. Being a low-level API, on one hand, Metal provides an ability to have a fine grained control over the hardware, and on the other hand, it introduces a little bit of complexity and redundancy in some cases. While writing metal pipeline, we operate such concepts as device, command queue, command buffer, command encoder, library, function and more. Some of these objects are created once and can be reused, others need to be initialised on every kernel dispatch. Some of them need to be initialised with their corresponding descriptors and some are not. In some places the API is throwable and returns optionals in the other. In general it feels like Metal was written for Objective-C users without any extra adaptation for Swift.

With all these thoughts in mind [Alloy](https://github.com/s1ddok/Alloy/) was born. This framework's purpose is to simplify Metal development on Swift, make the code cleaner and consistent without changing the main paradigm of low-level control over how things work. It provides nano-tiny layer over vanilla Metal API, that hides the majority of redundant explicity in the Metal code, while not limiting a flexibility a bit. Originally Alloy written in 2018 by my colleague [Andrey Volodin](https://twitter.com/s1ddok) and what concerns me, I was one of the main contributors to it for the last few years.

To make code more consistent, the Alloys API is designed to be throwable in those places where vanilla Metal is either throwable or returns optionals. A lot of extensions were added to device, texture, command queue and other classes to reduce the number of repeating boilerplate code. 

### Device

The device has been upgraded and by using it you are able to:

- allocate a heap without a descriptor:

{% highlight swift %}
let heap = try device.heap(size: 512,
                           storageMode: .shared,
                           cpuCacheMode: .defaultCache)
{% endhighlight %}

- create a texture with few lines of code:

{% highlight swift %}
let texture = try device.texture(width: 512,
                                 height: 512,
                                 pixelFormat: .bgra8Unorm,
                                 usage: [.shaderRead, .shaderWrite])
{% endhighlight %}

- allocate a buffer with a value:

{% highlight swift %}
let someValue = SIMD4<Float>(repeating: 1)
let buffer = try device.buffer(with: someValue,
                               options: .storageModeShared)
{% endhighlight %}

... and more!

### Command Queue

We extended the command queue with two convenience function that allow you to:

- dispatch a command buffer in async manner:

{% highlight swift %}
commandQueue.schedule { commandBuffer in
    // encoding logic
}
{% endhighlight %}

- dispatch a command buffer synchronously:

{% highlight swift %}
commandQueue.scheduleAndWait { commandBuffer in
    // encoding logic
}
{% endhighlight %}

### Command Buffer

Now encoding the commands to command buffer can be done by calling just one function. Also, you don't need to worry about committing the work. You can easily encode:

- a compute command:

{% highlight swift %}
commandBuffer.compute { computeCommandEncoder in
    // compute command encoding logic
}
{% endhighlight %}

- a render command:

{% highlight swift %}
commandBuffer.render(descriptor: MTLRenderPassDescriptor) { renderCommandEncoder in
    // render command encoding logic
}
{% endhighlight %}

- a blit command:

{% highlight swift %}
commandBuffer.blit { blitCommandEncoder
    // blit command encoding logic
}
{% endhighlight %}

### Compute Command Encoder

Remember how you passed data to shaders via compute command encoder? If you needed to pass any value, you needed to calculate it's size in bytes and pass a reference to the value. Now you can just call:

{% highlight swift %}
let someValue = SIMD4<Float>(repeating: 1)
encoder.setValue(someValue, at: 0)
{% endhighlight %}

or if you need to pass an array:

{% highlight swift %}
let someArray = [Float](repeating: 1, count: 256)
encoder.setValue(someArray, at: 0)
{% endhighlight %}

Also you can set a number of textures just by calling:

{% highlight swift %}
encoder.setTextures(textureOne, textureTwo)
{% endhighlight %}

and buffers:

{% highlight swift %}
encoder.setBuffers(bufferOne, bufferTwo)
{% endhighlight %}

One of key things is that now you don't need to write threadgrop size computations by hand and the code can be reduced just to:

{% highlight swift %}
if self.deviceSupportsNonuniformThreadgroups {
    encoder.dispatch2d(state: pipelineState,
                       exactly: size)
} else {
    encoder.dispatch2d(state: pipelineState,
                       covering: size)
}
{% endhighlight %}

### Textutre

What concerns textures, now you are able to create images from them by calling:

{% highlight swift %}
let image = try texture.image(colorSpace: .displayP3Space)
{% endhighlight %}

and pixel buffers:

{% highlight swift %}
let pixelBuffer = texture.pixelBuffer
{% endhighlight %}

Now it is easy to get `size`, `region` and a `descriptor` of a texture as well as to create it's empty copy:

{% highlight swift %}
let textureTwo = try textureOne.matchingTexture(usage: [.shaderRead, .shaderWrite],
                                                storage: .shared)
{% endhighlight %}

### Context

The only new concept that Alloy introduces is `MTLContext`. The context is an object that is designed to be injected across the app. Internally, the context holds references to such objects that remain the same over the whole metal pipeline lifecycle (device, command queue, library cache and texture loader) and provides a convenience API to maintain it. With the help of context you can:

- create a texture form `CGImage`:

{% highlight swift %}
let texture = try self.context.texture(from: CGImage,
                                       srgb: Bool?,
                                       usage: MTLTextureUsage,
                                       generateMipmaps: Bool)
{% endhighlight %}

- create a shaders library for a given bundle:

{% highlight swift %}
let library = try self.context.library(for: Bundle)
{% endhighlight %}

- do everything that a device and command queue can.

Also it is important to notice that this framework provides a set of handwritten utility kernels that are commonly used in image processing:

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

And yet we have covered just a little part of all the extensions that Alloy has. Alloy is a production ready tool and it is used in the development of all [Prisma](https://prisma-ai.com)'s apps. I highly recommend you to give this framework a try, and I am sure, you won't return to vanilla Metal any more 🙂.

## Demo App

Let's migrate our existing codebase to Alloy and see it in action. First, add is as a dependency to the project.

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-alloy/add-alloy-package.png"
alt="add-alloy-package" 
width="900"
/>
</p>

### Adjustments

Navigate to `Adjustments.swift` and replace `import Metal` with `import Alloy`.

Now let's modify the class constructor. First thing is `deviceSupportsNonuniformThreadgroups`. Make this property a constant by replacing `var` with `let`and init it this way:

{% highlight swift %}
self.deviceSupportsNonuniformThreadgroups = library.device.supports(feature: .nonUniformThreadgroups)
{% endhighlight %}

The function constants can be created much cleaner:

{% highlight swift %}
let constantValues = MTLFunctionConstantValues()
constantValues.set(self.deviceSupportsNonuniformThreadgroups, at: 0)
{% endhighlight %}

To crate a pipeline state now you don't need to init with a function. Instead, pass the function name directly to the pipeline state constructor:

{% highlight swift %}
self.pipelineState = try library.computePipelineState(function: "adjustments",
                                                      constants: constantValues)
{% endhighlight %}

The final variant of the `Adjustments` init looks like this:

{% highlight swift %}
init(library: MTLLibrary) throws {
    self.deviceSupportsNonuniformThreadgroups = library.device.supports(feature: .nonUniformThreadgroups)
    let constantValues = MTLFunctionConstantValues()
    constantValues.set(self.deviceSupportsNonuniformThreadgroups, at: 0)
    self.pipelineState = try library.computePipelineState(function: "adjustments",
                                                          constants: constantValues)
}
{% endhighlight %}

Next stop is the encoding function. Currently, it looks large, explicit and takes about 39 lines of code. Thanks to Alloy's extensions over command buffer and command encoder, we can significantly reduce the amount of the code. Replace the encoding with the following:

{% highlight swift %}
func encode(source: MTLTexture,
            destination: MTLTexture,
            in commandBuffer: MTLCommandBuffer) {
    commandBuffer.compute { encoder in
	// ...
    }
}
{% endhighlight %}

As you can see, now you don't need to create encoder by hand, instead we are using Swift's closures which looks much cleaner. Now let's add the encoding logic:

- set the label

{% highlight swift %}
encoder.label = "Adjustments"
{% endhighlight %}

- set the textures:

{% highlight swift %}
encoder.setTextures(source, destination)
{% endhighlight %}

- set the floats:

{% highlight swift %}
encoder.setValue(self.temperature, at: 0)
encoder.setValue(self.tint, at: 1)
{% endhighlight %}

- dispatch the command:

{% highlight swift %}
if self.deviceSupportsNonuniformThreadgroups {
    encoder.dispatch2d(state: self.pipelineState,
                       exactly: destination.size)
} else {
    encoder.dispatch2d(state: self.pipelineState,
                       covering: destination.size)
}
{% endhighlight %}

The result function looks like this:

{% highlight swift %}
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
{% endhighlight %}

and it takes only 16 lines of code! Note that we don't need to worry about the `encoder.endEncoding()`, because Alloy does it under the hood and reduces the amount of redundant code just by changing the way you call it.

### ViewController

Next, delete the `TextureManager` file. All its logic now can be replaced by Alloy's `MTLContext`. Navigate to the `ViewController`. Import Alloy and replace `device`, `commandQueue` and `textureManager` properties with:

{% highlight swift %}
private let context: MTLContext
{% endhighlight %}

Now, the `ViewContoller`'s constructor should be modified:

{% highlight swift %}
init(context: MTLContext) throws {
    self.context = context
    self.adjustments = try .init(library: context.library(for: .main))
    self.imageView = .init()
    super.init(nibName: nil, bundle: nil)
    self.commonInit()
}
{% endhighlight %}

as well as the calling of this constructor in `SceneDelegate.swift`:

{% highlight swift %}
guard let windowScene = (scene as? UIWindowScene),
      let vc = try? ViewController(context: .init())
else { return }
{% endhighlight %}

Note, how easy it is to initialise Adjustments with just one line of code.

Next, the `handlePickedImage`'s texture creation logic needs to be replaced with:

{% highlight swift %}
guard let cgImage = image.cgImage,
      let source = try? self.context.texture(from: cgImage,
                                             srgb: false,
                                             usage: .shaderRead),
      let destination = try? source.matchingTexture(usage: [.shaderRead, .shaderWrite])
else { return }
{% endhighlight %}

Creation textures from images and making matching textures is now super easy.

Knowing that the context holds a command queue inside and the textures now are able to output images, we can delete the command queue property from the class and replace the `redraw` function with the following:

{% highlight swift %}
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
                self.imageView.image = try? destination.image()
            }
        }
    }
}
{% endhighlight %}

Now it looks much better and cleaner. And the bonus is that overall we reduced the number of lines of code by 122, which is very cool 🎉. 

## Texture View

Creation an image by every slider change is extremely inefficient: each time the system allocates memory, copies the texture bytes to it, creates an image, sets it to the image view and draws the layer to the screen. The better approach is just to render the texture directly to `CAMetalLayer`. In order to do that we will use [Texture View](https://github.com/eugenebokhan/texture-view). Internally this little framework renders two triangles with a texture stretched over them.

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-alloy/rendering.png"
alt="rendering" 
width="600"
/>
</p>

Let's add it as a dependency.

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-alloy/add-alloy-package.png"
alt="add-alloy-package" 
width="900"
/>
</p>

 Import `TextureView` in the `ViewController`. Replace the image view with a texture view:

- the property:

{% highlight swift %}
private let textureView: TextureView
{% endhighlight %}

- in the `ViewController`'s constructor:

{% highlight swift %}
self.textureView = try .init(device: context.device)
{% endhighlight %}

- in the `commonInit` function:

{% highlight swift %}
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
{% endhighlight %}

Replace the getting result image logic of `share` function with the following:

{% highlight swift %}
guard let destination = self.texturePair?.destination,
      let image = try? destination.image()
else { return }
{% endhighlight %}

In the `handlePickedImage` replace the image view code with:

{% highlight swift %}
self.textureView.texture = destination
{% endhighlight %}

And finally, the the redraw function should look like this:

{% highlight swift %}
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
{% endhighlight %}

Nice! Now we have an image editing app with an end-to-end Metal pipeline 🤘. The source code of the result project can be found [here](https://github.com/eugenebokhan/introduction-to-metal-compute/tree/main/part-5). In the next chapter we will learn how to use another cool tool for kernel encoder codegeneration.