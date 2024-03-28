+++
title = "Introduction to Metal Compute: Kernel Shader"
date = 2021-05-09

[taxonomies]
categories = ["Tech"]
tags = ["iOS", "Metal"]
+++

## Starter Project

First, clone [this](https://github.com/eugenebokhan/introduction-to-metal-compute) repository. It contains two folders: the starter and the final project. Open the `XCode` project in the `starter-project` folder.

{{
    figure(
        src="starter-project.png",
        alt="starter-project",
        caption=""
    )
}}

The starter project contains the boilerplate code. The main logic is located in `ViewController.swift`. If you compile and run the `Image Editor Demo` app you'll see that it is able only to choose an image display it in `UIImageView` and export via `UIActivityViewController`. There are two `Temperature` and `Tint` sliders that do absolutely nothing. From this point, we're going to add image editing functionality to our app.

## GPU Side: Image Editing Kernel

To adjust images using GPU, we need to provide some sort of instructions or a program to it. Originally such GPU programs were used only in 3D pipelines and often were responsible for lighting and shading effects. That is why over time, they were named shaders. `Metal` shaders are written in [`Metal Shading Language`](https://developer.apple.com/metal/Metal-Shading-Language-Specification.pdf) which is a subset of `C++`. It means that MSL has some restrictions (there are no lambda expressions, dynamic_cast operator, etc.) and extensions (support of textures, buffers, etc.), but mostly shader functions are pretty similar to ordinary C++ code.

Let's create a `Shaders.metal` file that will contain our compute code.

{{
    figure(
        src="create-shaders-file.png",
        alt="create-shaders-file",
        caption=""
    )
}}

It should look like this:

{{
    figure(
        src="empty-shaders-file.png",
        alt="empty-shaders-file",
        caption=""
    )
}}

Now, add the following snippet of code:

```cpp
kernel void adjustments(texture2d<float, access::read> source [[ texture(0) ]],
                        texture2d<float, access::write> destination [[ texture(1) ]],
                        uint2 position [[ thread_position_in_grid ]]) {
    const auto textureSize = ushort2(destination.get_width(),
                                     destination.get_height());
    if (position.x >= textureSize.x || position.y >= textureSize.y) {
        return;
    }
    const auto sourceValue = source.read(position);
    destination.write(sourceValue, position);
}
```

This is our starter shader function. Currently, it is able only to copy pixels from one image to another. Let's take a look, at how it works.

### Kernel Function Declaration

```cpp
kernel /* void adjustments ... */
```

The first declaration is the `kernel`. It means that the following function is a compute kernel - a set of instructions for general-purpose computing. Metal also provides `vertex` and `fragment` types of functions used in 3D.

```cpp
/* kernel */ void /* adjustments ... */
```

Next, `void` means that our function does not return anything. Kernels are always void, they only read, modify and write the data.

```cpp
/* kernel void */ adjustments /* ... */
```

The name of our function is `adjustments`. You can call your functions whatever you like. The only function naming restriction in MSL is that you cannot call your function `main`.

### Kernel Function Arguments

```cpp
/*kernel void adjustments(*/texture2d<float, access::read> source [[ texture(0) ]],
                            texture2d<float, access::write> destination [[ texture(1) ]],
//                          uint2 position [[ thread_position_in_grid ]]) {
```

In the arguments section of the function, we can see the source, destination and position. The source and destination are `textures`. A texture is a structured collection of texture elements, often called `texels` or `pixels`. The exact configuration of these texture elements depends on the type of texture. The source pixels are stored in a two-dimensional texture as floats. This is exactly what is described with a templated type `texture2d<float, access::read>`. To write the result, we use a texture of a similar type with `access::write`.

Metal provides several texture templates: `texture1d`, `texture2d`, `texture3d`, `texture1d_array`, `texture2d_array` and more. You can see all of them in chapter 2.8 of MSL specification, but speaking about image processing, you will need only `texture2d`.

The first texture template parameter is the data type. It specifies the type of one of the components returned when reading from a texture or the type of one of the components specified when writing to the texture. The data type can be `float`, `half`, `int`, `short`, `uint` or `ushort`. Most of the time you're going to use `float` and `half`.

The `access` template parameter describes the way of access to texture data:

* `read` means that you can access this texture only for reading;
* `write` is used for destination textures to write results into;
* `read_write` can be used for textures that can be used both for reading and writing. Note, that `read_write` textures are supported only on latest devices (Apple A11 devices and later);
* `sample` gives an ability to both read and sample texture with a sampler. Sampling is not only a more advanced way of gathering data than reading, but also it takes more time.

Each texture has to be provided with a unique identifier. It is done with the `[[ texture(n) ]]` attribute, where `n` is used as an index of texture slot while passing the texture object to the shader encoder on the CPU side.

```cpp
//                      texture2d<float, access::write> destination [[ texture(1) ]],
                        uint2 position [[ thread_position_in_grid ]]/* ) { */
// const auto textureSize = ushort2(destination.get_width(),
```

The final argument is `position`. When a kernel function is submitted for execution, it executes over an `n`-dimensional grid of threads, where `n` is one, two or three. A thread is an instance of the kernel function that executes for each point in this grid, and `thread_position_in_grid` identifies its position in the grid.

Generally, while working with images, you aim to dispatch a grid of threads of the same dimension as the image. In such cases, there is a correspondence between the position of a destination pixel and the position of a thread in the grid which computes the result value.

### Kernel Function Body: Boundary Check

```cpp
//                     uint2 position [[ thread_position_in_grid ]]) {
   const auto textureSize = ushort2(destination.get_width(),
                                    destination.get_height());
   if (position.x >= textureSize.x || position.y >= textureSize.y) {
       return;
   }
// const auto sourceValue = source.read(position);
```

Now let's look at the body of the kernel function. First, we get the size of the texture. Then inside of the `if` statement we use the texture size to ignore out-of-bounds execution via early return. To understand why we do this, we need to get familiar with the structure of the parallel work of threads.

Threads are organized into threadgroups that are executed together and can share a common threadgroup memory. In most image processing kernels threads run independently of each other, but sometimes shader functions are designed so that threads in a threadgroup collaborate on their working set, for example, while calculating texture mean, min or max.

The threads in a threadgroup are further organized into single-instruction, multiple-data (SIMD) groups, that execute concurrently. It is important to notice that the threads in a SIMD group execute the same code. If there is an `if` branching in the shaders code and one of the threads in the SIMD group takes a different path from the others, all threads in that group execute both branches, and the execution time for the group is the sum of the execution time of both branches. So It is a good practice to avoid `if` statements in shaders or make them as *thin* as possible.

So, given that we need to minimize the number of `if`s in shaders, why do we still have it at the beginning of the function? The answer is that old Metal-backed devices can operate only uniform-sized threadgroups which creates a constraint on the total size of the grid. To support old devices you need to dispatch such an amount of threadgroups that the entire size of the grid overlaps the size of the image. And to ignore out-of-bounds execution on the edges of the grid, we make an early return.

{{
    figure(
        src="threadgroups.png",
        alt="threadgroups",
        caption=""
    )
}}

Modern devices support non-uniform-sized threadgroups and Metal can generate smaller threadgroups along the edges of the grid, as shown below.

{{
    figure(
        src="non-uniform.png",
        alt="non-uniform",
        caption=""
    )
}}

To optimize the instructions for modern devices and avoid unnecessary code branching, we can create a separate version of our kernel without the boundary check.

One of the traditional ways to do it is to use preprocessor macro defines. For example, we could do something like this:

```cpp
#ifndef DEVICE_SUPPORTS_NON_UNIFORM_TREADGROUPS
if (position.x >= textureSize.x || position.y >= textureSize.y) {
    return;
}
#endif
```

Compiling one function many times with different preprocessor macros to enable different features is called `uber shaders`. However, this approach has a drawback as the size of the result shading library increases significantly.

Another way is to use Metal's `function constants`. Function constants provide the same ease of use as preprocessor macros but move the generation of the specific variants to the creation of the compute pipeline state - the state the GPU is in during the execution of the instructions, so you don't have to compile the variants offline.

Let's declare our function constant by adding the following piece of code before the kernel function declaration:

```cpp
constant bool deviceSupportsNonuniformThreadgroups [[ function_constant(0) ]];
```

Next, replace the boundary check with the following:

```cpp
if (!deviceSupportsNonuniformThreadgroups) {
    if (position.x >= textureSize.x || position.y >= textureSize.y) {
        return;
    }
}
```

Similar to textures, FCs also need to be provided with identifiers with the help of an attribute `[[ function_constant(n) ]]`. As you can see, function constants are not initialized in the Metal function source. Instead, using `n`, their values are specified during the creation of the compute pipeline state. To learn more about FCs, look at Chapter 5.8 of MSL spec.

Great! Now, if the device supports non-uniform threadgroups, the compute pipeline state will be initialized with function constant `deviceSupportsNonuniformThreadgroups` set to `true`, and the boundary check will be removed from the GPU instructions.

Boundary check is a common pattern used almost in every image processing compute shader, so honestly saying, it is just copy-pasted every time at the beginning of the functions 🙂.

### Kernel Function Body: Texture Read & Write

```cpp
//    const auto textureSize = ushort2(destination.get_width(),
//                                     destination.get_height());
//    if (!deviceSupportsNonuniformThreadgroups) {
//        if (position.x >= textureSize.x || position.y >= textureSize.y) {
//            return;
//        }
//    }
      const auto sourceValue = source.read(position);
      destination.write(sourceValue, position);
// }
```

Finally, the last two lines of code demonstrate, how to read and write texture data. To get pixel values from a certain position, you can use the `read` texture member function, and `write` to store the values. Metal also allows you to `sample` and `gather` from a texture as well as get its width, height and mipmap level count. If you want to learn more about it, read Chapter 2.8 of the MSL specification.

### Kernel Function: Adjustments

Now let's add the adjustments functionality to our shader. Replace the arguments of the kernel with the following:

```cpp
texture2d<float, access::read> source [[ texture(0) ]],
texture2d<float, access::write> destination [[ texture(1) ]],
constant float& temperature [[ buffer(0) ]],
constant float& tint [[ buffer(1) ]],
uint2 position [[thread_position_in_grid]]
```

All arguments to functions that are a pointer or reference to a type must be declared with an address space attribute. An address space attribute specifies the region of memory from where buffer memory objects are allocated. There are several address spaces: `device`, `constant`, `thread`, `threadgroup`, `threadgroup_imageblock`, but the most commonly used are the first two spaces. The `device` address space name refers to buffer memory objects allocated from the device memory pool that are both readable and writeable while `constant` address space refers to read-only memory. A buffer memory object can be declared as a pointer or reference to a scalar, vector or user-defined structure. If you're sure you need to pass some data to the shader and you won't modify it, It's a good practice to use `constant` address space, because Metal applies some optimizations on such buffers for better access to the memory.

Similar to textures and function constants, the attribute `[[ buffer(n) ]]` sets an ID to a buffer.

So, we passed temperature and tint references to floats that will be modified with UI sliders on the CPU side and accessed as read-only on the GPU side. Let's use these values for adjusting pixels of the texture. To change the temperature and tint of the image, first, we need to convert its [color space](https://en.wikipedia.org/wiki/Color_space) from [RGB](https://en.wikipedia.org/wiki/RGB_color_space) to [LAB](https://en.wikipedia.org/wiki/CIELAB_color_space). Color spaces are a huge subject for another article, but the main idea is that we can interpret color values in different ways, and the commonly used approach for image editing requires working in LAB. Now, import a header with convenience conversion functions below `metal_stdlib` include.

```cpp
#include "ColorConversion.h"
```

Next, replace the last two lines of the adjustments function with the following code:

```cpp
const auto sourceValue = source.read(position);
auto labValue = rgb2lab(sourceValue.rgb);
labValue = denormalizeLab(labValue);

labValue.b += temperature * 10.0f;
labValue.g += tint * 10.0f;

labValue = clipLab(labValue);
labValue = normalizeLab(labValue);
const auto resultValue = float4(lab2rgb(labValue), sourceValue.a);

destination.write(resultValue, position);
```

As we can see, each thread reads a pixel value from a texture at its position, converts the value from RGB color space to LAB, adjusts the value using temperature and tint arguments and converts it back to RGB. The result value is written to the destination texture at the same position it was read from a source.

If you did everything right, the final kernel should look like this:

```cpp
#include <metal_stdlib>
#include "ColorConversion.h"
using namespace metal;

constant bool deviceSupportsNonuniformThreadgroups [[ function_constant(0) ]];

// MARK: - Adjustments

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
    auto labValue = rgb2lab(sourceValue.rgb);
    labValue = denormalizeLab(labValue);
    
    labValue.b += temperature * 10.0f;
    labValue.g += tint * 10.0f;
    
    labValue = clipLab(labValue);
    labValue = normalizeLab(labValue);
    const auto resultValue = float4(lab2rgb(labValue), sourceValue.a);
    
    destination.write(resultValue, position);
}

```

Congratulations! You've written your first metal compute shader 🎉! In the next chapter, we are going to write the encoder for this kernel.
