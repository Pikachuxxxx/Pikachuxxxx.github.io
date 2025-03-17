---
title: Linux Graphics Stack - An Overview
tags: [Graphics, Linux]
style: fill
color: warning
description: Understanding the different layers in the linux gfx stack
blog: [Graphics]
published: true
permalink: /graphics/:title
---

Hi all, So a lot has happened in recent times, I got a new job at Qualcomm. Yayy! I suppose. I will be working on something rather out of my comfort zone, maybe too much out of it.
I will be working on a bit of GPU drivers, primarily with User Mode Drivers (UMDs) and the linux graphics stack and drop into the unknown territory like compositors and using mesa drivers etc. So I thought I should make a little self-learning post on how various libraries are used and how the linux graphics stack looks from a high-level persepctive. All this new stuff is very fun and exciting to learn especially the Adreno GPUs. Also one more thing... I'm finally moving back to my hometown Hyderabad, yeah I know I will be missing Bangalore a lot, especially the people and my Sony ex-colleagues and the amazing firnedships I made over the last few years. Idk I felt like I needed some change to really push myself and I think it's a nice memory/experience to get to work in your hometown and ofc all things aside the work seems hella lot of fun so I finally decied to pull the trigger to move here for a while. Let's see how this chapter goes. Now, shall we dive in?

## Overview
The linux graphics stack is very complex and for someone like me coming from a API level it can seem very daunting but it get's acquaintable as you spend more time with and take time to understand what each layer does and what different libs and technologies are available as alternatives. Let's go through each later from the App to GPU HW and see what the stack contains.

## 1. App Level
At the very top of the stack we have is the applications we run, they can be the games we play, browsers for "Netflix and Chill" and GUI Apps like calculators, Blender etc. They use components like Graphics APIs (Opengl, Vulkan etc.), GUIs libraries (GTK, QT etc.) to render.

## 2. Desktop Shell/UI
One level down we have desktop environments like GNOME, KDE, XFCE, and others (depending on your Linux distribution). These environments provide the user interface shell, including panels, docks, and application launchers. They typically use window managers to handle the positioning, resizing, and stacking of application windows.

## 3. Display Servers, Compositors, and WSI

**Display servers:** They handle communication between applications and the display hardware. They provide a protocol for Window System Integration (WSI) that allows multiple windows to be managed and displayed. The two major display server protocols in Linux are:

- **X11/Xorg:** The traditional display server, which follows a client-server architecture. Applications connect to the X server to draw their windows.
- **Wayland:** A newer protocol designed to replace X11, offering better security, performance, and simplified architecture.

**Compositors:** combine the contents of multiple windows into a final image for display. They handle effects like transparency, animations, and window transitions. Some notable examples:

Libraries that facilitate this communication include:
- XCB/Xlib: Client libraries for X11
- libwayland-client: Client library for Wayland
- EGL: Provides the interface between rendering APIs and the native window system

**Misc:**
- Mutter: Used by GNOME Shell, supports both X11 and Wayland
- KWin: Used by KDE Plasma, supports both X11 and Wayland
- Sway: A tiling compositor for Wayland, similar to the i3 window manager
- XWayland: A compatibility layer allowing X11 applications to run on Wayland

## 4. Graphics APIs 
This layer includes the 3D accelerated graphics APIs:

- **OpenGL** (libGL.so): The traditional 3D graphics API
- **Vulkan** (libvulkan.so): A modern, low-level graphics and compute API
- **OpenGL ES** (libGLESv2.so): A version of OpenGL designed for embedded systems
- **EGL** (libEGL.so): An interface between rendering APIs and the native window system

These libraries are just wrappers that call into the User Mode Driver (UMD) implementations.

## 5. UMD - User Mode Drivers
This is the user space driver component. 
- The main role of UMD is to implement the Graphics API spec. 
- It translates the API calls into GPU HW specific commands via kernel calls into the KMD via **ioctl** calls
- Shader compilation and optimizations
- Manages allocating buffers/memory via GBM if needed. 
- It is responsible for generating the PM4 (programmable model 4) packets, which encodes GPU commands for shaders, memory access, and rendering operations.

UMD implementations include:
- **Mesa:** An open-source implementation supporting various GPUs
- **Proprietary drivers:** From vendors like NVIDIA, AMD, and Qualcomm

_Note: Mesa uses Gallium3D as an abstraction layer for most drivers, Exception: Turnip (Qualcomm Vulkan driver) is written from scratch_

## 6. KMD - Kernal Mode Drivers
This is the kernel space driver component manages direct GPU HW access.
- It manages the HW initialization and config and info
- KMD writes the PM4 packets to the GPU HW ring buffer for submimssion
- It manages command submisstion and their execution and low-level synchronization events and primitives
- Manages the VRAM memory subsystem and exposes to be used in UMD via GBM and ioctl calls
- It handles the work scheduling and context switching mechanisms
- It handles GPU interrupts and page faults handling
- Manages the GPU power and exposes **perf-counters** for debugging

#### DRM KMS and GBM
The Direct Rendering Manager (DRM) is a kernel subsystem that manages access to GPU hardware. 

Kernel Mode Setting (KMS) is a mechanism that allows the kernel to set display modes (resolution, color depth, refresh rate).

The Generic Buffer Manager (GBM) is an API that provides a mechanism for allocating buffers for graphics rendering. It works with DRM/KMS to manage display buffers and facilitate zero-copy rendering paths.

## 7. GPU HW
Last but not the least we have the GPU hardware itself with their own ISA as per the vendor.

- AMD: RDNA and GCN (older GPUs and current gen consoles)
- NVIDIA: Ampere, Turing, Blackwell etc.
- Intel: Gen graphics, Xe
- Qualcomm: Adreno
- ARM: Mali
- Apple: M series 

## Closing Notes
I hope this gives a very high-level insights into how different linux graphics components intreract to render the beautiful pixels we see. Also stay tuned for the wild stories from hyderabad diaries. I will try to slip in some life updates from time to time. 

Until they see ya!


## References
1. [The DRM/KMS subsystem from a newbie's point of view](https://events.static.linuxfound.org/sites/events/files/slides/brezillon-drm-kms.pdf)
2. [Wayland Architecture](https://wayland.freedesktop.org/architecture.html)
3. [Mesa3D Documentation](https://docs.mesa3d.org/)
4. [Vulkan WSI (Window System Integration) Extensions](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/vkspec.html#wsi)
5. [Intro to Linux Graphics Stack](https://flusp.ime.usp.br/blogs,/kernel-graphics/an_introduction_to_the_linux_graphics_stack/)
6. [X Window System core protocol Wikipedia](https://en.wikipedia.org/wiki/X_Window_System_core_protocol)

