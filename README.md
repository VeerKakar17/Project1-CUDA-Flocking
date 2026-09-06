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

## Technical Implementation Specifics

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
- The `Naive` implementation's FPS decreased quadratically (proportional to `O(n^2)`) as boid count increased. 
- The `Uniform` and `Coherent` implementations' FPS's both decreased roughly linearly as boid count increased (proportional to `O(n)`).
- While at very low boid counts `Uniform` has higher FPS than `Coherent`, as the boid count increases, `Coherent` stabally has higher FPS. 

### How does Block Count / Size affect performance?
![](/images/Block%20Size%20to%20Framerate.png)
![](/images/Naive%20(5k)%20vs.%20Block%20Size.png)
These graphs here are showing the performance change based on block size (number of threads per block).

### Does the Coherent implementation perform strictly better than Uniform?

### How does checking all 27 neighboring cells with half the cell width affect performance?
![](/images/8%20vs%2027%20block%20to%20Performance.png)
