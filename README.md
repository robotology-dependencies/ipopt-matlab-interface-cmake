# ipopt-matlab-interface-cmake

## Rationale 
IPOPT is distributed with an own MATLAB interface, that however is outdated and is not working with recent MATLAB versions. 
For this reason, this repo contains a copy of the [mexIPOPT](https://github.com/ebertolazzi/mexIPOPT) MATLAB interface to [IPOPT](https://projects.coin-or.org/Ipopt), with its build system modified to use [CMake](https://cmake.org/) to build the bindings both for MATLAB and for [Octave](https://www.gnu.org/software/octave/).  

## Installation 
Build this project as any CMake-based project, for example on \*nix system :
~~~
git clone https://github.com/traversaro/ipopt-matlab-interface-cmake
cd ipopt-matlab-interface-cmake
# Enable the IPOPT_USES_MATLAB and IPOPT_USES_OCTAVE depending if MATLAB or Octave is available 
cmake -DCMAKE_INSTALL_PREFIX=./install -DIPOPT_USES_MATLAB=ON -DIPOPT_USES_OCTAVE=ON ..
cmake --build . 
cmake --build . --target install 
~~~

We use the [CMake's FindMatlab.cmake](https://cmake.org/cmake/help/v3.5/module/FindMatlab.html) and a local [FindOctave.cmake](cmake/FindOctave.cmake) to find MATLAB and Octave. In both cases, CMake should find the programs if the `matlab` and `octave` programs are available in the [PATH](https://en.wikipedia.org/wiki/PATH_(variable)).


## Usage
Once you installed the bindings, just add `${CMAKE_INSTALL_PREFIX}/mex` to the [MATLAB path](https://mathworks.com/help/matlab/matlab_env/what-is-the-matlab-search-path.html) if you are using MATLAB and add `${CMAKE_INSTALL_PREFIX}/octave` to the [Octave PATH](https://www.gnu.org/software/octave/doc/v4.2.0/Manipulating-the-Load-Path.html) if you are using Octave. 

To use the bindings, check the documentation available typing `help ipopt` in the MATLAB/Octave shell, or check the [examples available in the mexIPOPT repository](https://github.com/ebertolazzi/mexIPOPT/tree/master/examples).


# Import a new version of mexIPOPT (only for mantainers)
If you want to update the version of mexIPOPT contained in this repo, please update the `GIT_TAG` CMake variable in https://github.com/traversaro/ipopt-matlab-interface-cmake/blob/master/CMakeLists.txt#L39 and execute the `admin-update-ipopt-vendored-code` target on a clean build directory. This will copy the necessary code in the `extern` directory.
