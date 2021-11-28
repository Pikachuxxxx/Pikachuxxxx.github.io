---
layout: post
title: Virtual File System
tags: [C++]
style: fill
color: warning
description: A necessary mount
blog: [Graphics]
published: true
permalink: /graphics/:title
---

# Virtual File System : A necessary "mount"

So let's talk about files! Where are they? I need to find this one urgently.

![](https://pikachuxxxx.github.io/assets/images/blog/pulp_fiction.gif)

```
FILE loveLetter =  *fopen("somewhereHidden/loveLetterToBossCauseIdontWantTobeFired.txt", "r+")
```
Ahhh.....Found it! Imagine I missed that...

Well....only if working with file systems were that simple. Jokes apart file management can get quite messy before one realizes, before you say anything about `std::filesystem` let me stop you right there. While `std::filesystem` provides with a huge library of methods it's much simpler and gives you finer control to design your own virtual file system to manage relative path resolution than use something you (or rather ME) don't quite fully understand like the standard library implementation.

Everyone loves relative paths and only if there was something that could do it so easily, that's what we will be doing today build a Virtual File System that does just that. It can mount paths and use those to resolve the final absolute path automatically.

## Building a simple File system

Yes you heard it right first we need a good way to manage the files whose absolute path is known, you can go ahead and create a few functions to check the following

- FileExists
- FolderExists
- GetFileSize
- Read/Write File

Yes! you can use std::filesysytem or use `C`'s `fopen` or any of the `C++`'s `std::fstream` functions but I used the native file system functions using Win32 API for windows because I plan to port my [Razix Engine](https://github.com/Pikachuxxxx/Razix) to different platforms such as consoles and I want to use dedicated IO functions for faster access and do other optimizations but you're free to use any. I won't go into the details of implementing these basic functions so you're free to do it however you want.

Checkout [Razix Engine's FileSystem Interface](https://github.com/Pikachuxxxx/Razix/blob/master/Engine/src/Razix/Core/OS/RZVirtualFileSystem.h) and [Implementation using Win32 API](https://github.com/Pikachuxxxx/Razix/blob/master/Engine/src/Razix/Platform/Windows/WindowsFileSystem.cpp) for reference.

Now that we can open files and do what ever we want with them, let's get virtual.

## Why use virtual paths?

Relative paths make your life easier, especially if you're building a Game Engine like me called [Razix Engine](https://github.com/Pikachuxxxx/Razix) accessing textures, models and other resources are hell of a lot easier, especially when the engine needs to load them during runtime or you're moving assets around to different locations while working on the project. It can easily keep track of the relative path changes and makes moving, copying and other operation hassle-free and not break anything.

## Let's get Virtual

The magic behind the Virtual File System is the power of a `std::unordered_map`. We use a keyword to store the full absolute path to a directory

For example let's say My engine loads some textures from a location like `C:/Dev/Game Engines/Razix/Sandbox/Assets/Textures` which has the file `RazixLogo.png`.

I can store them as a key value pair

 `{"Textures", "C:/Dev/Game Engines/Razix/Sandbox/Assets/Textures"}`

onto a map and use the Key `"Textures"` to get where that folder is, mow everytime I want to get the path to a file inside the `Textures` folder I can use it as a key to get the absoltue path and append my filename for it to get the complete final path.


**So this what I do first mount the absolute path to the directories from which I will be loading some files from by storing those absolute directory paths with a fancy and easy to use keyword in a map as key-value pair**

For example,

```
RZVirtualFileSystem::Get().mount("Assets",      m_AppFilePath + std::string("Assets"));
RZVirtualFileSystem::Get().mount("Meshes",      m_AppFilePath + std::string("Assets/Meshes"));
RZVirtualFileSystem::Get().mount("Scenes",      m_AppFilePath + std::string("Assets/Scenes"));
RZVirtualFileSystem::Get().mount("Scripts",     m_AppFilePath + std::string("Assets/Scripts"));
RZVirtualFileSystem::Get().mount("Sounds",      m_AppFilePath + std::string("Assets/Sounds"));
RZVirtualFileSystem::Get().mount("Textures",    m_AppFilePath + std::string("Assets/Textures"));
```
Where `m_AppFilePath` is the full absolute path to where the folders are located, i.e.
`m_AppFilePath` = `C:\Dev\Game Engines\Razix\Sandbox`

What happens behgind is
```
    m_MountPoints[virtualPath].push_back(physicalPath);
```
where,
`std::unordered_map<std::string, std::string> m_MountPoints;`
is a map of a key and the actual absolute path

## How is the relative aspect of this makes sense you ask?

It's simple all I need to do is to when I'm trying to load a file from a mounted location like Textures
 I need to say that it's relative path and not absolute, so you can use a token before the start of a path to the VirtualFileSystem that it's relative, I use `//` to denote that it's a relative path.

```
 std::string physicalPath;
     if (!RZVirtualFileSystem::Get().resolvePhysicalPath("//Textures/RazixLogo.png", physicalPath))
         return nullptr;
```

Now what happens is the VirtualFileSystem reads the path and see `//` (you're free to use any token to identify) and knows that's it's a relative path, it used the delimiter '/' to separate the keyword and rest of the relative path into 2 parts

It uses the keyword Textures to search the map for the absolute path to that directory and attaches the rest of relative path to that and produces the final complete absolute path that can be used by any file system to open a file.

Simple now isn't it?


### Some precautions

- Use a proper convention with file paths use either of `\\` or `/`` or `\` for all paths, do some processing checks to maintain consistency
- Use a proper token to identify the relative path, the VFS can also open absolute paths if given by directly just use the path as is
- Break the keyword and rest of the relative path using proper delimiters as per your path convention to consistency and avoid errors
