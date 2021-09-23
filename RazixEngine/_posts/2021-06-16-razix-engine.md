---
title: Razix Engine
tags: [OpenGL, C++]
style: fill
color: warning
description: What is razix engine?
blog: [Razix]
published: true
permalink: /razix/:title
---
![](https://pikachuxxxx.github.io/assets/images/blog/razix/RazixLogo.png)

# Razix Engine

Razix is custom game engine I've been building for past 3 years.

Razix is a High Performance Research Engine for production pipeline with emphasis on experimenting with different rendering techniques. Razix supports Windows, Mac, Linux, PSVita and PS3 systems.

 ---

WARNING: Currently Razix is WIP and the renderer is undergoing major design overhaul hence nothing will make sense

# About
Cross-Platform 2D and 3D engine with multi render API support (OpenGL, Vulkan DirectX 11, GXM, GCM, GNM and GNMX). Supports a wide range of Renders with extreme emphasis on scene optimization and implementing state-of-the art rendering techniques. The engine architecture supports a very educational and optimized design.

View the [Trello Board](https://trello.com/b/yvWKH1Xr/razix-engine) and the [Architecture notes](https://drive.google.com/file/d/1y5ZFf-h02z3cx6WmUzR8giKScvORzmwx/view?usp=sharing)

# Features
- Support for Windows, Linux, macOS, PSVita, PS3 and PS4 systems.
- Support for OpenGL, Vulkan DirectX 11, GXM, GCM, GNM and GNMX.
- 3D audio using OpenAL.
- Rendering 3D models with deferred PBR shading.
- Editor GUI using ImGui.
- Multi Physics engine support.
- 3D physics using PhysX, Bullet and Havok.
- 2D physic using Box2D.
- Basic lua scripting support for entities.
- Extremely detailed profiling using Tracy, RenderDoc and Razor integrated deep into the engine systems.
- Custom Animation and state machine engine and supports Havok Animation system
- Supports GLSL, HLSL and PSSL shading languages to create custom materials
- Supports Hull, Domain, Geometry, Compute shaders for all Platforms
- Asset streaming pipeline and custom asset format
- Future support for Falcor and Render Graph Editor

# Architecture

#### :warning: Still a work in progress

 **For individual module architecture and documentation check the Docs folder or check the individual folders for a detailed description (ex. [Core Systems](./Docs/Architecture/RazixEngine-CoreSystems.png))**

<p float="left">
  <img src="https://github.com/Pikachuxxxx/Razix/blob/master/Docs/Architecture/RazixEngine-Architecture.png" width="500" />
  <img src="https://github.com/Pikachuxxxx/Razix/blob/master/Docs/Architecture/RazixEngine-CoreSystems.png" width="400" />
</p>

[Click to view changelog](https://github.com/Pikachuxxxx/Razix/blob/master/Docs/CHANGELOG.md)

[Click to view ReleaseNotes](./Docs/ReleaseNotes.md)
