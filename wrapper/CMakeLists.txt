cmake_minimum_required(VERSION 2.8)
project(trtNet)

set(CMAKE_BUILD_TYPE Release)

#include
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)

#src
set(PLUGIN_SOURCES
  src/EntroyCalibrator.cpp
  src/UpsampleLayer.cpp
  src/UpsampleLayer.cu
  src/TrtNet.cpp
)

#
# CUDA Configuration
#
find_package(CUDA REQUIRED)

if(NOT CUDA_FOUND)
  return()
endif()

message(STATUS "CUDA detected: " ${CUDA_VERSION})
# list(APPEND Caffe_INCLUDE_DIRS PUBLIC ${CUDA_INCLUDE_DIRS})
# list(APPEND Caffe_LINKER_LIBS PUBLIC ${CUDA_CUDART_LIBRARY}
#                                      ${CUDA_curand_LIBRARY} ${CUDA_CUBLAS_LIBRARIES})

# # cudnn detection
# if(USE_CUDNN)
#   detect_cuDNN()
#   if(HAVE_CUDNN)
#     list(APPEND Caffe_DEFINITIONS PUBLIC -DUSE_CUDNN)
#     list(APPEND Caffe_INCLUDE_DIRS PUBLIC ${CUDNN_INCLUDE})
#     list(APPEND Caffe_LINKER_LIBS PUBLIC ${CUDNN_LIBRARY})
#   endif()
# endif()

# # setting nvcc arch flags
# caffe_select_nvcc_arch_flags(NVCC_FLAGS_EXTRA)
# list(APPEND CUDA_NVCC_FLAGS ${NVCC_FLAGS_EXTRA})
# message(STATUS "Added CUDA NVCC flags for: ${NVCC_FLAGS_EXTRA_readable}")

# # Boost 1.55 workaround, see https://svn.boost.org/trac/boost/ticket/9392 or
# # https://github.com/ComputationalRadiationPhysics/picongpu/blob/master/src/picongpu/CMakeLists.txt
# if(Boost_VERSION EQUAL 105500)
#   message(STATUS "Cuda + Boost 1.55: Applying noinline work around")
#   # avoid warning for CMake >= 2.8.12
#   set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} \"-DBOOST_NOINLINE=__attribute__((noinline))\" ")
# endif()

# # disable some nvcc diagnostic that apears in boost, glog, glags, opencv, etc.
# foreach(diag cc_clobber_ignored integer_sign_change useless_using_declaration set_but_not_used)
#   list(APPEND CUDA_NVCC_FLAGS -Xcudafe --diag_suppress=${diag})
# endforeach()

# # setting default testing device
# if(NOT CUDA_TEST_DEVICE)
#   set(CUDA_TEST_DEVICE -1)
# endif()

# mark_as_advanced(CUDA_BUILD_CUBIN CUDA_BUILD_EMULATION CUDA_VERBOSE_BUILD)
# mark_as_advanced(CUDA_SDK_ROOT_DIR CUDA_SEPARABLE_COMPILATION)

# # Handle clang/libc++ issue
# if(APPLE)
#   caffe_detect_darwin_version(OSX_VERSION)

#   # OSX 10.9 and higher uses clang/libc++ by default which is incompatible with old CUDA toolkits
#   if(OSX_VERSION VERSION_GREATER 10.8)
#     # enabled by default if and only if CUDA version is less than 7.0
#     caffe_option(USE_libstdcpp "Use libstdc++ instead of libc++" (CUDA_VERSION VERSION_LESS 7.0))
#   endif()
# endif()

set(CUDA_VERBOSE_BUILD ON)


# Specify the cuda host compiler to use the same compiler as cmake.
set(CUDA_HOST_COMPILER ${CMAKE_CXX_COMPILER})


# TensorRT
find_path(TENSORRT_INCLUDE_DIR NvInfer.h
  HINTS ${TENSORRT_ROOT} ${CUDA_TOOLKIT_ROOT_DIR}
  PATH_SUFFIXES include)
MESSAGE(STATUS "Found TensorRT headers at ${TENSORRT_INCLUDE_DIR}")
find_library(TENSORRT_LIBRARY_INFER nvinfer
  HINTS ${TENSORRT_ROOT} ${TENSORRT_BUILD} ${CUDA_TOOLKIT_ROOT_DIR}
  PATH_SUFFIXES lib lib64 lib/x64)
find_library(TENSORRT_LIBRARY_INFER_PLUGIN nvinfer_plugin
  HINTS  ${TENSORRT_ROOT} ${TENSORRT_BUILD} ${CUDA_TOOLKIT_ROOT_DIR}
  PATH_SUFFIXES lib lib64 lib/x64)
  find_library(TENSORRT_LIBRARY_PARSER nvparsers
  HINTS  ${TENSORRT_ROOT} ${TENSORRT_BUILD} ${CUDA_TOOLKIT_ROOT_DIR}
  PATH_SUFFIXES lib lib64 lib/x64)
set(TENSORRT_LIBRARY ${TENSORRT_LIBRARY_INFER} ${TENSORRT_LIBRARY_INFER_PLUGIN} ${TENSORRT_LIBRARY_PARSER})
MESSAGE(STATUS "Find TensorRT libs at ${TENSORRT_LIBRARY}")
find_package_handle_standard_args(
  TENSORRT DEFAULT_MSG TENSORRT_INCLUDE_DIR TENSORRT_LIBRARY)
if(NOT TENSORRT_FOUND)
  message(ERROR
    "Cannot find TensorRT library.")
endif()

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall -Ofast -Wfatal-errors -D_MWAITXINTRIN_H_INCLUDED")	# -std=gnu++11
set(BUILD_DEPS "YES" CACHE BOOL "If YES, will install dependencies into sandbox.  Automatically reset to NO after dependencies are installed.")

# if(NOT "${CUDA_NVCC_FLAGS}" MATCHES "-std=c\\+\\+11" )
#   list(APPEND CUDA_NVCC_FLAGS -std=c++11)
# endif()
list(APPEND CUDA_NVCC_FLAGS "-D_FORCE_INLINES -Xcompiler -fPIC")
CUDA_INCLUDE_DIRECTORIES(${CUDNN_INCLUDE_DIR} ${TENSORRT_INCLUDE_DIR})
CUDA_ADD_LIBRARY(TrtNet STATIC ${PLUGIN_SOURCES})

message(${CUDA_INCLUDE_DIRS})
target_include_directories(TrtNet PUBLIC ${CUDA_INCLUDE_DIRS} ${TENSORRT_INCLUDE_DIR} ${CUDNN_INCLUDE_DIR})
target_link_libraries(TrtNet ${TENSORRT_LIBRARY})