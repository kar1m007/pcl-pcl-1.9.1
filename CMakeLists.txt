### ---[ PCL global CMake
cmake_minimum_required(VERSION 3.5 FATAL_ERROR)

if(POLICY CMP0074)
  # TODO:
  # 1. Find*.cmake modules need to be individually verified.
  # 2. PCLConfig.cmake needs to be changed.
  cmake_policy(SET CMP0074 OLD)
endif()

set(CMAKE_CONFIGURATION_TYPES "Debug;Release" CACHE STRING "possible configurations" FORCE)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# In case the user does not setup CMAKE_BUILD_TYPE, assume it's RelWithDebInfo
if("${CMAKE_BUILD_TYPE}" STREQUAL "")
  set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "build type default to RelWithDebInfo, set to Release to improve performance" FORCE)
endif()

project(PCL VERSION 1.9.1)
string(TOLOWER ${PROJECT_NAME} PROJECT_NAME_LOWER)

### ---[ Find universal dependencies
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules/" ${CMAKE_MODULE_PATH})

# ---[ Include pkgconfig
include(FindPkgConfig)

# ---[ Release/Debug specific flags
if(CMAKE_BUILD_TYPE STREQUAL "Release" OR CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
  add_definitions("-DBOOST_DISABLE_ASSERTS -DEIGEN_NO_DEBUG")
endif()
if(WIN32 AND NOT MINGW)
  if(NOT DEFINED CMAKE_DEBUG_POSTFIX)
    set(CMAKE_DEBUG_POSTFIX "_debug")
  endif()
  if(NOT DEFINED CMAKE_RELEASE_POSTFIX)
    set(CMAKE_RELEASE_POSTFIX "_release")
  endif()
  if(NOT DEFINED CMAKE_RELWITHDEBINFO_POSTFIX)
    set(CMAKE_RELWITHDEBINFO_POSTFIX "_release")
  endif()
  if(NOT DEFINED CMAKE_MINSIZEREL_POSTFIX)
    set(CMAKE_MINSIZEREL_POSTFIX "_release")
  endif()
endif()

# ---[ special maintainer mode
SET(CMAKE_CXX_FLAGS_MAINTAINER "-pedantic -Wno-variadic-macros -Weffc++ -Wno-long-long" CACHE STRING
    "Flags used by the C++ compiler during maintainer builds."
    FORCE)
SET(CMAKE_C_FLAGS_MAINTAINER "-pedantic -Wno-variadic-macros -Weffc++ -Wno-long-long" CACHE STRING
    "Flags used by the C compiler during maintainer builds."
    FORCE)
SET(CMAKE_EXE_LINKER_FLAGS_MAINTAINER
    "-Wl,--warn-unresolved-symbols,--warn-once" CACHE STRING
    "Flags used for linking binaries during maintainer builds."
    FORCE)
SET(CMAKE_SHARED_LINKER_FLAGS_MAINTAINER
    "-Wl,--warn-unresolved-symbols,--warn-once" CACHE STRING
    "Flags used by the shared libraries linker during maintainer builds."
    FORCE)
MARK_AS_ADVANCED(
    CMAKE_CXX_FLAGS_MAINTAINER
    CMAKE_C_FLAGS_MAINTAINER
    CMAKE_EXE_LINKER_FLAGS_MAINTAINER
    CMAKE_SHARED_LINKER_FLAGS_MAINTAINER )
# Update the documentation string of CMAKE_BUILD_TYPE for GUIs
SET(CMAKE_BUILD_TYPE "${CMAKE_BUILD_TYPE}" CACHE STRING
    "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel Maintainer."
    FORCE)

# Compiler identification
# Define a variable CMAKE_COMPILER_IS_X where X is the compiler short name.
# Note: CMake automatically defines one for GNUCXX, nothing to do in this case.
if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
  set(CMAKE_COMPILER_IS_CLANG 1)
elseif(__COMPILER_PATHSCALE)
  set(CMAKE_COMPILER_IS_PATHSCALE 1)
elseif(MSVC)
  set(CMAKE_COMPILER_IS_MSVC 1)
endif()

# Create a variable with expected default CXX flags
# This will be used further down the road to check if the user explicitly provided CXX flags
if(CMAKE_COMPILER_IS_MSVC)
  set(CMAKE_CXX_FLAGS_DEFAULT "/DWIN32 /D_WINDOWS /W3 /GR /EHsc")
else()
  set(CMAKE_CXX_FLAGS_DEFAULT "")
endif()

include("${PCL_SOURCE_DIR}/cmake/pcl_verbosity.cmake")
include("${PCL_SOURCE_DIR}/cmake/pcl_targets.cmake")
include("${PCL_SOURCE_DIR}/cmake/pcl_options.cmake")

# Enable verbose timing display?
if(CMAKE_TIMING_VERBOSE AND UNIX)
  set_property(GLOBAL PROPERTY RULE_MESSAGES OFF)
  set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${CMAKE_SOURCE_DIR}/cmake/custom_output.sh")
endif()

# check for SSE flags
include("${PCL_SOURCE_DIR}/cmake/pcl_find_sse.cmake")
if(PCL_ENABLE_SSE AND "${CMAKE_CXX_FLAGS}" STREQUAL "${CMAKE_CXX_FLAGS_DEFAULT}")
  PCL_CHECK_FOR_SSE()
endif()

# ---[ Unix/Darwin/Windows specific flags
if(CMAKE_COMPILER_IS_GNUCXX)
  if("${CMAKE_CXX_FLAGS}" STREQUAL "${CMAKE_CXX_FLAGS_DEFAULT}")
    SET(CMAKE_CXX_FLAGS "-Wall -Wextra -Wno-unknown-pragmas -fno-strict-aliasing -Wno-format-extra-args -Wno-sign-compare -Wno-invalid-offsetof -Wno-conversion ${SSE_FLAGS_STR}")

    # Enable -Wabi for GCC > 4.3, and -Wno-deprecated for GCC < 4.3
    # to disable a lot of warnings which are not fixable
    execute_process(COMMAND ${CMAKE_C_COMPILER} -dumpversion OUTPUT_VARIABLE GCC_VERSION)
    if(GCC_VERSION VERSION_GREATER 4.3)
      message(STATUS "-- GCC > 4.3 found, enabling -Wabi")
      SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wabi")
    else()
      message(STATUS "-- GCC < 4.3 found, enabling -Wno-deprecated")
      SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-deprecated")
    endif()
  endif()

  if(NOT ANDROID)
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread")
  endif(NOT ANDROID)

  if("${CMAKE_SHARED_LINKER_FLAGS}" STREQUAL "" AND NOT CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    SET(CMAKE_SHARED_LINKER_FLAGS "-Wl,--as-needed")
  endif()

  if(WIN32)
    if(PCL_SHARED_LIBS)
      SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--export-all-symbols -Wl,--enable-auto-import")
      if(MINGW)
        add_definitions("-DBOOST_THREAD_USE_LIB")
        SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--allow-multiple-definition")
      endif()
    else()
      add_definitions("-DBOOST_LIB_DIAGNOSTIC -DBOOST_THREAD_USE_LIB")
    endif()
  endif()
endif()

if(CMAKE_COMPILER_IS_MSVC)
  add_definitions("-DBOOST_ALL_NO_LIB -D_SCL_SECURE_NO_WARNINGS -D_CRT_SECURE_NO_WARNINGS -DNOMINMAX -DPCL_ONLY_CORE_POINT_TYPES /bigobj ${SSE_DEFINITIONS}")

  if("${CMAKE_CXX_FLAGS}" STREQUAL "${CMAKE_CXX_FLAGS_DEFAULT}")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /bigobj /fp:precise /wd4800 /wd4521 /wd4251 /wd4275 /wd4305 /wd4355 ${SSE_FLAGS_STR}")

    # Add extra code generation/link optimizations
    if(CMAKE_MSVC_CODE_LINK_OPTIMIZATION)
      SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /GL")
      SET(CMAKE_SHARED_LINKER_FLAGS_RELEASE "${CMAKE_SHARED_LINKER_FLAGS_RELEASE} /LTCG /OPT:REF")
      SET(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /LTCG")
    endif()
    # /MANIFEST:NO") # please, don't disable manifest generation, otherwise crash at start for vs2008

    if(MSVC_VERSION GREATER 1500 AND ${CMAKE_VERSION} VERSION_GREATER "2.8.6")
      include(ProcessorCount)
      ProcessorCount(N)
      if(NOT N EQUAL 0)
        SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /MP${N}")
        SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP${N}")
      endif()
    endif()
  endif()

  if(CMAKE_GENERATOR STREQUAL "Ninja")
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /FS")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /FS")
  endif()
endif()

if(CMAKE_COMPILER_IS_PATHSCALE)
  if("${CMAKE_CXX_FLAGS}" STREQUAL "${CMAKE_CXX_FLAGS_DEFAULT}")
    SET(CMAKE_CXX_FLAGS "-Wno-uninitialized -zerouv -pthread -mp")
  endif()
  if("${CMAKE_SHARED_LINKER_FLAGS}" STREQUAL "")
    SET(CMAKE_SHARED_LINKER_FLAGS "-mp")
  endif()
endif()

if(CMAKE_COMPILER_IS_CLANG)
  if("${CMAKE_C_FLAGS}" STREQUAL "${CMAKE_CXX_FLAGS_DEFAULT}")
    SET(CMAKE_C_FLAGS "-Qunused-arguments")
  endif()
  if("${CMAKE_CXX_FLAGS}" STREQUAL "")
    SET(CMAKE_CXX_FLAGS "-ftemplate-depth=1024 -Qunused-arguments -Wno-invalid-offsetof ${SSE_FLAGS_STR}") # Unfortunately older Clang versions do not have this: -Wno-unnamed-type-template-args
    if(APPLE AND WITH_CUDA AND CUDA_FOUND)
      SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libstdc++")
    endif()
  endif()
  SET(CLANG_LIBRARIES "stdc++")
endif()

include("${PCL_SOURCE_DIR}/cmake/pcl_utils.cmake")
DISSECT_VERSION()
GET_OS_INFO()
SET_INSTALL_DIRS()

if(WIN32)
  set(PCL_RESOURCES_DIR "${PCL_SOURCE_DIR}/resources")
  set(PCL_POINTCLOUDS_DIR "${PCL_RESOURCES_DIR}/pointclouds")
endif()

set(PCL_OUTPUT_LIB_DIR "${PCL_BINARY_DIR}/${LIB_INSTALL_DIR}")
set(PCL_OUTPUT_BIN_DIR "${PCL_BINARY_DIR}/${BIN_INSTALL_DIR}")
make_directory("${PCL_OUTPUT_LIB_DIR}")
make_directory("${PCL_OUTPUT_BIN_DIR}")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${PCL_OUTPUT_LIB_DIR}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PCL_OUTPUT_BIN_DIR}")
if(WIN32)
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PCL_OUTPUT_BIN_DIR}")
  foreach(config ${CMAKE_CONFIGURATION_TYPES})
    string(TOUPPER ${config} CONFIG)
    set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_${CONFIG} "${PCL_OUTPUT_LIB_DIR}")
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_${CONFIG} "${PCL_OUTPUT_BIN_DIR}")
    # ---[ Windows requires DLLs (shared libraries) to be installed in the same directory as executables
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_${CONFIG} "${PCL_OUTPUT_BIN_DIR}")
  endforeach(config)
