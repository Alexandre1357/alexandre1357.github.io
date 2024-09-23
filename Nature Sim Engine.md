---
layout: default
title: Nature Sim Engine
---
# Procedural Placement of Props
For our nature sim we were going to have to place a lot of trees, flowers and plants. We didn't want to do this manually for large scenes so I worked on a tool to place props procedurally. The first step to this was implementing blue noise for placement of props on a 2D plane. I wrote the first implementation but it had some problems and was rewritten by my teammate. The next part was done by me which was getting the proper placement of the props on the z-axis as the terrain was generated using a height map. We also wanted to be able to use a density map to affect where the props would spawn. Since I had to read from two textures for possibly thousands of props I decided to do it in a compute shader. I had also written compute shaders before so it wouldn't be to much of a hassle to get it working.
![prop placement](assets/PropPlacementExample.gif)
As you can see above the height of the terrain affects the trees placement on the z-axis. The trees match the slope of the terrain so they aren't all growing straight up. I approximate the terrain normal by picking three points in a radius on the terrain and doing a cross product.
# Vertex Displacement
We wanted the trees and flowers to have some movement to them to make it look wind was blowing through them. I implemented vertex displacement and a simple algorithm to generate the vertex data. The implementation is a simpler version of what is shown off in the GDC talk on [Interactive Wind and Vegetation in ‘God of War’](https://www.youtube.com/watch?v=MKX45_riWQA)
![vertex displacement](assets/Displacement.gif)
The white debug shader displays how much the vertex is affected by wind. 

# Jolt

# Camera / Player Movement

# Menu UI

