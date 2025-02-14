---
title: Devlog 2 - Voronoi textures using roots of unity
tags: [Math]
style: fill
color: success
description: generating phase-offset polygons 
blog: [Graphics]
published: true
permalink: /graphics/:title
---
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script> 


Hi all, Happy Valentined Day guys! ❤️ to all the hearbreaks...I hope they heal someday. 

Yes, today our goal is to generate a weird looking but some what acceptable voronoi texture.

## Scaling the phase offset polygons 
Phase varitions shifts the roots and generates polygons with irregular sides inside a circle of unit radius.
 
Take this python code:

```python

import numpy as np
import matplotlib.pyplot as plt
from scipy.spatial import Voronoi, voronoi_plot_2d

def generate_randomized_polygon(n, offset_range=0.1, randomize=True, scale=1.0):
    """
    Generates a polygon based on the n-th roots of unity with optional random offsets.
    Scales the resulting polygon.

    Parameters:
        n (int): Number of sides of the polygon.
        offset_range (float): Maximum offset for randomizing the angle (radians).
        randomize (bool): If True, randomize angles; otherwise, keep a perfect polygon.
        scale (float): Scaling factor for the polygon.

    Returns:
        np.ndarray: x and y coordinates of the vertices.
    """
    angles = np.linspace(0, 2 * np.pi, n, endpoint=False)

    if randomize:
        random_offsets = np.random.uniform(-offset_range, offset_range, n)
        angles += random_offsets

    points = np.exp(1j * angles) * scale
    return points.real, points.imag

def create_voronoi_texture_in_rectangle(rows, cols, n_sides, offset_range, scale=1.0):
    """
    Creates a Voronoi texture with randomized polygons inside a large rectangular bounding box.

    Parameters:
        rows (int): Number of rows in the grid.
        cols (int): Number of columns in the grid.
        n_sides (int): Number of sides for the polygons.
        offset_range (float): Randomization range for polygon vertices.
        scale (float): Scale of the polygons.

    Returns:
        None
    """
    centers = []
    all_vertices = []

    # Generate polygons and collect their centers
    for row in range(rows):
        for col in range(cols):
            x_offset, y_offset = col * 1.2, row * 1.2
            x, y = generate_randomized_polygon(n_sides, offset_range, scale=scale)
            x += x_offset
            y += y_offset
            all_vertices.append((x, y))
            centers.append((x_offset, y_offset))

    # Convert centers to an array for Voronoi computation
    centers = np.array(centers)

    # Generate Voronoi diagram
    vor = Voronoi(centers)

    # Overlay polygons
    for vertices in all_vertices:
        x, y = vertices
        plt.fill(x, y, alpha=0.5)

    # Set rectangle bounds
    plt.xlim(-1, cols * 1.2 + 1)
    plt.ylim(-1, rows * 1.2 + 1)

    # Remove gridlines and axes
    plt.gca().set_aspect('equal', adjustable='box')
    plt.axis('off')  # Completely remove axes and gridlines

    plt.title("Voronoi Texture with Randomized Polygons in a Rectangle")
    plt.show()

# Parameters
rows, cols = 5, 7   # Grid size
n_sides = 6          # Number of sides for each polygon
offset_range = 0.6   # Randomization range
scale = 0.6          # Scale for polygons

create_voronoi_texture_in_rectangle(rows, cols, n_sides, offset_range, scale)

```
This will generate the output as below: 

![](https://pikachuxxxx.github.io/assets/images/blog/voronoi/voronoi_tiled.png)

Noice we are getting forward, this post is just to show some progress and I'm hoping to get a poster ready for HPC 2025 conference. Let's see what happens in the next devlog.

See you soon.