else()
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PCL_OUTPUT_LIB_DIR}")
endif()

# Add an "uninstall" target
configure_file("${PCL_SOURCE_DIR}/cmake/uninstall_target.cmake.in"
               "${PCL_BINARY_DIR}/uninstall_target.cmake" IMMEDIATE @ONLY)
add_custom_target(uninstall "${CMAKE_COMMAND}" -P
                  "${PCL_BINARY_DIR}/uninstall_target.cmake")

###
# this is copy paste form http://www.itk.org/Wiki/CMake_RPATH_handling
# in order to always make a full statement RPATH
###
SET(CMAKE_SKIP_BUILD_RPATH  FALSE)
SET(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
SET(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/${LIB_INSTALL_DIR}")
SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
LIST(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/${LIB_INSTALL_DIR}" is_system_dir)
IF("${is_system_dir}" STREQUAL "-1")
SET(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/${LIB_INSTALL_DIR}")
ENDIF("${is_system_dir}" STREQUAL "-1")

### ---[ Find universal dependencies
# the gcc-4.2.1 coming with MacOS X is not compatible with the OpenMP pragmas we use, so disabling OpenMP for it
if((NOT APPLE) OR (NOT CMAKE_COMPILER_IS_GNUCXX) OR (GCC_VERSION VERSION_GREATER 4.2.1) OR (CMAKE_COMPILER_IS_CLANG))
  find_package(OpenMP)
endif()
if(OPENMP_FOUND)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
  message(STATUS "Found OpenMP")
  if(MSVC)
    if(MSVC_VERSION EQUAL 1500)
      set(OPENMP_DLL VCOMP90)
    elseif(MSVC_VERSION EQUAL 1600)
      set(OPENMP_DLL VCOMP100)
    elseif(MSVC_VERSION EQUAL 1700)
      set(OPENMP_DLL VCOMP110)
    elseif(MSVC_VERSION EQUAL 1800)
      set(OPENMP_DLL VCOMP120)
    elseif(MSVC_VERSION EQUAL 1900)
      set(OPENMP_DLL VCOMP140)
    elseif(MSVC_VERSION MATCHES "^191[0-9]$")
      set(OPENMP_DLL VCOMP140)
    endif()
    if(OPENMP_DLL)
      set(CMAKE_SHARED_LINKER_FLAGS_DEBUG "${CMAKE_SHARED_LINKER_FLAGS_DEBUG} /DELAYLOAD:${OPENMP_DLL}D.dll")
      set(CMAKE_SHARED_LINKER_FLAGS_RELEASE "${CMAKE_SHARED_LINKER_FLAGS_RELEASE} /DELAYLOAD:${OPENMP_DLL}.dll")
    else()
      message(WARNING "Delay loading flag for OpenMP DLL is invalid.")
    endif()
  endif(MSVC)
else()
  message(STATUS "Not found OpenMP")
endif()

# Eigen (required)
find_package(Eigen REQUIRED)
include_directories(SYSTEM ${EIGEN_INCLUDE_DIRS})

# FLANN (required)
if(NOT PCL_SHARED_LIBS OR ((WIN32 AND NOT MINGW) AND NOT PCL_BUILD_WITH_FLANN_DYNAMIC_LINKING_WIN32))
  set(FLANN_USE_STATIC ON)
endif()
find_package(FLANN 1.7.0 REQUIRED)
include_directories(${FLANN_INCLUDE_DIRS})

# libusb-1.0
option(WITH_LIBUSB "Build USB RGBD-Camera drivers" TRUE)
if(WITH_LIBUSB)
  find_package(libusb-1.0)
  if(LIBUSB_1_FOUND)
    include_directories("${LIBUSB_1_INCLUDE_DIR}")
  endif()
endif()

# Dependencies for different grabbers
PCL_ADD_GRABBER_DEPENDENCY("OpenNI" "OpenNI grabber support")
PCL_ADD_GRABBER_DEPENDENCY("OpenNI2" "OpenNI2 grabber support")
PCL_ADD_GRABBER_DEPENDENCY("FZAPI" "Fotonic camera support")
PCL_ADD_GRABBER_DEPENDENCY("Ensenso" "IDS-Imaging Ensenso camera support")
PCL_ADD_GRABBER_DEPENDENCY("davidSDK" "David Vision Systems SDK support")
PCL_ADD_GRABBER_DEPENDENCY("DSSDK" "DepthSense SDK support")
PCL_ADD_GRABBER_DEPENDENCY("RSSDK" "RealSense SDK support")

# metslib
if(PKG_CONFIG_FOUND)
  pkg_check_modules(METSLIB metslib)
  if(METSLIB_FOUND)
    set(HAVE_METSLIB ON)
    include_directories(${METSLIB_INCLUDE_DIRS})
  else()
    include_directories("${PCL_SOURCE_DIR}/recognition/include/pcl/recognition/3rdparty/")
  endif()
else()
    include_directories(${PCL_SOURCE_DIR}/recognition/include/pcl/recognition/3rdparty/)
endif()

# LibPNG
option(WITH_PNG "PNG file support" TRUE)
if(WITH_PNG)
  find_package(PNG)
  if(PNG_FOUND)
    set(HAVE_PNG ON)
    include_directories("${PNG_INCLUDE_DIR}")
  endif()
endif()

# Qhull
option(WITH_QHULL "Include convex-hull operations" TRUE)
if(WITH_QHULL)
  if(NOT PCL_SHARED_LIBS OR ((WIN32 AND NOT MINGW) AND NOT PCL_BUILD_WITH_QHULL_DYNAMIC_LINKING_WIN32))
    set(QHULL_USE_STATIC ON)
  endif()
  find_package(Qhull)
  if(QHULL_FOUND)
    include_directories(${QHULL_INCLUDE_DIRS})
  endif()
endif()

# Cuda
option(WITH_CUDA "Build NVIDIA-CUDA support" TRUE)
if(WITH_CUDA)
  include("${PCL_SOURCE_DIR}/cmake/pcl_find_cuda.cmake")
endif()

option(WITH_QT "Build QT Front-End" TRUE)
if(WITH_QT)
  set(PCL_QT_VERSION 5 CACHE STRING "Which QT version to use")
  if("${PCL_QT_VERSION}" STREQUAL "4")
    find_package(Qt4)
    if(QT4_FOUND)
      include("${QT_USE_FILE}")
    endif()
  elseif("${PCL_QT_VERSION}" STREQUAL "5")
    include(cmake/pcl_find_qt5.cmake)
  else()
    message(SEND_ERROR "PCL_QT_VERSION must be 4 or 5")
  endif()
endif()

# Find VTK
option(WITH_VTK "Build VTK-Visualizations" TRUE)
if(WITH_VTK AND NOT ANDROID)
  find_package(VTK)
  if(VTK_FOUND)
    if(NOT DEFINED VTK_RENDERING_BACKEND)
      # On old VTK versions this variable does not exist. In this case it is
      # safe to assume OpenGL backend
      set(VTK_RENDERING_BACKEND "OpenGL")
    endif()
    message(STATUS "VTK_MAJOR_VERSION ${VTK_MAJOR_VERSION}, rendering backend: ${VTK_RENDERING_BACKEND}")
    if(PCL_SHARED_LIBS OR (NOT (PCL_SHARED_LIBS) AND NOT (VTK_BUILD_SHARED_LIBS)))
      set(VTK_FOUND TRUE)
      find_package(QVTK)
      if(${VTK_MAJOR_VERSION} VERSION_LESS "6.0")
         message(STATUS "VTK found (include: ${VTK_INCLUDE_DIRS}, lib: ${VTK_LIBRARY_DIRS})")
         link_directories(${VTK_LIBRARY_DIRS})
      else()
         include(${VTK_USE_FILE})
         message(STATUS "VTK found (include: ${VTK_INCLUDE_DIRS}, lib: ${VTK_LIBRARIES}")
      endif()
      if(APPLE)
          option(VTK_USE_COCOA "Use Cocoa for VTK render windows" ON)
          MARK_AS_ADVANCED(VTK_USE_COCOA)
      endif()
      if(${VTK_RENDERING_BACKEND} STREQUAL "OpenGL")
        set(VTK_RENDERING_BACKEND_OPENGL_VERSION "1")
      elseif(${VTK_RENDERING_BACKEND} STREQUAL "OpenGL2")
        set(VTK_RENDERING_BACKEND_OPENGL_VERSION "2")
      endif()
      set(HAVE_VTK ON)
    else()
      set(VTK_FOUND OFF)
      set(HAVE_VTK OFF)
      message("Warning: You are to build PCL in STATIC but VTK is SHARED!")
      message("Warning: VTK disabled!")
    endif()
  endif()
else()
  set(VTK_FOUND OFF)
  set(HAVE_VTK OFF)
endif()


#Find PCAP
option(WITH_PCAP "pcap file capabilities in Velodyne HDL driver" TRUE)
if(WITH_PCAP)
  find_package(Pcap)
endif()

# OpenGL and GLUT
option(WITH_OPENGL "Support for OpenGL" TRUE)
if(WITH_OPENGL)
  include("${PCL_SOURCE_DIR}/cmake/pcl_find_gl.cmake")
endif()

# Boost (required)
include("${PCL_SOURCE_DIR}/cmake/pcl_find_boost.cmake")

### ---[ Create the config.h file
set(pcl_config_h_in "${CMAKE_CURRENT_SOURCE_DIR}/pcl_config.h.in")
set(pcl_config_h "${CMAKE_CURRENT_BINARY_DIR}/include/pcl/pcl_config.h")
configure_file("${pcl_config_h_in}" "${pcl_config_h}")
PCL_ADD_INCLUDES(common "" "${pcl_config_h}")
include_directories("${CMAKE_CURRENT_BINARY_DIR}/include")

### ---[ Set up for tests
enable_testing()

### ---[ Set up for examples
#include("${PCL_SOURCE_DIR}/cmake/pcl_examples.cmake")

### ---[ Add the libraries subdirectories
include("${PCL_SOURCE_DIR}/cmake/pcl_targets.cmake")

collect_subproject_directory_names("${PCL_SOURCE_DIR}" "CMakeLists.txt" PCL_MODULES_NAMES PCL_MODULES_DIRS doc)
set(PCL_MODULES_NAMES_UNSORTED ${PCL_MODULES_NAMES})
topological_sort(PCL_MODULES_NAMES PCL_ _DEPENDS)
sort_relative(PCL_MODULES_NAMES_UNSORTED PCL_MODULES_NAMES PCL_MODULES_DIRS)
foreach(subdir ${PCL_MODULES_DIRS})
  add_subdirectory("${PCL_SOURCE_DIR}/${subdir}")
endforeach(subdir)

### ---[ Documentation
add_subdirectory(doc)

### ---[ Configure PCLConfig.cmake
include("${PCL_SOURCE_DIR}/cmake/pcl_pclconfig.cmake")

### ---[ Package creation
include("${PCL_SOURCE_DIR}/cmake/pcl_all_in_one_installer.cmake")
include("${PCL_SOURCE_DIR}/cmake/pcl_cpack.cmake")

if(CPACK_GENERATOR)
  message(STATUS "Found CPack generators: ${CPACK_GENERATOR}")
  PCL_MAKE_CPACK_INPUT()
  set(CPACK_PROJECT_CONFIG_FILE "${PCL_CPACK_CFG_FILE}")
  include(CPack)
endif()
### ---[ Make a pretty picture of the dependency graph
include("${PCL_SOURCE_DIR}/cmake/dep_graph.cmake")
MAKE_DEP_GRAPH()

### ---[ Finish up
PCL_WRITE_STATUS_REPORT()
PCL_RESET_MAPS()
