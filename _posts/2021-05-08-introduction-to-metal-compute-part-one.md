---
layout: post
title:  "Introduction to Metal Compute"
date:   2021-05-08 15:01:35 +0300
image:  '/images/introduction-to-metal-compute-part-one/icon.png'
tags:   iOS Metal
---

I've been working as an iOS software engineer focusing on GPGPU using Metal for a couple of years.  It is an exciting sphere of iOS development that still lacks spread. In a series of articles, I will describe how to build a simple image-processing Metal app.

## Importance of visual data processing

"It is better to see something once than to hear about it a thousand times." - Asian proverb.

Most of us hear that the brain gets most of the information about the surrounding world with the help of the eyes. The eye’s retina, which contains 150 million light-sensitive rod and cone cells, is an outgrowth of the brain. In the brain, neurons devoted to visual processing number in the hundreds of millions and take up about 30% of the cortex, compared with 8 % for touch and just 3% for hearing. Humans are visual creatures, and graphical information, such as paintings, photos, videos, 3D games, etc., surrounds us everywhere.

The tech industry has always tried to help us with the constantly increasing need to create and process graphical data. With the distribution of various social networks, image and video processing apps gained popularity over the last years. Being an iOS software engineer, it is pretty promising to master modern ways of processing visual data. Currently, the most efficient way to work with images is to use horsepowers of device’s GPU with the lowest abstraction layer. Luckily, Apple platforms provide us with such possibilities with Metal API. But first, let’s look at the brief history of how Metal came to life.

## Road to Metal

First, it was all about drawing graphics on the screen as fast as possible. In the 1980s and early 1990s, the CPU often did this kind of work, which, due to its architecture, was not very efficient at that task. As with all demanding and domain-specific tasks, this work was gradually offloaded to a dedicated processor: the graphics processing unit was born.

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-one/gpu.png"
alt="demo app" 
width="500"
/>
</p>

At first, GPU vendors focused on 2D acceleration for desktop systems, monitor resolution, and the quality of the generated analog signal. However, a new branch gained popularity over time: 3D acceleration. Graphics APIs like DirectX, OpenGL, and 3dfx Glide, developed for computer games and data visualization, were designed with hardware support in mind. More and more calculation steps of these APIs were moved to dedicated hardware.

As the GPUs became faster in rendering basic computer graphics, demand for more advanced techniques grew. New pipeline stages for these effects were added to the APIs and quickly implemented in hardware. At that time, most functionalities were “fixed functions”. That is, for each effect, dedicated API calls existed and were implemented in hardware. Each new effect resulted in API and hardware changes, significantly limiting the possibilities of graphics programmers.

After about 2001, general-purpose computing on GPUs became more practical and popular. The early efforts to use GPUs as general-purpose processors required reformulating computational problems in terms of graphics primitives.

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-one/gpgpu.png"
alt="demo app" 
width="600"
/>
</p>

This limitation resulted in a significant architecture switch of GPUs. Highly specialized function units were replaced by small and straightforward moderately specialized processors. Simple and limited programmable execution units replaced the complete fixed functions. This significantly increased the flexibility of the hardware and allowed it to leverage the speed of a GPU without requiring complete and explicit conversion of the data to a graphical form.

The idea to ignore the underlying graphical concepts in favor of more common high-performance computing concepts became the basis of frameworks like Nvidia's CUDA, Microsoft's DirectCompute, and Apple/Khronos Group's OpenCL. This means that modern GPGPU pipelines can leverage the speed of a GPU without requiring complete and explicit conversion of the data to a graphical form.

Speaking about Apple platforms, before 2014, to take advantage of the GPU's power, one could use OpenGL and OpenCL on macOS and OpenGL ES on iOS. OpenGL, a low-level API that provides programmable access to GPU hardware for 3D graphics, still tended to hide the communication between the CPU and the GPU, leading to performance overhead problems. Also, there was no way to use GPU for general-purpose computations on iOS.

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-one/metal.png"
alt="demo app" 
width="300"
/>
</p>

Apple decided to solve all GPU-related demands by announcing a powerful new unified, low-level, low-overhead GPU programming API called Metal. It’s unified because it applies to 3D graphics and data-parallel computation paradigms. Metal is a low-level API because it provides programmers near-direct access to the GPU. Finally, Metal is a low-overhead API because it allows for greater efficiency in terms of CPU and GPU communication and pre-compiling of resources.

## Example App

To learn how to work with Metal, one needs practice. In this series of articles, following step-by-step instructions, we will write a small image editor app capable of making basic image adjustments.

<p style="text-align:center;">
<img 
src="{{site.baseurl}}/images/introduction-to-metal-compute-part-one/demo-app.png"
alt="demo app" 
width="900"
/>
</p>

Here is our plan:

- Begin with a starter project.
- GPU side: write our image editing kernel.
- CPU side: write encoder for the kernel.
- CPU side: image-to-texture conversion and kernel dispatching.
- Replace UIKit image drawing with Metal.

See you in the next part!

----