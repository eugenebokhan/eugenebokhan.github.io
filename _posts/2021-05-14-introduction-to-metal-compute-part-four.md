---
layout: post
title:  "Introduction to Metal Compute: Textures & Dispatching"
date:   2021-05-14 15:01:35 +0300
image:  '/images/introduction-to-metal-compute-part-four/icon.png'
tags:   iOS Metal Texture
---

In this section we will write image to texture conversion and the kernel dispatching code.

### MTLTexture, or There and Back Again

Create an empty swift file called `TextureManager.swift`.

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-four/empty-file.png"
alt="demo app" 
width="800"
/>
</p>

Create a class `TextureManager`. We are going to incapsulate texture to image conversion in it with a throwable API.

{% highlight swift %}
import MetalKit

final class TextureManager {
    
    enum Error: Swift.Error {
        case cgImageCreationFailed
        case textureCreationFailed
    }

}
{% endhighlight %}

The first and the only property that this class will hold is [`MTKTextureLoader`](https://developer.apple.com/documentation/metalkit/mtktextureloader). 

{% highlight swift %}
private let textureLoader: MTKTextureLoader
{% endhighlight %}

This object is able read common file formats like PNG, JPEG, asset catalogs, `CGImage`s and decode them into Metal textures.

To initialise the class, `MTLDevice` is passed to the constructor:

{% highlight swift %}
init(device: MTLDevice) {
    self.textureLoader = .init(device: device)
}
{% endhighlight %}

## Image To Texture Conversion

Now let's write a function for creation a texture from a `CGImage` that will use the texture loader.

{% highlight swift %}
func texture(from cgImage: CGImage,
             usage: MTLTextureUsage = [.shaderRead, .shaderWrite]) throws -> MTLTexture {
    let textureOptions: [MTKTextureLoader.Option: Any] = [

    ]
    return try self.textureLoader.newTexture(cgImage: cgImage,
                                             options: textureOptions)
}
{% endhighlight %}

Here we create `MTKTextureLoader` options that specify how the result texture will be created. Currently the options are empty and now we are going to fill the dictionary.

#### Texture Usage

{% highlight swift %}
.textureUsage: NSNumber(value: usage.rawValue),
{% endhighlight %}

The texture usage `OptionSet` describes what operations will be performed with the texture. But given that we have images which we can read and modify like the other data, why does Metal provide such API? The point is that MTLTextures holds a reference to the block of memory where the real pixels are located and Metal can apply certain optimisations for a given texture, based on its intended use. I's a good practice to set `.shaderRead` for read-only textures and `.shaderWrite` to those which will contain the result.

#### Mipmaps

{% highlight swift %}
.generateMipmaps: NSNumber(value: false),
{% endhighlight %}

[Mipmaps](https://en.wikipedia.org/wiki/Mipmap) are smaller, pre-filtered versions of a texture, representing different levels of detail (LOD) of the image. They are mostly used in 3D graphics to draw objects that are far away with less detail to save memory and computational resources.

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-four/mipmaps.png"
alt="demo app" 
width="500"
/>
</p>

We won't use mipmaps in our app, so we disable their allocation and generation. 

#### Gamma Correction

{% highlight swift %}
.SRGB: NSNumber(value: false)
{% endhighlight %}

In order to understand why do we pass `sRGB` as `false` to the texture options, first we need to talk a little bit about gamma. Look a the picture below. 

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-four/gamma.png"
alt="demo app" 
width="600"
/>
</p>

The top line looks like the correct brightness scale to our eyes with consistent differences. The funny thing is that when we’re talking about the physical brightness of light, e.g., the number of photons leaving the display, the bottom line actually displays the correct brightness.

Now, look at the chart below. It depicts the difference of light perceivation between human eyes and a digital camera.

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-four/chart.png"
alt="demo app" 
width="500"
/>
</p>

As we can see in the chart, compared to a camera, we are much more sensitive to changes in dark tones than we are to similar changes in bright tones. There is a biological reason for this peculiarity: it is much more useful to perceive light in a non-linear manner and see better at night than during the day.

That's why the bottom line looks like incorrect brightness scale despite the fact that the increase of actual number of photons is linear.

So, given the fact that human perceivation of light is non-linear, in order not to waste bytes representing many lighter shades that look very similar to us, digital cameras do gamma encoding and redistribute tonal levels closer to how our eyes perceive them.

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-four/gamma-correction.png"
alt="demo app" 
width="250"
/>
</p>

This is done only for recording the image, not for displaying the image. Standard RGB or sRGB color space defines a nonlinear transfer function between the intensity of the color primaries and the actual number stored in such images.

If you load an sRGB image file, `iOS` it will create a `CGImage` with **sRGB IEC61966-2.1** color profile and gamma decoding applied to the pixel values.
For the same image saved as general RGB file a `CGImage` will be created with **Adobe RGB (1998)** color profile and no gamma decoding applied to the pixels.
Both of these `CGImages` will contain the same pixel values in memory.

Now, getting back to the texture loader. When it creates a texture from a `CGImage`, it copies the bytes from `CGImage` data without modification. The `sRGB` option doesn't influence on anything with RGB images passed to the texture loader and you always get textures with ordinary `bgra8Unorm` pixel format. But when you use `CGImage`s with sRGB color profile, this option does influence on what pixel format the created texture will have. If you pass `false`, you will get a `bgra8Unorm` texture. If you pass `true` , you will get `bgra8Unorm_srgb` texture. And if you don't add this option at all, the texture loader will make a decision for you and return an `bgra8Unorm_srgb` texture too.

Why do we care about that? The thing is that in shaders sRGB textures are gamma decoded while being read and gamma encoded during the writing. It means that you will get *twice gamma decoded* pixel values in the kernel: first gamma decoding is done by the system when you create a `CGImage` from an sRGB image file, and second is done by Metal when you passed sRGB texture to the shader. In this case you get incorrect, darkened values in the shader. This looks pretty messy, so the best option is just always explicitly pass `sRGB: false` while creating textures from images to avoid gamma-related issues.

The final function should look like this:

{% highlight swift %}
func texture(from cgImage: CGImage,
             usage: MTLTextureUsage = [.shaderRead, .shaderWrite]) throws -> MTLTexture {
    let textureOptions: [MTKTextureLoader.Option: Any] = [
        .textureUsage: NSNumber(value: usage.rawValue),
        .generateMipmaps: NSNumber(value: false),
        .SRGB: NSNumber(value: false)
    ]
    return try self.textureLoader.newTexture(cgImage: cgImage,
                                             options: textureOptions)
}
{% endhighlight %}

## Texture To Image Conversion

Now, let's create a function to convert `MTLTexture` back to `CGImage`. Add an empty function:

{% highlight swift %}
func cgImage(from texture: MTLTexture) throws -> CGImage {

}
{% endhighlight %}

In this function we are going to allocate some memory where the texture will export its contents. Then we will create an image from this block of memory. From this point we are going to fill the body of the function.

#### Bytes Allocation

{% highlight swift %}
let bytesPerRow = texture.width * 4
let length = bytesPerRow * texture.height

let rgbaBytes = UnsafeMutableRawPointer.allocate(byteCount: length,
                                                 alignment: MemoryLayout<UInt8>.alignment)
defer { rgbaBytes.deallocate() }
{% endhighlight %}

First, we calculate bytes per row. In memory the image is stored contiguously row by row with optional paddings between them. The paddings might be added for better CPU access to the blocks of memory. In our case we don't add any paddings and calculate the value as the width of the texture multiplied by 4 bytes per pixel.

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-four/memory.png"
alt="demo app" 
width="700"
/>
</p>

The length of the memory where the image bytes will be stored can be calculated as bytes per row multiplied by the texture height.
Next, we allocate a chunk of memory where the pixels will be stored. It is aligned by `UInt8` because  `bgra8Unorm` textures pixels are stored as 8-bit integers. This memory is temporary, so we will free it at the end of function execution.

#### Texture Data Export

{% highlight swift %}
let destinationRegion = MTLRegion(origin: .init(x: 0, y: 0, z: 0),
                                  size: .init(width: texture.width,
                                              height: texture.height,
                                              depth: texture.depth))
texture.getBytes(pixelBytes,
                 bytesPerRow: bytesPerRow,
                 from: destinationRegion,
                 mipmapLevel: 0)
{% endhighlight %}

Calculate the region of the texture and call the [`getBytes`](https://developer.apple.com/documentation/metal/mtltexture/1516318-getbytes) function to store texture pixel values to the memory we allocated. This function takes a pointer to the start of the preallocated memory and writes pixel values of the specified region to it with predefined bytes per row. It is quite interesting that Metal doesn’t allow you to get the pointer to the raw pixels of the texture; instead, it allows you to export them via `getBytes` and import them with the help of [`replace(region:)`](https://developer.apple.com/documentation/metal/mtltexture/1515464-replace) function. The explanation is that Metal can adjust the private layout of the texture in memory to improve pixel access on GPU while sampling. This can be done with texture descriptor’s `allowGPUOptimizedContents` flag set `true`. There is no documentation on that, the texture memory layout may differ from GPU to GPU, but here is an example of how the memory reordering could look like this:

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-four/gpu-content-optimisation.png"
alt="demo app" 
width="700"
/>
</p>

#### Bitmap

{% highlight swift %}
let colorScape = CGColorSpaceCreateDeviceRGB()
let bitmapInfo = CGBitmapInfo(rawValue: CGImageByteOrderInfo.order32Little.rawValue | CGImageAlphaInfo.noneSkipFirst.rawValue)
{% endhighlight %}

Create a color space and bitmap info values. The bitmap info can be interpreted as: the pixels of the image are represented by 32 bits with little-endian byte order and we don't care about the information in the alpha channel. Here is the difference between little and big endian byte orders:

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-four/little-big-endian.png"
alt="demo app" 
width="500"
/>
</p>

#### CGImage Creation

{% highlight swift %}
guard let data = CFDataCreate(nil,
                              pixelBytes.assumingMemoryBound(to: UInt8.self),
                              length),
      let dataProvider = CGDataProvider(data: data),
      let cgImage = CGImage(width: texture.width,
                            height: texture.height,
                            bitsPerComponent: 8,
                            bitsPerPixel: 32,
                            bytesPerRow: bytesPerRow,
                            space: colorScape,
                            bitmapInfo: bitmapInfo,
                            provider: dataProvider,
                            decode: nil,
                            shouldInterpolate: true,
                            intent: .defaultIntent)
else { throw Error.cgImageCreationFailed }
return cgImage
{% endhighlight %}

Finally, create a `CGImage` with the data and information we provided: each pixel contains of 4 `UInt8`s or 32 bits, each byte is representing one channel. The layout of the pixels is described with bitmap info. The function should look like this:

{% highlight swift %}
func cgImage(from texture: MTLTexture) throws -> CGImage {
    let bytesPerRow = texture.width * 4
    let length = bytesPerRow * texture.height

    let rgbaBytes = UnsafeMutableRawPointer.allocate(byteCount: length,
                                                     alignment: MemoryLayout<UInt8>.alignment)
    defer { rgbaBytes.deallocate() }

    let destinationRegion = MTLRegion(origin: .init(x: 0, y: 0, z: 0),
                                      size: .init(width: texture.width,
                                                  height: texture.height,
                                                  depth: texture.depth))
    texture.getBytes(rgbaBytes,
                     bytesPerRow: bytesPerRow,
                     from: destinationRegion,
                     mipmapLevel: 0)

    let colorScape = CGColorSpaceCreateDeviceRGB()
    let bitmapInfo = CGBitmapInfo(rawValue: CGImageByteOrderInfo.order32Little.rawValue | CGImageAlphaInfo.premultipliedFirst.rawValue)

    guard let data = CFDataCreate(nil,
                                  rgbaBytes.assumingMemoryBound(to: UInt8.self),
                                  length),
          let dataProvider = CGDataProvider(data: data),
          let cgImage = CGImage(width: texture.width,
                                height: texture.height,
                                bitsPerComponent: 8,
                                bitsPerPixel: 32,
                                bytesPerRow: bytesPerRow,
                                space: colorScape,
                                bitmapInfo: bitmapInfo,
                                provider: dataProvider,
                                decode: nil,
                                shouldInterpolate: true,
                                intent: .defaultIntent)
    else { throw Error.cgImageCreationFailed }
    return cgImage
}
{% endhighlight %}

Let's add the last convenience function to the texture manager. This one helps to create a texture with similar properties.

{% highlight swift %}
func matchingTexture(to texture: MTLTexture) throws -> MTLTexture {
    let matchingDescriptor = MTLTextureDescriptor()
    matchingDescriptor.width = texture.width
    matchingDescriptor.height = texture.height
    matchingDescriptor.usage = texture.usage
    matchingDescriptor.pixelFormat = texture.pixelFormat
    matchingDescriptor.storageMode = texture.storageMode

    guard let matchingTexture = self.textureLoader.device.makeTexture(descriptor: matchingDescriptor)
    else { throw Error.textureCreationFailed }

    return matchingTexture
}
{% endhighlight %}

We have finished with texture manager. Now the object of this class is able to create a texture from a CGImage, create an image from a texture and create an empty copy of a texture with the same dimension pixel format, usage and storage.

## Kernel Dispatching

The final step is to create a command queue, a command buffer and pass it to the encoding function.

Navigate to `ViewController.swift` and add the following properties:

{% highlight swift %}
private let device: MTLDevice
private let commandQueue: MTLCommandQueue
private let textureManager: TextureManager
private let adjustments: Adjustments
private var texturePair: (source: MTLTexture, destination: MTLTexture)?
{% endhighlight %}

Now, declare `Error` enum and replace the constructor of this ViewController with the following:

{% highlight swift %}
enum Error: Swift.Error {
    case commandQueueCreationFailed
}

// ...

init(device: MTLDevice) throws {
    let library = try device.makeDefaultLibrary(bundle: .main)
    guard let commandQueue = device.makeCommandQueue()
    else { throw Error.commandQueueCreationFailed }
    self.device = device
    self.commandQueue = commandQueue
    self.imageView = .init()
    self.adjustments = try .init(library: library)
    self.textureManager = .init(device: device)
    super.init(nibName: nil, bundle: nil)
    self.commonInit()
}
{% endhighlight %}

Here we create the library for the main bundle and initialise command queue, adjustments and texture manager.

Let's add the drawing function:

{% highlight swift %}
private func redraw() {
    guard let source = self.texturePair?.source,
          let destination = self.texturePair?.destination,
          let commandBuffer = self.commandQueue.makeCommandBuffer()
    else { return }

    self.adjustments.encode(source: source,
                            destination: destination,
                            in: commandBuffer)

    commandBuffer.addCompletedHandler { _ in
        guard let cgImage = try? self.textureManager.cgImage(from: destination)
        else { return }

        DispatchQueue.main.async {
            self.imageView.image = UIImage(cgImage: cgImage)
        }
    }

    commandBuffer.commit()
}
{% endhighlight %}

Inside the `redraw`, the command queue creates a command buffer. The `Adjustments` object encodes everything in the command buffer, and at the end, we commit it. After the command buffer is committed, Metal sends it to the GPU for execution.

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-four/command-queue.png"
alt="demo app" 
width="500"
/>
</p>

The final two steps are: create textures from images and update the sliders logic. Update the `handlePickedImage` function:

{% highlight swift %}
func handlePickedImage(image: UIImage) {
    guard let cgImage = image.cgImage,
          let source = try? self.textureManager.texture(from: cgImage),
          let destination = try? self.textureManager.matchingTexture(to: source)
    else { return }
    
    self.texturePair = (source, destination)
    self.imageView.image = image
    self.redraw()
}
{% endhighlight %}

And finally, in the `commonInit` body replace the settings:

{% highlight swift %}
self.settings.settings = [
    FloatSetting(name: "Temperature",
                 defaultValue: .zero,
                 min: -1,
                 max: 1) {
        self.adjustments.temperature = $0
        self.redraw()
    },
    FloatSetting(name: "Tint",
                 defaultValue: .zero,
                 min: -1,
                 max: 1) {
        self.adjustments.tint = $0
        self.redraw()
    },
]
{% endhighlight %}

Hooray! We're finally done! From now on, if you have done everything correctly, you can compile and run the application 🎉.

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-four/final-app.gif"
alt="demo app" 
width="900"
/>
</p>

In these four parts, we learned a lot: how to write a kernel, how to make an encoder for it, how to dispatch the work to the GPU and how to convert textures to images and backwards, got familiar with gamma correction, the internals of how pixels are stored in memory and more. The source code of the final project is located [here](https://github.com/eugenebokhan/introduction-to-metal-compute/tree/main/part-1-4). But the that's not all yet. At this moment we are using `UIImage` to display the result. In the next part, we will replace it with rendering in [`CAMetalLayer`](https://developer.apple.com/documentation/quartzcore/cametallayer) and start using more Swift-friendly API to work with Metal.