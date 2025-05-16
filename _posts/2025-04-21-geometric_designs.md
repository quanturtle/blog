---
title: "🖼️ Geometric and Artistic Designs with your Micro Computer"
subtitle: "Recursion + Computer Graphics + 3D printing"
date: "2025-04-22"
---

> TL;DR — Translated designs from BASIC to modern python and then 3D printed them. All code for this project can be found in my repository: Dessins Geometriques et Artistiques.

Tangents lead to tangents that lead to further tangents and then I found v3ga’s repository.

That’s how I read about Jean-Paul Delahaye and his book Dessins Géométriques et Artistiques avec votre micro-ordinateur (1985).

![](/img/geometric_designs/geometric_designs_01.jpg)

I immediately fell in love with his work and how he was using the technology available at the time to bring this equations into the real world. Namely Microsoft’s BASIC programming language and a Canon X-07 with a X-710 plotter.

Have a look:

[![Video Title](https://img.youtube.com/vi/JWhNcsYoXQ0/0.jpg)](https://www.youtube.com/watch?v=JWhNcsYoXQ0)

His designs, scanned from an original edition, have that distinctive hue of yellow that you get from an old book.

![](/img/geometric_designs/geometric_designs_02.jpg)

![](/img/geometric_designs/geometric_designs_03.jpg)

![](/img/geometric_designs/geometric_designs_04.jpg)

![](/img/geometric_designs/geometric_designs_05.jpg)

![](/img/geometric_designs/geometric_designs_06.jpg)

![](/img/geometric_designs/geometric_designs_07.jpg)

## How can I recreate these designs?
The book provides clear instructions on how to set up everything we need to draw:

![](/img/geometric_designs/geometric_designs_08.jpg)

![](/img/geometric_designs/geometric_designs_09.jpg)

![](/img/geometric_designs/geometric_designs_10.jpg)

Important things:

* Program structure
    * ligne 10: indicates de name of the program
    * ligne 50: variable initialization
    * Rest of the program (generally, there is a drawing loop)
* Drawing windows
    * Generally, programs use a drawing window of 480px (NP=480)
    * Values for X and Y are constrained:
        * 0 < X < NP
        * 0 < Y < NP
* If you were using a different printer, there are instructions on how to translate those commands to different hardware
* Finally, directions to modify the programs so users can create their own designs are provided.

## No French? No BASIC?
v3ga used p5.js to code the designs again in modern javascript. I don’t know javascript so I opted for python and specifically its Turtle module. Turtle allows you to move a cursor and draw in the same way the Canon X-07 could, so we give it a try:

![](/img/geometric_designs/geometric_designs_11.jpg)

Import + set up our canvas:
```python
import math
import turtle
from typing import Optional


def setup_canvas(NP: int = 480):
    """
    Sets a window of size NP x NP in 'turtle' coordinates.
    """
    turtle.setup(width=NP, height=NP)
    turtle.setworldcoordinates(0, 0, NP, NP)
    turtle.speed("fastest")
    
    turtle.penup()
    turtle.goto(0, 0)
    turtle.pendown()
```

Translate and compose a drawing function:
```python
def draw_star_at_center(
    K: int, 
    H: int, 
    center_x: int, 
    center_y: int, 
    radius: int, 
    angle_offset: float=0.0
):
    """
    Draw a star of K edges, skipping H points, 
    centered at (center_x, center_y) with given radius and angle offset.
    """
    for i in range(K + 1):
        angle = (2 * math.pi * H * i / K) + angle_offset
        x = center_x + radius * math.cos(angle)
        y = center_y + radius * math.sin(angle)
        
        if i == 0:
            turtle.penup()
        else:
            turtle.pendown()
        
        turtle.goto(x, y)
```

Use the drawing function inside a composition:
```python
def composition_1(
    NP: int=480, 
    K1: int=5, 
    R1_ratio: float=0.27, 
    A1: float=math.pi, 
    DX: Optional[int]=None, 
    DY: Optional[int]=None, 
    K: int=25, 
    H: int=12, 
    R_ratio: float=0.22, 
    AD: float=math.pi/2
):
    """
      1) We have K1 equally spaced points around a circle of radius (R1_ratio*NP)
         centered at (DX, DY). The angle offset for that circle is A1.
      2) For each of those K1 points, we draw a star using skip H,
         with K edges, radius = (R_ratio * NP), angle offset = AD.
    """
    if DX is None: 
        DX = NP / 2
    if DY is None: 
        DY = NP / 2
    
    # Loop over the K1 points around a circle
    for i1 in range(K1):
    
        # Compute the center (CX, CY) of the star
        theta = (2 * math.pi * i1 / K1) + A1
        CX = DX + (R1_ratio * NP) * math.cos(theta)
        CY = DY + (R1_ratio * NP) * math.sin(theta)
        
        # Now draw an H-skip star with K edges at (CX, CY)
        draw_star_at_center(
            K=K, H=H, 
            center_x=CX, center_y=CY, 
            radius=R_ratio * NP, 
            angle_offset=AD
        )

    turtle.hideturtle()
    turtle.exitonclick()
```

Run the whole thing:
```python
def main():
    NP=480
    setup_canvas(NP)
    composition_1(
        NP=NP,
        K1=25,
        R1_ratio=0.1,
        A1=math.pi,
        K=5,
        H=2,
        R_ratio=0.4,
        AD=math.pi/2
    )

main()
```

But…but…I don’t have one of *those* printers

I don’t have a X-710 plotter but I have a 3D printer (a BambuLab X1C).

## How can I print something like this?
In this exercise in creative coding, I wanted to take these shapes and make them real.

My programs generate X, Y coordinates so that the turtle head can trace the designs. We could use those same coordinates to describe a 3D shape.

In order to achieve this, I decided to 3D print these designs but for that, I had to learn CAD.

After trying some libraries, I decided to go for [`build123d`](https://github.com/gumyr/build123d):

```python
from typing import List
from build123d import *


def generate_cad(pts: List, name: str):
    with BuildPart() as part:
        with BuildSketch(Plane.XZ) as s:
            with BuildLine() as l:
                l1 = Polyline(*pts)
            
            trace(line_width=3.5)
        
        extrude(amount=10)

    try:
        if name:
            export_stl(part.part, f"{name}.stl")
        
        else:
            export_stl(part.part, "my_design.stl")

    except Exception as err:
        print(f"Error exporting STL: {err}")
        
    return
```

Important things here:
* We use `build123d` `builder_mode` (using context managers to build the object)
* We use `polyline`: we provide a list of coordinates and connect them to generate our shape
* We trace that line so that it has actual `width` (turn the object from 1D to 2D)
* We `extrude` our 2D object over the `Z axis` to have an object with `volume`
* Finally, export the design and you can throw it in your slicer ready to print!

![](/img/geometric_designs/geometric_designs_12.jpg)

![](/img/geometric_designs/geometric_designs_15.jpg)

![](/img/geometric_designs/geometric_designs_16.jpg)

Re-coding Jean-Paul Delahaye’s 1980-era BASIC sketches in modern Python—and then turning those digital tracings into tangible objects—felt like finishing a conversation that began four decades ago.

Along the way we bridged three generations of “maker” tools: early micro-computers, modern programming languages, and today’s rapid-prototyping hardware. What started as a simple port became a reminder that code—no matter how old—can keep expressing itself in new mediums as long as we’re willing to translate it.

![](/img/geometric_designs/geometric_designs_13.jpg)

![](/img/geometric_designs/geometric_designs_14.jpg)

I hope these examples encourage you to dust off other vintage algorithms, give them a modern dialect, and let today’s machines draw them in plastic, resin, or whatever comes next.