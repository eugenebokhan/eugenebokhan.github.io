+++
title = "Introduction to Metal Compute: Textures & Dispatching"
date = 2021-05-14

[taxonomies]
categories = ["Tech"]
tags = ["iOS", "Metal"]
+++

In this section, we will write image-to-texture conversion and the kernel dispatching code.

### MTLTexture, or There and Back Again

Create an empty swift file called `TextureManager.swift`.

{{
    figure(
        src="empty-file.png",
        alt="empty-file",
        caption=""
    )
}}

Create a class `TextureManager`. We are going to encapsulate texture-to-image conversion in it with a throwable API.

```swift
import MetalKit

final class TextureManager {
    
    enum Error: Swift.Error {
        case cgImageCreationFailed
        case textureCreationFailed
    }

}
```

The first and only property that this class will hold is [`MTKTextureLoader`](https://developer.apple.com/documentation/metalkit/mtktextureloader).

```swift
private let textureLoader: MTKTextureLoader
```

This object can read common file formats like PNG, JPEG, asset catalogs, `CGImage`s and decode them into Metal textures.

To initialize the class, `MTLDevice` is passed to the constructor:

```swift
init(device: MTLDevice) {
    self.textureLoader = .init(device: device)
}
```

## Image To Texture Conversion

Now let's write a function for creating a texture from a `CGImage` that will use the texture loader.

```swift
func texture(from cgImage: CGImage,
             usage: MTLTextureUsage = [.shaderRead, .shaderWrite]) throws -> MTLTexture {
    let textureOptions: [MTKTextureLoader.Option: Any] = [

    ]
    return try self.textureLoader.newTexture(cgImage: cgImage,
                                             options: textureOptions)
}
```

Here we create `MTKTextureLoader` options that specify how the result texture will be created. Currently, the options are empty and now we are going to fill the dictionary.

### Texture Usage

```swift
.textureUsage: NSNumber(value: usage.rawValue),
```

The texture usage `OptionSet` describes what operations will be performed with the texture. But given that we have images that we can read and modify like the other data, why does Metal provide such API? The point is that MTLTextures holds a reference to the block of memory where the real pixels are located and Metal can apply certain optimizations for a given texture, based on its intended use. It's a good practice to set `.shaderRead` for read-only textures and `.shaderWrite` to those that will contain the result.

### Mipmaps

```swift
.generateMipmaps: NSNumber(value: false),
```

[Mipmaps](https://en.wikipedia.org/wiki/Mipmap) are smaller, pre-filtered versions of a texture, representing different levels of detail (LOD) of the image. They are mostly used in 3D graphics to draw objects that are far away with less detail to save memory and computational resources.

{{
    figure(
        src="mipmaps.png",
        alt="mipmaps",
        caption=""
    )
}}

We won't use mipmaps in our app, so we disable their allocation and generation.

### Gamma Correction

```swift
.SRGB: NSNumber(value: false)
```

To understand why we pass `sRGB` as `false` to the texture options, first, we need to talk a little bit about gamma. Look a the picture below.

{{
    figure(
        src="gamma.png",
        alt="gamma",
        caption=""
    )
}}

The top line looks like the correct brightness scale to our eyes with consistent differences. The funny thing is that when we’re talking about the physical brightness of light, e.g., the number of photons leaving the display, the bottom line displays the correct brightness.

Now, look at the chart below. It depicts the difference in light perception between human eyes and a digital camera.

{{
    figure(
        src="chart.png",
        alt="chart",
        caption=""
    )
}}

As we can see in the chart, compared to a camera, we are much more sensitive to changes in dark tones than we are to similar changes in bright tones. There is a biological reason for this peculiarity: it is much more useful to perceive light in a non-linear manner and see better at night than during the day.

That's why the bottom line looks like an incorrect brightness scale despite the fact that the increase of an actual number of photons is linear.

So, given the fact that human perception of light is non-linear, in order not to waste bytes representing many lighter shades that look very similar to us, digital cameras do gamma encoding and redistribute tonal levels closer to how our eyes perceive them.

{{
    figure(
        src="gamma-correction.png",
        alt="gamma-correction",
        caption=""
    )
}}

This is done only for recording the image, not for displaying the image. Standard RGB or sRGB color space defines a nonlinear transfer function between the intensity of the color primaries and the actual number stored in such images.

If you load an sRGB image file, `iOS` will create a `CGImage` with **sRGB IEC61966-2.1** color profile and gamma decoding applied to the pixel values.
For the same image saved as a general RGB file, a `CGImage` will be created with **Adobe RGB (1998)** color profile and no gamma decoding applied to the pixels.
Both of these `CGImage` objects will contain the same pixel values in memory.

Now, getting back to the texture loader. When it creates a texture from a `CGImage`, it copies the bytes from `CGImage` data without modification. The `sRGB` option doesn't influence on anything with RGB images passed to the texture loader and you always get textures with ordinary `bgra8Unorm` pixel format. But when you use `CGImage` with sRGB color profile, this option does influence what pixel format the created texture will have. If you pass `false`, you will get a `bgra8Unorm` texture. If you pass `true`, you will get `bgra8Unorm_srgb` texture. And if you don't add this option at all, the texture loader will decide for you and return a `bgra8Unorm_srgb` texture too.

Why do we care about that? The thing is that in shaders sRGB textures are gamma decoded while being read and gamma encoded during the writing. It means that you will get *twice gamma decoded* pixel values in the kernel: first gamma decoding is done by the system when you create a `CGImage` from an sRGB image file, and second, is done by Metal when you pass sRGB texture to the shader. In this case, you get incorrect, darkened values in the shader. This looks pretty messy, so the best option is just always explicitly pass `sRGB: false` while creating textures from images to avoid gamma-related issues.

The final function should look like this:

```swift
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
```

## Texture To Image Conversion

Now, let's create a function to convert `MTLTexture` back to `CGImage`. Add an empty function:

```swift
func cgfigure(from texture: MTLTexture) throws -> CGImage {

}
```

In this function, we are going to allocate some memory where the texture will export its contents. Then we will create an image from this block of memory. From this point, we are going to fill the body of the function.

### Bytes Allocation

```swift
let bytesPerRow = texture.width * 4
let length = bytesPerRow * texture.height

let rgbaBytes = UnsafeMutableRawPointer.allocate(byteCount: length,
                                                 alignment: MemoryLayout<UInt8>.alignment)
defer { rgbaBytes.deallocate() }
```

First, we calculate bytes per row. In memory, the image is stored contiguously row by row with optional paddings between them. The paddings might be added for better CPU access to the blocks of memory. In our case we don't add any paddings and calculate the value as the width of the texture multiplied by 4 bytes per pixel.

{{
    figure(
        src="memory.png",
        alt="memory",
        caption=""
    )
}}

The length of the memory where the image bytes will be stored can be calculated as bytes per row multiplied by the texture height.
Next, we allocate a chunk of memory where the pixels will be stored. It is aligned by `UInt8` because  `bgra8Unorm` texture pixels are stored as 8-bit integers. This memory is temporary, so we will free it at the end of function execution.

### Texture Data Export

```swift
let destinationRegion = MTLRegion(origin: .init(x: 0, y: 0, z: 0),
                                  size: .init(width: texture.width,
                                              height: texture.height,
                                              depth: texture.depth))
texture.getBytes(pixelBytes,
                 bytesPerRow: bytesPerRow,
                 from: destinationRegion,
                 mipmapLevel: 0)
```

Calculate the region of the texture and call the [`getBytes`](https://developer.apple.com/documentation/metal/mtltexture/1516318-getbytes) function to store texture pixel values to the memory we allocated. This function takes a pointer to the start of the preallocated memory and writes pixel values of the specified region to it with predefined bytes per row. It is quite interesting that Metal doesn’t allow you to get the pointer to the raw pixels of the texture; instead, it allows you to export them via `getBytes` and import them with the help of [`replace(region:)`](https://developer.apple.com/documentation/metal/mtltexture/1515464-replace) function. The explanation is that Metal can adjust the private layout of the texture in memory to improve pixel access on GPU while sampling. This can be done with the texture descriptor’s `allowGPUOptimizedContents` flag set to `true`. There is no documentation on that, the texture memory layout may differ from GPU to GPU, but here is an example of how the memory reordering could look like this:

{{
    figure(
        src="gpu-content-optimisation.png",
        alt="gpu-content-optimisation",
        caption=""
    )
}}

### Bitmap

```swift
let colorScape = CGColorSpaceCreateDeviceRGB()
let bitmapInfo = CGBitmapInfo(rawValue: CGImageByteOrderInfo.order32Little.rawValue | CGImageAlphaInfo.noneSkipFirst.rawValue)
```

Create a color space and bitmap info values. The bitmap info can be interpreted as the pixels of the image are represented by 32 bits with little-endian byte order and we don't care about the information in the alpha channel. Here is the difference between little and big-endian byte orders:

{{
    figure(
        src="little-big-endian.png",
        alt="little-big-endian",
        caption=""
    )
}}

### CGImage Creation

```swift
guard let data = CFDataCreate(nil,
                              pixelBytes.assumingMemoryBound(to: UInt8.self),
                              length),
      let dataProvider = CGDataProvider(data: data),
      let cgImage = CGfigure(width: texture.width,
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
```

Finally, create a `CGImage` with the data and information we provided: each pixel contains 4 `UInt8`s or 32 bits, and each byte represents one channel. The layout of the pixels is described with bitmap info. The function should look like this:

```swift
func cgfigure(from texture: MTLTexture) throws -> CGImage {
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
          let cgImage = CGfigure(width: texture.width,
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
```

Let's add the last convenience function to the texture manager. This one helps to create a texture with similar properties.

```swift
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
```

We have finished with the texture manager. Now the object of this class can create a texture from a CGImage, create an image from a texture and create an empty copy of a texture with the same dimension pixel format, usage and storage.

## Kernel Dispatching

The final step is to create the command queue as well as the command buffer and pass it to the encoding function.

Navigate to `ViewController.swift` and add the following properties:

```swift
private let device: MTLDevice
private let commandQueue: MTLCommandQueue
private let textureManager: TextureManager
private let adjustments: Adjustments
private var texturePair: (source: MTLTexture, destination: MTLTexture)?
```

Now, declare `Error` enum and replace the constructor of this ViewController with the following:

```swift
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
```

Here we create the library for the main bundle and initialize the command queue, adjustments and texture manager.

Let's add the drawing function:

```swift
private func redraw() {
    guard let source = self.texturePair?.source,
          let destination = self.texturePair?.destination,
          let commandBuffer = self.commandQueue.makeCommandBuffer()
    else { return }

    self.adjustments.encode(source: source,
                            destination: destination,
                            in: commandBuffer)

    commandBuffer.addCompletedHandler { _ in
        guard let cgImage = try? self.textureManager.cgfigure(from: destination)
        else { return }

        DispatchQueue.main.async {
            self.imageView.image = UIfigure(cgImage: cgImage)
        }
    }

    commandBuffer.commit()
}
```

Inside the `redraw`, the command queue creates a command buffer. The `Adjustments` object encodes everything in the command buffer, and at the end, we commit it. After the command buffer is committed, Metal sends it to the GPU for execution.

{{
    figure(
        src="command-queue.png",
        alt="command-queue",
        caption=""
    )
}}

The final two steps are: to create textures from images and update the slider logic. Update the `handlePickedImage` function:

```swift
func handlePickedfigure(image: UIImage) {
    guard let cgImage = image.cgImage,
          let source = try? self.textureManager.texture(from: cgImage),
          let destination = try? self.textureManager.matchingTexture(to: source)
    else { return }
    
    self.texturePair = (source, destination)
    self.imageView.image = image
    self.redraw()
}
```

And finally, in the `commonInit` body replace the settings:

```swift
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
```

Hooray! We're finally done! From now on, if you have done everything correctly, you can compile and run the application 🎉.

{{
    figure(
        src="final-app.gif",
        alt="final-app",
        caption=""
    )
}}

In these four parts, we learned a lot: how to write a kernel, how to make an encoder for it, how to dispatch the work to the GPU and how to convert textures to images and backward, got familiar with gamma correction, the internals of how pixels are stored in memory and more. The source code of the final project is located [here](https://github.com/eugenebokhan/introduction-to-metal-compute/tree/main/part-1-4). But then that's not all yet. At this moment we are using `UIImage` to display the result. In the next part, we will replace it with rendering in [`CAMetalLayer`](https://developer.apple.com/documentation/quartzcore/cametallayer) and start using a more Swift-friendly API to work with Metal.
