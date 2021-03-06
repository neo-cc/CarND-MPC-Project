# CarND-Controls-MPC
Self-Driving Car Engineer Nanodegree Program

---
## MPC Reflection

[//]: # (Image References)
[image1]: ./result/mpc_speed.png

![alt text][image1]

#### The Model

This car is using Kinematic Models, which are simplifications of dynamic models that ignore tire forces, gravity, and mass. This simplification reduces the accuracy of the models, but it also makes them more tractable. At low and moderate speeds, kinematic models often approximate the actual vehicle dynamics.

The model has following states and actuators:

* States: x(x\_position), y(y\_position), psi(orientation), v(velocity)
* Actuators: deltaPsi(steering), a(acceleration)

The update equations for this model are:

```      
      x[1 + t] = x[t] + v[t] * cos(psi[t]) * dt);
      y[1 + t] = y[t] + (y[t] + v[t] * sin(psi[t]) * dt);
      psi[1 + t] = psi[t] + v[t] * delta[t] / Lf * dt);
      v[1 + t] = v[t] + a[t] * dt;
      cte[1 + t] = cte[t] - y[t] + v[t] * sin(epsi[t]) * dt;
      epsi[1 + t] = epsi[t] - psides[t] + v[t] * delta[t] / Lf * dt);
```

#### Timestep Length and Elapsed Duration (N & dt)

In the end, I set the timestep `N=11` and `dt=0.05s`. The car can successfully drive a lap around the track with average speed `60MPH` as shown in the above picture.  

I also tried with other parameters for N and dt: 

* When fixed with dt=0.05s, if I set N to larger number such as 20, the car failed to stay on the track at high speed even with slightly turn. If I set N=5, the car always oscillate on the road and cannot drive in a stable state.
* When fixed with N=11, if I set dt to larger value such as 0.1s, the car still oscillates on the road and cannot deal with Latency very well.  

#### Polynomial Fitting and MPC Preprocessing

I am using 2nd order polynomial fit. I also tried with 3rd order polynomial fit but it didn't provide better result than 2nd order fit. 

      // 2nd order polynomial fit
      f0 = coeffs[0] + coeffs[1] * x0 + coeffs[2] * pow(x0, 2);
      psides0 = atan(coeffs[1] + 2 * coeffs[2] * x0);

The waypoints are first converted to vehicle space parametes and then fit to the path. The x position, y position and car orientation are set to 0 for the initial state and these are fed into the MPC solve function.

MPC prodict the N states and N-1 actuaors with a cost function as follows:

    // The part of the cost based on the reference state.
    for (int t = 0; t < N; t++) {
      fg[0] += 10*CppAD::pow(vars[cte_start + t] , 2);        // add weight to minimize the CTE
      fg[0] += CppAD::pow(vars[epsi_start + t], 2);
      fg[0] += CppAD::pow(vars[v_start + t] - ref_v, 2);
    }
    
    // Minimize the use of actuators.
    for (int t = 0; t < N - 1; t++) {
      fg[0] += 10000*CppAD::pow(vars[deltaPsi_start + t], 2); // add weight to minize the use of deltaPsi
      fg[0] += 30*CppAD::pow(vars[a_start + t], 2);           // add weight to reduce speed to 60mph
    }
    
    // Minimize the value gap between sequential actuations.
    for (int t = 0; t < N - 2; t++) {
      fg[0] += CppAD::pow(vars[deltaPsi_start + t + 1] - vars[deltaPsi_start + t], 2);
      fg[0] += CppAD::pow(vars[a_start + t + 1] - vars[a_start + t], 2);
    }
  
* To make sure the CTE as small as possible so the car always close to center, I added `weight=10` for cte.
* To minimize the use of steering so the car do not oscillate on the track, I add `weight=10000` for deltaPsi. 
* To make sure the car is in a stable state, I added `weight=30` to slow down the car to 60mph, not reaching the ref_v=100mph. 

#### Model Predictive Control with Latency

0.1 second latency in introduced into this system in order to simulate real-life environment. 

To compensate this error, first I added the `latency_step=2` (0.1s/dt) and fixed the steering and acceleration during this latency time. 

```
  // do not change steering during latency
  for (int i = deltaPsi_start; i < deltaPsi_start + latency_step; i++) {
    vars_lowerbound[i] = delta_;
    vars_upperbound[i] = delta_;
  }
  
    // do not change acceleration during latency
  for (int i = a_start; i < a_start + latency_step; i++) {
    vars_lowerbound[i] = a_;
    vars_upperbound[i] = a_;
  }
``` 
Second I only used the steering and acceleration value at the latency time. 
 
```  
  delta_ = solution.x[deltaPsi_start + latency_step];
  a_ = solution.x[a_start + latency_step];
```
In the end, the car is able to compensate the system 100ms latency and can successfully drive through the track with average speed of 60MPH. 

## Dependencies

* cmake >= 3.5
 * All OSes: [click here for installation instructions](https://cmake.org/install/)
* make >= 4.1
  * Linux: make is installed by default on most Linux distros
  * Mac: [install Xcode command line tools to get make](https://developer.apple.com/xcode/features/)
  * Windows: [Click here for installation instructions](http://gnuwin32.sourceforge.net/packages/make.htm)
* gcc/g++ >= 5.4
  * Linux: gcc / g++ is installed by default on most Linux distros
  * Mac: same deal as make - [install Xcode command line tools]((https://developer.apple.com/xcode/features/)
  * Windows: recommend using [MinGW](http://www.mingw.org/)
* [uWebSockets](https://github.com/uWebSockets/uWebSockets)
  * Run either `install-mac.sh` or `install-ubuntu.sh`.
  * If you install from source, checkout to commit `e94b6e1`, i.e.
    ```
    git clone https://github.com/uWebSockets/uWebSockets 
    cd uWebSockets
    git checkout e94b6e1
    ```
    Some function signatures have changed in v0.14.x. See [this PR](https://github.com/udacity/CarND-MPC-Project/pull/3) for more details.
* Fortran Compiler
  * Mac: `brew install gcc` (might not be required)
  * Linux: `sudo apt-get install gfortran`. Additionall you have also have to install gcc and g++, `sudo apt-get install gcc g++`. Look in [this Dockerfile](https://github.com/udacity/CarND-MPC-Quizzes/blob/master/Dockerfile) for more info.
* [Ipopt](https://projects.coin-or.org/Ipopt)
  * Mac: `brew install ipopt`
       +  Some Mac users have experienced the following error:
       ```
       Listening to port 4567
       Connected!!!
       mpc(4561,0x7ffff1eed3c0) malloc: *** error for object 0x7f911e007600: incorrect checksum for freed object
       - object was probably modified after being freed.
       *** set a breakpoint in malloc_error_break to debug
       ```
       This error has been resolved by updrading ipopt with
       ```brew upgrade ipopt --with-openblas```
       per this [forum post](https://discussions.udacity.com/t/incorrect-checksum-for-freed-object/313433/19).
  * Linux
    * You will need a version of Ipopt 3.12.1 or higher. The version available through `apt-get` is 3.11.x. If you can get that version to work great but if not there's a script `install_ipopt.sh` that will install Ipopt. You just need to download the source from the Ipopt [releases page](https://www.coin-or.org/download/source/Ipopt/) or the [Github releases](https://github.com/coin-or/Ipopt/releases) page.
    * Then call `install_ipopt.sh` with the source directory as the first argument, ex: `sudo bash install_ipopt.sh Ipopt-3.12.1`. 
  * Windows: TODO. If you can use the Linux subsystem and follow the Linux instructions.
* [CppAD](https://www.coin-or.org/CppAD/)
  * Mac: `brew install cppad`
  * Linux `sudo apt-get install cppad` or equivalent.
  * Windows: TODO. If you can use the Linux subsystem and follow the Linux instructions.
* [Eigen](http://eigen.tuxfamily.org/index.php?title=Main_Page). This is already part of the repo so you shouldn't have to worry about it.
* Simulator. You can download these from the [releases tab](https://github.com/udacity/self-driving-car-sim/releases).
* Not a dependency but read the [DATA.md](./DATA.md) for a description of the data sent back from the simulator.


## Basic Build Instructions


1. Clone this repo.
2. Make a build directory: `mkdir build && cd build`
3. Compile: `cmake .. && make`
4. Run it: `./mpc`.

## Tips

1. It's recommended to test the MPC on basic examples to see if your implementation behaves as desired. One possible example
is the vehicle starting offset of a straight line (reference). If the MPC implementation is correct, after some number of timesteps
(not too many) it should find and track the reference line.
2. The `lake_track_waypoints.csv` file has the waypoints of the lake track. You could use this to fit polynomials and points and see of how well your model tracks curve. NOTE: This file might be not completely in sync with the simulator so your solution should NOT depend on it.
3. For visualization this C++ [matplotlib wrapper](https://github.com/lava/matplotlib-cpp) could be helpful.

## Editor Settings

We've purposefully kept editor configuration files out of this repo in order to
keep it as simple and environment agnostic as possible. However, we recommend
using the following settings:

* indent using spaces
* set tab width to 2 spaces (keeps the matrices in source code aligned)

## Code Style

Please (do your best to) stick to [Google's C++ style guide](https://google.github.io/styleguide/cppguide.html).

## Project Instructions and Rubric

Note: regardless of the changes you make, your project must be buildable using
cmake and make!

More information is only accessible by people who are already enrolled in Term 2
of CarND. If you are enrolled, see [the project page](https://classroom.udacity.com/nanodegrees/nd013/parts/40f38239-66b6-46ec-ae68-03afd8a601c8/modules/f1820894-8322-4bb3-81aa-b26b3c6dcbaf/lessons/b1ff3be0-c904-438e-aad3-2b5379f0e0c3/concepts/1a2255a0-e23c-44cf-8d41-39b8a3c8264a)
for instructions and the project rubric.

## Hints!

* You don't have to follow this directory structure, but if you do, your work
  will span all of the .cpp files here. Keep an eye out for TODOs.

## Call for IDE Profiles Pull Requests

Help your fellow students!

We decided to create Makefiles with cmake to keep this project as platform
agnostic as possible. Similarly, we omitted IDE profiles in order to we ensure
that students don't feel pressured to use one IDE or another.

However! I'd love to help people get up and running with their IDEs of choice.
If you've created a profile for an IDE that you think other students would
appreciate, we'd love to have you add the requisite profile files and
instructions to ide_profiles/. For example if you wanted to add a VS Code
profile, you'd add:

* /ide_profiles/vscode/.vscode
* /ide_profiles/vscode/README.md

The README should explain what the profile does, how to take advantage of it,
and how to install it.

Frankly, I've never been involved in a project with multiple IDE profiles
before. I believe the best way to handle this would be to keep them out of the
repo root to avoid clutter. My expectation is that most profiles will include
instructions to copy files to a new location to get picked up by the IDE, but
that's just a guess.

One last note here: regardless of the IDE used, every submitted project must
still be compilable with cmake and make./
