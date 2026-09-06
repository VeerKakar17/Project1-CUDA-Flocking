**University of Pennsylvania, CIS 5650: GPU Programming and Architecture,
Project 1 - Flocking**

  Veer Kakar
  * [LinkedIn](www.linkedin.com/in/VeerKakar), [personal website](https://veerkakar17.github.io/PortfolioWebsite)
* Tested on: Linux Fedora 44 (Dual Boot from Windows Laptop), Intel Ultra 9 275HX, NVIDIA 5070 laptop

![](/images/demo_ss.png)
![](/images/Screencast_20260905_235742.gif)

## Overview
The following program is an implementation of a Boids flocking simulation, where each particle represents a single boid and they move with repspect to each other according to the 3 rules of cohesion, separation, and alignment.

There are 3 implementations created here, which are togglable by modifying the defined flags in [main.cpp]. They are as follows.

### Naive
The naive solution is defined by using one GPU thread per each boid. For each boid, they check all other boids for if they are within a specified distance, and then apply the cohesion, seperation, and alignment rules with respect to them.

### Uniform
The uniform solution is defined by spliting up the entire space into a grid. It first determines which grid cell each boid is in, computing a grid index.
This then uses `Thrust` to sort all the boids by this grid index, and determines the start/end index in this sorted array of each grid. By doing this, we can quickly determine which boids are in which grid cells, and then only check boids in nearby grid cells.

The benefit over Naive is that we are now only checking the subset of boids which could potentially be close enough to affect this boid's velocity, as opposed to checking if every single boid is close enough with respect every other boid.

For the size of the grid cells, this uses 2 times the max rule distance between cohesion, separation, and alignment. This allows us to only then check the 8 closest grid cells for boids, as we can guarantee that the other cells will all be outside of this range and will not impact this boid's velocity.

Since we only sort the grid indices, the position and velocity data is still scattered, so this uses a `particleArrayIndices` array to map these sorted indices to the original boid position/velocity indices, and uses this to fetch them.

### Coherent
The coherent implementation is an optimization over the Uniform implementation. This uses the same grid method, but reduces the scatter of the data.

The only difference is that after sorting, this implementation uses thrust's gather operation to order the position and velocity arrays to match the sorted grid indices array. This prevents the need for `particleArrayIndices` and allows us to directly access the data in these arrays, guaranteeing that accessing data for boids in a grid cell is done sequentially in memory.

## Performance Analysis

The following tests were completed using `NVIDIA Nsight Systems` by running the simulation without visualization initially for 25 seconds to stabalize, and then while profiling for 20 seconds. 
I then took the total number of calls to `kernUpdatePos`, a kernel called once every frame in all 3 implementations, and divided this number by the 20 second profiling time to get number of frames per second. With this measure of performance, a higher FPS is better.

The reason I profiled without visualization here is to prevent graphics overhead from introducing slowdowns and to allow FPS to be uncapped past 165 (the refresh rate of my monitor).
These tests were run with visualization on as well, but dur to the scale of the FPS, only the high boid count naive tests went below 165 FPS, and everything else just read at this cap. 

Some additional information is listed below:
- Block size was set to 128 for all tests unless it was explicitly specified otherwise.
- These were all tested with build mode Release

### How does the number of boids affect performance?
![](/images/Boid%20Count%20vs%20Framerate.png)

| Boid Count | Uniform | Coherent | Naive |
| ---------- | ------- | -------- | ----- |
| 1000 | 2659.15 | 2592.9 | 2317.65 |
| 2500 | 2568.933333 | 2571.933333 | 1625.1 |
| 5000 | 2395.6 | 2430.56 | 1074 |
| 10000 | 2274.6 | 2380.75 | 627.9 |
| 25000 | 2129.25 | 2304.95 | 195.9 |
| 50000 | 2111 | 2100.65 | 59.35 |
| 100000 | 1674.55 | 1758.7 | 15.6 |

- The `Naive` implementation's FPS decreased quadratically (proportional to `O(n^2)`) as boid count increased. This was expected as this is comparing every boid against every other boid for every frame, which equates to `O(n^2)` operations.
- The `Uniform` and `Coherent` implementations' FPS's both decreased roughly linearly as boid count increased (proportional to `O(n)`). Due to using the grid method for these, we are greatly reducing the number of boids we need to compare (as we don't comapre a boid against boids outside the 8 closest grid cells which can't be within our rule distances, greatly reducing the number of comparisons needed).
- While at very low boid counts `Uniform` has higher FPS than `Coherent`, as the boid count increases, `Coherent` stabally has higher FPS. This scaling with `Coherent` having higher FPS is expected as we are reordering the data to be all sequential in memory with respect to each grid cell. Sequential memory accesses are more efficient than requiring constant jumps around memory, as done with `Uniform` where we shift the access order but keep the position and velocity arrays unmodified. Due to this, `Coherent` ends up being more optimal as our boid count get large and the disparity caused by constant sequential vs scattered memory accesses causes a difference.

### How does Block Count / Size affect performance?
![](/images/Block%20Size%20to%20Framerate.png)
![](/images/Naive%20(5k)%20vs.%20Block%20Size.png)

| Block Size | Uniform (50k) | Coherent (50k) | Naive (5k) |
| ---------- | ------------- | -------------- | ---------- |
| 32 | 1809 | 1988.45 | 1065.05 |
| 64 | 2085.5 | 2079.45 | 1069.15 |
| 128 | 2111 | 2100.65 | 1074 |
| 256 | 2104.55 | 2096.6 | 1075.7 |
| 512 | 2121.1 | 2071.05 | 1012.05 |

These graphs here are showing the performance change based on block size (number of threads per block). The number of blocks (block count) can be derived by doing `blockCount = ciel(numBoids / blockSize)`.

There was a fixed boid count used here, with 50k for `Uniform` and `Coherent`, and 5k for `Naive`. These were chosen with respect to each implementation to best demonstrate how block count/size affects each one, as we get the best estimate for the grid-based implementations at higher boid counts like 50k, where `Naive` will scale too badly and drop off, giving us a better estimate closer to 5k.

What we can see here is that increasing or decreasing block size does not necessarily always positively or negatively increase performance. For Uniform and Coherent, they both had near optimal performance at `128` threads per block with a decrease as we modify this number in either directions. However, while the rest of the `Uniform` data followed the same trent, it had a spike with the best performance at `512` threads per block.

For Naive, we had an almost constant FPS with a slight increase in performance up to 256, with a decrease in performance at 512. 

### Does the Coherent implementation perform strictly better than Uniform?

The answer here is no. While `Coherent` was seen to scale better and have better performance than `Uniform` as our boid count increased, the same is not true at all boid counts. 

When comparing at 1000 boids and small numbers, `Uniform` actually performed 2.56% better. This is because, while the benefit of sequential memory accesses with `Coherent` is more efficient than scattered memory accesses with `Uniform`, there is still an overhead cost associated with it. For `Coherent`, we now have to gather on both our position and velocity array with respect to the grid-index sorted order. The instances where `Uniform` perform better are because the overhead cost from gathering is larger than the performance gained by these sequential memory accesses, reduing the performace.

The reason this happens more at lower boid counts is because there is less data to compare and less data in each grid, which results in less sequential memory accesses gained with `Coherent` while still having the overhead cost to gather. 

However, at higher boid counts, there are significantly more boids in each grid cell, so there are a lot more sequential memory accesses gained by `Coherent` while the gather overhead cost does not scale as quickly, resulting in our performance increase.

### How does checking all 27 neighboring cells with half the cell width affect performance?
![](/images/8%20vs%2027%20block%20to%20Performance.png)

| Search Method | Uniform | Coherent |
| ------------- | ------- | -------- |
| 8 blocks, width*2 | 2111 | 2100.65 |
| 27 blocks, width*2 | 1531.15 | 1705 |
| 27 blocks, width | 2293.8 | 2292.1 |

This experiment compared the search method for blocks and change in grid cell width against performance. There were 2 variables modified.
Please note that `Naive` is not tested here as it does not use this grid logic.

Grid Cell Width:
- This is the width of each grid cell we are considering, which we split up all boids into.
- The original sets `gridCellWidth = 2.0f * max(rule1Dist, rule2Dist, rule3Dist)`, or in other words, 2 times the max consideration distance for any of the 3 rules.
- For testing with 27 blocks, I set the `gridCellWidth = max(rulel1Dist, rule2Dist, rule3Dist)`, which is the max consideration distance for any of the 3 rules.

Blocks Searched:
- The original method checks the nearest 8 blocks. This works since we set our gridCellWidth is twice the highest rule distance, so we can guarantee that only these 8 grid cells could have boids within this distance. 
  - This is done by looking at the current grid cell this boid is in, and then looking at if it is in the upper or lower half of the cell in all 3 axises (x, y, and z). We then only check the surrounding cells in those directions and composites of those directions.
- The new 27 method instead checks all surrounding grid cells in every direction and checks all boids in all of these. The test here is looking for a performance change by setting our grid cell size to just the max distance, meaning any of these cells could possibly have close enough boids to consider. The tradeoff here is that each cell will now contain less boids due to the smaller size.
  - I also added a test with 27 blocks and an unchanged with as an additional baseline check.

For the results, I found them unexpected at 50,000 boids. What I expected was that the 8 blocks would be more optimal with `Coherent`. This is because we only have sequential memory accesses for boids within the same grid cell. By checking more cells and reducing the number of boids in each cell, this should reduce the number of sequential memory accesses we do and introduce more scatttered jumps, reducing our performance. 
However, in this case for both `Uniform` and `Coherent`, the 27 blocks with smaller width performed better. I believe this is because the overhead for checking more grid cells ended up being smaller than I expected, and since this resulted in checking less boids overall in each frame, the overhead from memory accesses did not dominate over the scattered memory access jumps between cells.

## Other Comments
- The CMakeLists.txt was modified to add a custom `run_boids` target. This runs the program with some additional env variables set. This was only needed as I use wayland on my device, so I need to specify my xdg session type as x11 for the cuda program to run correctly, and specify that I am using my nvidia graphics card as opposed to intel integrated graphics. This only is there for convenience on my device and does not affect any of the other build processes, as it is a net new change.
