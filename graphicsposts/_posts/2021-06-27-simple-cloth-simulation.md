---
title: Simple and Fast Cloth Simulation
tags: [OpenGL, C++]
style: fill
color: secondary
description: Simple cloth simulation
blog: [Graphics]
published: false
permalink: /graphics/:title
---

Object Outlining is the most basic of things to implement if you want to visually represent the player's selection. As easy as it sounds it's a bit tricky to wrap your head around the concept of it. Ofc there are a lot of amazing tutorials out there such as this one by [LearnOpenGL](). I suggest you read this before we continue, because what we will be doing here is try to achieve a different look than the one that was shown in the tutorial, but the basic concept remains same. What we want is the actual practical implementation in games.

So the other day I was playing control and saw this ![](). Hmmm I was like how do I do this? I read the [LearnOpenGL]() tutorial right, so I tried to mimic the exact behaviour, it sounded simple yet crucial to get that working.

The major difference is that inorder to get the oultine of the object through all the objects in the scene, we have to draw the outline at the last and disable depth testing altogether. Simple right? Indeed it is but it can sometimes mess up your entire achitecture, especially if you're batching verts together.
