# Copyright (C) 2018 Fondazione Istituto Italiano di Tecnologia
# This code is published under the Eclipse Public License.

cmake_minimum_required(VERSION 3.5)

project(ipopt-matlab-interface-cmake)

# for FindMatlab.cmake
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")

option(IPOPT_USES_MATLAB "Do you want to create the MATLAB bindings" FALSE)
option(IPOPT_USES_OCTAVE "Do you want to create the OCTAVE bindings" FALSE)

find_package(IPOPT REQUIRED)

# Options related to installation directories
# If you want to install bindings for packaging you may need to install
# following several rules (for example placing the .m files in share
# and the compiled libraries in some architecture-specific directory)
# We support this use-cases by exposing this CMake advanced option.
# The default values, however, are choosen to simplify the use of
# the bindings for a developer that compiled the library from source:
# to use the octave bindings just add <prefix>/octave to the octave path,
# to use the matlab bindings just add <prefix>/mex    to the matlab path.
set(IPOPT_INSTALL_MATLAB_LIBDIR "mex" CACHE STRING "Location (relative to the install prefix) in which the Matlab mex libraries are installed.")
mark_as_advanced(IPOPT_INSTALL_MATLAB_LIBDIR)
set(IPOPT_INSTALL_MATLAB_MFILESDIR "mex" CACHE STRING "Location (relative to the install prefix) in which the Matlab .m files are installed.")
mark_as_advanced(IPOPT_INSTALL_MATLAB_MFILESDIR)
set(IPOPT_INSTALL_OCTAVE_LIBDIR "octave" CACHE STRING "Location (relative to the install prefix) in which the Octave mex libraries are installed.")
mark_as_advanced(IPOPT_INSTALL_OCTAVE_LIBDIR)
set(IPOPT_INSTALL_OCTAVE_MFILESDIR "octave" CACHE STRING "Location (relative to the install prefix) in which the Octave .m files are installed.")
mark_as_advanced(IPOPT_INSTALL_OCTAVE_MFILESDIR)

# Targets to update the external code vendored with the repo
# Must run from an empty build directory
find_package(Git REQUIRED)
set(VENDORED_GIT_TAG "releases/3.12.9")
add_custom_target(admin-update-ipopt-vendored-code
                  COMMAND ${GIT_EXECUTABLE} clone https://github.com/coin-or/Ipopt ${CMAKE_CURRENT_BINARY_DIR}/ipopt
                  COMMAND ${GIT_EXECUTABLE} -C ${CMAKE_CURRENT_BINARY_DIR}/ipopt checkout ${VENDORED_GIT_TAG}
                  COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_CURRENT_BINARY_DIR}/ipopt/Ipopt/contrib/MatlabInterface/ ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt
                  COMMENT "Updating vendored ipopt code")

set(src_files ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/src/callbackfunctions.cpp
              ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/src/ipopt.cpp
              ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/src/ipoptoptions.cpp
              ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/src/iterate.cpp
              ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/src/matlabexception.cpp
              ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/src/matlabfunctionhandle.cpp
              ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/src/matlabinfo.cpp
              ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/src/matlabjournal.cpp
              ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/src/matlabprogram.cpp
              ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/src/options.cpp
              ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/src/sparsematrix.cpp)

if(IPOPT_USES_MATLAB)
  find_package(Matlab
          REQUIRED
          MX_LIBRARY
          MAIN_PROGRAM)

  matlab_add_mex(
    NAME ipoptMatlabMex
    SRC ${src_files}
    DOCUMENTATION ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/ipopt.m)

  target_include_directories(ipoptMatlabMex PUBLIC ${Matlab_INCLUDE_DIRS} ${IPOPT_INCLUDE_DIRS})
  target_compile_definitions(ipoptMatlabMex PUBLIC ${IPOPT_DEFINITIONS})

  set_target_properties(ipoptMatlabMex PROPERTIES OUTPUT_NAME "ipopt" PREFIX "" SUFFIX .${Matlab_MEX_EXTENSION})

  # entry point in the mex file + taking care of visibility and symbol clashes.
  if(WIN32)
    set_target_properties(ipoptMatlabMex
            PROPERTIES
            DEFINE_SYMBOL "DLL_EXPORT_SYM=__declspec(dllexport)")
  endif()

  # Install the generated front-end to ${CMAKE_INSTALL_PREFIX}/mex
  install(TARGETS ${ipoptMatlabMex} DESTINATION ${CMAKE_INSTALL_PREFIX}/${IPOPT_INSTALL_MATLAB_LIBDIR})
  install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/ipopt.m DESTINATION ${CMAKE_INSTALL_PREFIX}/${IPOPT_INSTALL_MATLAB_MFILESDIR})
  install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/ipopt_auxdata.m DESTINATION ${CMAKE_INSTALL_PREFIX}/${IPOPT_INSTALL_MATLAB_MFILESDIR})

  #On new versions of Clang, MATLAB requires C++11.
  if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    if(${CMAKE_CXX_COMPILER_ID} MATCHES "Clang")
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
      if (${CMAKE_GENERATOR} MATCHES "Xcode")
        #this should set explictly the option in xcode. Unfortunately it does not work.
        set(XCODE_ATTRIBUTE_CLANG_CXX_LANGUAGE_STANDARD "C++11")
      endif(${CMAKE_GENERATOR} MATCHES "Xcode")
    endif(${CMAKE_CXX_COMPILER_ID} MATCHES "Clang")
  endif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
endif()

if(IPOPT_USES_OCTAVE)
  find_package(Octave REQUIRED)

  # Compile MEX file
  add_library(ipoptOctaveMex MODULE ${src_files})
  target_include_directories(ipoptOctaveMex PUBLIC ${OCTAVE_INCLUDE_DIRS} ${IPOPT_INCLUDE_DIRS})
  target_compile_definitions(ipoptOctaveMex PUBLIC ${IPOPT_DEFINITIONS})
  if(APPLE)
    target_link_libraries(ipoptOctaveMex ${IPOPT_LIBRARIES})
    set_target_properties(ipoptOctaveMex PROPERTIES LINK_FLAGS "-undefined dynamic_lookup")
  else()
    target_link_libraries(ipoptOctaveMex ${OCTAVE_LIBRARIES} ${IPOPT_LIBRARIES})
  endif()
  set_target_properties(ipoptOctaveMex
          PROPERTIES OUTPUT_NAME "ipopt"
          PREFIX ""
          SUFFIX .mex)
  install(TARGETS ipoptOctaveMex DESTINATION ${CMAKE_INSTALL_PREFIX}/${IPOPT_INSTALL_OCTAVE_LIBDIR})
  install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/ipopt.m DESTINATION ${CMAKE_INSTALL_PREFIX}/${IPOPT_INSTALL_OCTAVE_MFILESDIR})
  install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/ipopt_auxdata.m DESTINATION ${CMAKE_INSTALL_PREFIX}/${IPOPT_INSTALL_OCTAVE_MFILESDIR})

endif()

