---
title: Devlog 1 - Voronoi textures using roots of unity
tags: [Math]
style: fill
color: success
description: soul-crushing devlogs on generative Voronoi textures using roots of unity
blog: [Graphics]
published: true
permalink: /graphics/:title
---

The other day I went to meet a friend for lunch. We got a little drunk with some Red Bull and Old Monk(_Old Monk_ is an Indian-made rum. If you're wondering what that is, a MUST TRY! with masala coke.) and then we started talking about math. He started explaining about roots of unity all of a sudden and started showing me [the proof to how roots of unity form polygons within a unit circle](https://proofwiki.org/wiki/Complex_Roots_of_Unity_are_Vertices_of_Regular_Polygon_Inscribed_in_Circle#google_vignette).

![](https://pikachuxxxx.github.io/assets/images/blog/voronoi/oldmonk.jpeg)

It was really fun to talk about math, and then on my way home I had a completely random and weird epiphany.

> _so Voronoi textures are made up of cells at the primitive level. What if, we use the polygons formed by roots of unity to create the building blocks as those cells? and tile the texture? We can randomise the phase offsets and create roots and generate cells for a truly random Voronoi texture. This method not only helps us generate Voronoi textures quickly and uniquely but also offers control over a lot of parameters. In theory, we should be able to control the polygon size, root phase offsets, polygon sides, side lengths, etc. So much control seems very tempting._

![](https://pikachuxxxx.github.io/assets/images/blog/voronoi/voronoi.png)

This series of devlogs is an attempt to create Voronoi textures quickly and efficiently using the roots of unity. We slowly generate each building block as a lemma and build on that to achieve our goal. 

At the end, we can also try to analyze the properties of such textures generated and see how we can control different parameters to create Voronoi textures with varying properties (ex. all cells have the same number of sides, or the same radius, etc.). 


## The Journey Begins

---

### Lemma 1: Structured Distribution of Points Using 𝑛-th Roots of Unity

> _The 𝑛-th roots of unity create a symmetrically distributed
set of points on the unit circle in the complex plane, forming the
fundamental building blocks of this Voronoi-based texture generation
algorithm._

**Proof:**
The 𝑛-th roots of unity are complex numbers
𝜔𝑘 = 𝑒<sup>2𝜋𝑖𝑘/𝑛</sup> , with 𝑘= 0, 1, . . . , 𝑛−1. Since each 𝜔𝑘 has unit magni-
tude (|𝜔𝑘 |= 1), these points are evenly spaced along the unit circle,
with an angular separation of 2𝜋/𝑛 between consecutive points.
This spacing ensures that the points are symmetrically distributed
around the origin.

**Implications:**
This initial structure, by using the 𝑛-th roots of unity
as seeds, the algorithm can quickly generate a well-structured and balanced set of points in 𝑂(𝑛) time,
serving as the core framework for Voronoi cell generation.

Take this Python code for example to generate some of such polygons:

```python

import matplotlib.pyplot as plt
import numpy as np

# Number of sides (vertices) of the polygon
n = 5  # for a pentagon, change this to any integer >= 3

# Calculate the n-th roots of unity
vertices = [np.exp(2j * np.pi * k / n) for k in range(n)]

# Plot the points
plt.figure(figsize=(6, 6))
plt.plot([v.real for v in vertices], [v.imag for v in vertices], 'o', markersize=8)
plt.plot([v.real for v in vertices] + [vertices[0].real],
         [v.imag for v in vertices] + [vertices[0].imag], 'b-')  # connect the vertices

# Add some style
plt.axhline(0, color='grey', lw=0.5)
plt.axvline(0, color='grey', lw=0.5)
plt.gca().set_aspect('equal', adjustable='box')
plt.title(f"{n}-gon using {n}-th roots of unity")
plt.show()

```
This will generate the output as below: 

![](https://pikachuxxxx.github.io/assets/images/blog/voronoi/voronoi_cell_pentagon.png)

If you can see the pattern, these can be used to tile once the polygon has enough parameters to randomise the phase-offsets.

### Next Goals
Now that we have the basic building blocks, the next step is to introduce random offsets into the phase to shift the roots and create irregular polygons. 

Until then, bye.

