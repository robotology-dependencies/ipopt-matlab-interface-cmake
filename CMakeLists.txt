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
# Unfortunatly the official version of the bindings provided by IPOPT
# is not compiling, so we need to use the mexIPOPT fork
find_package(Git REQUIRED)
set(VENDORED_GIT_TAG "7adaae5cd52180d5291981043d937a822c2df3a0")
add_custom_target(admin-update-ipopt-vendored-code
                  COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_CURRENT_BINARY_DIR}/ipopt
                  COMMAND ${GIT_EXECUTABLE} clone https://github.com/ebertolazzi/mexIPOPT ${CMAKE_CURRENT_BINARY_DIR}/ipopt
                  COMMAND ${GIT_EXECUTABLE} -C ${CMAKE_CURRENT_BINARY_DIR}/ipopt checkout ${VENDORED_GIT_TAG}
                  COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_BINARY_DIR}/ipopt/lib/ipopt.m
                                                   ${CMAKE_CURRENT_BINARY_DIR}/ipopt/lib/ipopt_auxdata.m
                                                   ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/lib
                  COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_BINARY_DIR}/ipopt/src/ipopt.cc
                                                   ${CMAKE_CURRENT_BINARY_DIR}/ipopt/src/IpoptInterfaceCommon.cc
                                                   ${CMAKE_CURRENT_BINARY_DIR}/ipopt/src/IpoptInterfaceCommon.hh
                                                   ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/src
                  COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_BINARY_DIR}/ipopt/examples/examplelasso.m
                                                   ${CMAKE_CURRENT_BINARY_DIR}/ipopt/examples/examplehs038.m
                                                   ${CMAKE_CURRENT_BINARY_DIR}/ipopt/examples/examplehs051.m
                                                   ${CMAKE_CURRENT_BINARY_DIR}/ipopt/examples/examplehs071.m
                                                   ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/examples
                  COMMENT "Updating vendored ipopt code")

set(src_files ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/src/IpoptInterfaceCommon.cc
              ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/src/IpoptInterfaceCommon.hh
              ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/src/ipopt.cc)

set(example_files ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/examples/examplelasso.m
                  ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/examples/examplehs038.m
                  ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/examples/examplehs051.m
                  ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/examples/examplehs071.m)

# The mexIPOPT fork requires C++11
set(CMAKE_CXX_STANDARD 11)

option(BUILD_TESTING "Create tests using CMake" OFF)
if(BUILD_TESTING)
  enable_testing()
endif()

if(IPOPT_USES_MATLAB)
  find_package(Matlab
          REQUIRED
          MX_LIBRARY
          MAIN_PROGRAM)

  matlab_add_mex(
    NAME ipoptMatlabMex
    SRC ${src_files}
    DOCUMENTATION ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/lib/ipopt.m)

  target_include_directories(ipoptMatlabMex PUBLIC ${Matlab_INCLUDE_DIRS} ${IPOPT_INCLUDE_DIRS})
  target_compile_definitions(ipoptMatlabMex PUBLIC ${IPOPT_DEFINITIONS})

  set_target_properties(ipoptMatlabMex
        PROPERTIES OUTPUT_NAME "ipopt"
        # Specify a different build directory for ipopt.mex between octave and matlab to avoid conflicts
        RUNTIME_OUTPUT_DIRECTORY "matlab"
        ARCHIVE_OUTPUT_DIRECTORY "matlab"
        LIBRARY_OUTPUT_DIRECTORY "matlab"
        PREFIX ""
        SUFFIX .${Matlab_MEX_EXTENSION})

  # entry point in the mex file + taking care of visibility and symbol clashes.
  if(WIN32)
    set_target_properties(ipoptMatlabMex
            PROPERTIES
            DEFINE_SYMBOL "DLL_EXPORT_SYM=__declspec(dllexport)")
  endif()

  # Install the generated front-end to ${CMAKE_INSTALL_PREFIX}/mex
  install(TARGETS ${ipoptMatlabMex} DESTINATION ${CMAKE_INSTALL_PREFIX}/${IPOPT_INSTALL_MATLAB_LIBDIR})
  install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/lib/ipopt.m DESTINATION ${CMAKE_INSTALL_PREFIX}/${IPOPT_INSTALL_MATLAB_MFILESDIR})
  install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/lib/ipopt_auxdata.m DESTINATION ${CMAKE_INSTALL_PREFIX}/${IPOPT_INSTALL_MATLAB_MFILESDIR})

  # Add tests (just checking that the interface runs fine, not checking the actual results)
  foreach(example_file ${example_files})
    get_filename_component(example_name ${example_file} NAME_WE)
    get_filename_component(example_dir ${example_file} DIRECTORY)

    add_test(NAME matlab_${example_name}
             COMMAND  ${Matlab_MAIN_PROGRAM} -nodisplay -nodesktop -nojvm -r "addpath('$<TARGET_FILE_DIR:ipoptMatlabMex>');addpath('${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/lib');addpath('${example_dir}');${example_name};exit(0);")
  endforeach()


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
  # The mexIPOPT fork requires Octave 4.2 because it uses the mexCallMATLABWithTrap function
  find_package(Octave 4.2 REQUIRED)

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
          # Specify a different build directory for ipopt.mex between octave and matlab to avoid conflicts
          RUNTIME_OUTPUT_DIRECTORY "octave"
          ARCHIVE_OUTPUT_DIRECTORY "octave"
          LIBRARY_OUTPUT_DIRECTORY "octave"
          PREFIX ""
          SUFFIX .mex)
  install(TARGETS ipoptOctaveMex DESTINATION ${CMAKE_INSTALL_PREFIX}/${IPOPT_INSTALL_OCTAVE_LIBDIR})
  install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/lib/ipopt.m DESTINATION ${CMAKE_INSTALL_PREFIX}/${IPOPT_INSTALL_OCTAVE_MFILESDIR})
  install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/lib/ipopt_auxdata.m DESTINATION ${CMAKE_INSTALL_PREFIX}/${IPOPT_INSTALL_OCTAVE_MFILESDIR})

  # Add tests (just checking that the interface runs fine, not checking the actual results)
  foreach(example_file ${example_files})
    get_filename_component(example_name ${example_file} NAME_WE)
    get_filename_component(example_dir ${example_file} DIRECTORY)

    add_test(NAME octave_${example_name}
             COMMAND ${OCTAVE_EXECUTABLE} --no-gui --quiet --eval "addpath('$<TARGET_FILE_DIR:ipoptOctaveMex>');addpath('${CMAKE_CURRENT_SOURCE_DIR}/extern/ipopt/lib');addpath('${example_dir}');${example_name};exit(0);")
  endforeach()

endif()

