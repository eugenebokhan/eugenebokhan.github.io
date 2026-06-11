+++
title = "Resume"
description = ""
template = "prose.html"
insert_anchor_links = "none"

[extra]
lang = 'en'
math = false
mermaid = false
copy = false
comment = false
+++

## Overview

I'm a software engineer with experience in working with iOS, Computer Graphics and Machine Learning Integration and a passion for creating impactful applications for users. I'm obsessed with solving challenging and interesting problems.

## Skills

- Rust, Metal, Vulkan, Swift, iOS
- Teamwork, communication, collaboration, project ownership, responsibility

## Professional Status

Holder of a UK Global Talent Visa, authorizing work in the United Kingdom.

## Education

### Belarusian National Technical University | Minsk, Belarus

2012 – 2017

- Relay Protection and Automation of Electrical Power Systems.

### Academy of Postgraduate Education | Minsk, Belarus

2015 – 2017

- Professional Communication (English).

## Experience

### Principal Software Engineer, Mirai

[https://trymirai.com](https://trymirai.com)

Dec 2024 – Present · London, England

At [Mirai](https://trymirai.com), I'm responsible for the design and development of [uzu](https://github.com/trymirai/uzu) — our open-source, high-performance on-device [LLM](https://en.wikipedia.org/wiki/Large_language_model) inference engine written in [Rust](https://www.rust-lang.org) and [Metal](https://developer.apple.com/metal) — leveraging my experience with low-level optimization, GPU programming, and ML.

- **GPU Compute & Matmul Engine**: designed and built the [Metal](https://developer.apple.com/metal) compute backend at the core of the engine, including [GEMM](https://en.wikipedia.org/wiki/Matrix_multiplication) / GEMV [matrix-multiplication](https://en.wikipedia.org/wiki/Matrix_multiplication) kernels with split-K reduction, [simdgroup-matrix](https://developer.apple.com/metal/Metal-Shading-Language-Specification.pdf) (MXU) acceleration, and tuned tiling and vectorized-loading heuristics for both prefill and decode.

- **Low-Bit Quantization**: implemented quantized matmul (QMM) supporting 4- and 8-bit weights with scale-bias, scale-zero-point, and symmetric [quantization](https://en.wikipedia.org/wiki/Quantization_(signal_processing)) schemes, and integrated the [Random Hadamard Transform](https://en.wikipedia.org/wiki/Hadamard_transform) (RHT) for accuracy-preserving low-bit inference.

- **Kernel DSL & Compute-Graph Generalization**: co-designed a build-time DSL a successor of [mtlswift](https://github.com/eugenebokhan/mtlswift) that code-generates and specializes [Metal](https://developer.apple.com/metal) kernels and generalized the engine's encodable compute blocks.

- **Profiling & Benchmarking**: built GPU tracing and Metal-analysis tooling along with a cross-device benchmarking harness, profiling and tuning token throughput (prefill / decode) across [Apple silicon](https://en.wikipedia.org/wiki/Apple_silicon) (M1–M4, A18 Pro) and iOS.

### Principal Software Engineer, ZERO10

[https://zero10.ar](https://zero10.ar)

Aug 2021 – Dec 2024 · London, England

At [ZER010](https://zero10.ar), I used my experience in [Swift](https://developer.apple.com/swift) and [Metal](https://developer.apple.com/metal) programming to work on the core of the real-time digital try-on [technology](https://zero10.ar/tech) of the [ZERO10 app](https://apps.apple.com/us/app/zero10-ar-fashion-platform/id1580413828).

- **Rendering Engine Development**: led development of the [Metal](https://developer.apple.com/metal) rendering engine with the following features: [PBR](https://en.wikipedia.org/wiki/Physically_based_rendering), [MSAA](https://en.wikipedia.org/wiki/Multisample_anti-aliasing), [OIT](https://en.wikipedia.org/wiki/Order-independent_transparency), [SSAO](https://en.wikipedia.org/wiki/Screen_space_ambient_occlusion), [Tessellation](https://en.wikipedia.org/wiki/Tessellation_(computer_graphics)), [Inpainting](https://en.wikipedia.org/wiki/Inpainting), Video Textures, [SceneKit](https://developer.apple.com/scenekit)-like shader modifiers.

- **Cloth Physics Engine Development**: played a crucial role in developing [Metal](https://developer.apple.com/metal) cloth [physics simulation](https://zero10.ar/tech#physics-simulation) that runs on the iPhone's real-time.

- **Core App Performance Optimization**: utilized [IOSurfaces](https://developer.apple.com/documentation/iosurface), page-aligned shared memory, [Metal Argument Buffers](https://developer.apple.com/documentation/metal/buffers/improving_cpu_performance_by_using_argument_buffers), [Tile Memory](https://developer.apple.com/documentation/metal/tailor_your_apps_for_apple_gpus_and_tile-based_deferred_rendering#), [Matrix Decomposition](https://en.wikipedia.org/wiki/Matrix_decomposition) and other instruments to significantly increase the try-on pipeline performance and solve the overheating problem.

- **Machine Learning Integration**: worked in a tight collaboration with RnD team and led the body tracking and segmentation models integration: converted [Pytorch](https://pytorch.org) models to [CoreML](https://developer.apple.com/machine-learning/core-ml), profiled the models performance, suggested replacing certain layers to be more [Neural Engine](https://en.wikipedia.org/wiki/Neural_Engine) friendly, suggested switching to [MLMultiArray](https://developer.apple.com/documentation/coreml/mlmultiarray) input / output to reduce memory traffic.

- **Tooling**: developed and maintained a set of internal tools for the ML team, 3D designers as well as marketing team, which led to an increase in the speed of delivering new garments, app improvements, and testing: Internal Garments Design Tool, Photo Try-On & Video Try-On Tools, CLI Tools.

### Software Engineer, Prisma Labs

[https://prisma-ai.com](https://prisma-ai.com)

May 2018 – Aug 2021 · Moscow, Russia

At [Prisma](https://prisma-ai.com), my role was to make a synergistic collaboration with both UI and RnD teams to solve lots of image processing-related tasks while working on [Lensa](https://apps.apple.com/us/app/lensa-photo-picture-editor/id1436732536) and [Prisma](https://apps.apple.com/us/app/prisma-photo-editor/id1122649984) apps.

- **Image Processing Pipeline Development**: led the development of the [Metal](https://developer.apple.com/metal) image processing and rendering engine, which included Face Retouching, Inpainting, Face-Aware Morphing, DOF Effects, Background Replacement, LUTs, Node-Based Filters and Effects, etc.

- **Core App Performance Optimization**: utilized [Metal](https://developer.apple.com/metal), [ARM NEON](https://developer.arm.com/Architectures/Neon), [Accelerate](https://developer.apple.com/documentation/accelerate), Cache-Friendly Memory Layout, etc., to significantly enhance the efficiency and performance of the core app functionalities. Implemented the real-time [Frequency Separation Retouch](https://www.adobe.com/products/photoshop/frequency-separation.html) that worked on the phone w/o overheating.

- **Machine Learning Integration**: developed and maintained [Smelter](https://github.com/prisma-ai/Smelter) - [ONNX](https://onnx.ai) inference engine backed by [Metal Performance Shaders](https://developer.apple.com/documentation/metalperformanceshaders) that allowed to avoid redundant CPU - GPU syncs. Worked in collaboration with the RnD team to achieve smooth ML integration and suggested switching to [YUV color space](https://en.wikipedia.org/wiki/Y′UV) to improve the performance.

- **Tooling**: developed and maintained internal tools for the ML and designers teams, which increased the speed of delivering new image effects: [Node Graph Visual Editing Tool](https://x.com/eugenebokhan/status/1372199083943849984), CLI Tools.

### Software Engineer, Taqtile

[https://taqtile.com](https://taqtile.com)

Feb 2017 – May 2018 · Minsk, Belarus

At [Taqtile](https://taqtile.com), I was responsible for the design and implementation of the [iOS version](https://taqtile.com/ipad) of the company's main product, Manifest.

- **AR Experience Development**: led the development of the AR experience using [ARKit](https://developer.apple.com/augmented-reality/arkit/), [SceneKit](https://developer.apple.com/scenekit/), and [Metal](https://developer.apple.com/metal)

- **Tooling and Prototyping**: led the research and development in the field of AR and 3D (3D Model Optimisation, ARKit + Geolocation, Pointcloud Mesh Reconstruction).

### Open Source Projects

[https://github.com/eugenebokhan](https://github.com/eugenebokhan)

#### [uzu](https://github.com/trymirai/uzu)

A high-performance on-device [LLM](https://en.wikipedia.org/wiki/Large_language_model) inference engine written in [Rust](https://www.rust-lang.org) and [Metal](https://developer.apple.com/metal), built at [Mirai](https://trymirai.com) to run models locally on [Apple silicon](https://en.wikipedia.org/wiki/Apple_silicon) with zero latency and full data privacy. I built its [Metal](https://developer.apple.com/metal) compute backend — quantized matmul kernels, the kernel DSL, and the benchmarking tooling.

#### [Metal Analyzer](https://github.com/computer-graphics-tools/metal-analyzer)

A [Metal Shading Language](https://developer.apple.com/metal) language server (LSP) for VS Code, Cursor, Zed, and IntelliJ, offering real-time diagnostics via `xcrun metal`, auto-completion, hover documentation, and [clang-format](https://clang.llvm.org/docs/ClangFormat.html)-based formatting.

#### [ANE](https://github.com/computer-graphics-tools/ane)

Rust bindings for the [Apple Neural Engine](https://en.wikipedia.org/wiki/Neural_Engine) via the private `AppleNeuralEngine.framework`, providing a graph builder and compilation pipeline with zero-copy [IOSurface](https://developer.apple.com/documentation/iosurface)-backed I/O.

#### [mtl-rs](https://github.com/computer-graphics-tools/mtl-rs)

Rust bindings for Apple's [Metal](https://developer.apple.com/metal) API, built on the modern [objc2](https://github.com/madsmtm/objc2) ecosystem with coverage of the latest [Metal 4](https://developer.apple.com/metal) API.

#### [mpsgraph-rs](https://github.com/computer-graphics-tools/mpsgraph-rs)

Modern Rust bindings for Apple's [Metal Performance Shaders Graph](https://developer.apple.com/documentation/metalperformanceshadersgraph) framework, exposing a high-level, type-safe graph API for defining and running neural networks on the GPU.

#### [Metal Tools](https://github.com/computer-graphics-tools/metal-tools)

MetalTools provides a convenient, Swifty way of working with [Metal](https://developer.apple.com/metal). This library contains a lot of [read-to-use compute kernels](https://github.com/computer-graphics-tools/metal-tools/tree/main/Sources/MetalComputeTools/Kernels) and is heavily used in computer vision startups like [Prisma](https://prisma-ai.com) and [ZERO10](https://zero10.ar).

#### [Shared Graphics Tools](https://github.com/computer-graphics-tools/shared-graphics-tools)

A set of tools and extensions that allow sharing page-aligned memory allocation between different graphics APIs and provide view interop for them.

#### [CoreVideoTools](https://github.com/computer-graphics-tools/core-video-tools)

CoreVideoTools offers a more idiomatic Swift interface to [CoreVideo](https://developer.apple.com/documentation/corevideo) functionality, making it easier and safer to work with [CVPixelBuffers](https://developer.apple.com/documentation/corevideo/cvpixelbuffer-q2e), [IOSurfaces](https://developer.apple.com/documentation/iosurface), and related CoreVideo concepts in Swift code.

#### [Smelter](https://github.com/prisma-ai/Smelter)

Overhead-free [ONNX](https://onnx.ai) graph inference engine with [Metal Performance Shaders](https://developer.apple.com/documentation/metalperformanceshaders) under the hood.

#### [WGPUTools](https://github.com/computer-graphics-tools/wgpu-tools)

A Rust library providing utility functions and abstractions for working with [WGPU](https://wgpu.rs).

## Community Contribution

#### [Introduction To Metal Compute](https://eugenebokhan.github.io/blog)

A [set of tutorials](https://eugenebokhan.github.io/blog) describing how to build an [image-processing Metal iOS app](https://github.com/eugenebokhan/introduction-to-metal-compute) from scratch.
