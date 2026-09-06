**University of Pennsylvania, CIS 5650: GPU Programming and Architecture,
Project 1 - Flocking**

  Veer Kakar
  * [LinkedIn](www.linkedin.com/in/VeerKakar), [personal website](https://veerkakar17.github.io/PortfolioWebsite)
* Tested on: Linux Fedora 44 (Dual Boot from Windows Laptop), Intel Ultra 9 275HX, NVIDIA 5070 laptop

![](/images/demo_ss.png)

## Performance Analysis

The following tests were completed using `NVIDIA Nsight Systems` by running the simulation without visualization initially for 25 seconds to stabalize, and then while profiling for 20 seconds. 
I then took the total number of calls to `kernUpdatePos`, a kernel called once every frame in all 3 implementations, and divided this number by the 20 second profiling time to get number of frames per second. With this measure of performance, a higher FPS is better.

The reason I profiled without visualization here is to prevent graphics overhead from introducing slowdowns and to allow FPS to be uncapped past 165 (the refresh rate of my monitor).
These tests were run with visualization on as well, but dur to the scale of the FPS, only the high boid count naive tests went below 165 FPS, and everything else just read at this cap. 

Additionally, block size was set to 128 for all tests unless it was explicitly specified otherwise.

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