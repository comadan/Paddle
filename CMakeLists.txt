cmake_minimum_required(VERSION 2.8)

project(paddle CXX C)
set(PADDLE_MAJOR_VERSION 0)
set(PADDLE_MINOR_VERSION 9)
set(PADDLE_PATCH_VERSION 0a0)
set(PADDLE_VERSION ${PADDLE_MAJOR_VERSION}.${PADDLE_MINOR_VERSION}.${PADDLE_PATCH_VERSION})

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")
set(PROJ_ROOT ${CMAKE_SOURCE_DIR})
include(package)
find_package(SWIG 2.0)
find_package(CUDA QUIET)
find_package(Protobuf REQUIRED)
find_package(PythonLibs 2.7 REQUIRED)
find_package(PythonInterp 2.7 REQUIRED)
find_package(ZLIB REQUIRED)
find_package(NumPy REQUIRED)
find_package(Threads REQUIRED)
find_package(AVX QUIET)
find_package(Glog)
find_package(Gflags QUIET)
find_package(GTest)
find_package(Sphinx)
find_package(Doxygen)
include(cblas)
find_program(M4_EXECUTABLE m4)
###################### Configurations ###########################
option(WITH_DSO "Compile PaddlePaddle with dynamic linked libraries" ON)
option(WITH_GPU "Compile PaddlePaddle with gpu" ${CUDA_FOUND})
option(WITH_DOUBLE "Compile PaddlePaddle with double precision, otherwise use single precision" OFF)
option(WITH_AVX "Compile PaddlePaddle with avx intrinsics" ${AVX_FOUND})
option(WITH_PYTHON "Compile PaddlePaddle with python interpreter" ON)
option(WITH_STYLE_CHECK "Style Check for PaddlePaddle" ${PYTHONINTERP_FOUND})
option(WITH_RDMA "Compile PaddlePaddle with rdma support" OFF)
option(WITH_GLOG "Compile PaddlePaddle use glog, otherwise use a log implement internally" ${LIBGLOG_FOUND})
option(WITH_GFLAGS "Compile PaddlePaddle use gflags, otherwise use a flag implement internally" ${GFLAGS_FOUND})
option(WITH_TIMER "Compile PaddlePaddle use timer" OFF)
option(WITH_PROFILER "Compile PaddlePaddle use gpu profiler" OFF)
option(WITH_TESTING "Compile and run unittest for PaddlePaddle" ${GTEST_FOUND})
option(WITH_DOC "Compile PaddlePaddle with documentation" OFF)
option(WITH_SWIG_PY "Compile PaddlePaddle with py PaddlePaddle prediction api" ${SWIG_FOUND})
option(ON_TRAVIS "Running test on travis-ci or not." OFF)
option(ON_COVERALLS "Generating code coverage data on coveralls or not." OFF)
option(COVERALLS_UPLOAD "Uploading the generated coveralls json." ON)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE STRING 
        "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel"
        FORCE)
endif()

include(enableCXX11)
include(cpplint)
include(ccache)
if(WITH_RDMA)
  include(rdma)
endif()
include(util)
include(flags)
include(cudnn)
include(FindPythonModule)
include(check_packages)
include(swig)
include(coveralls)

# add PaddlePaddle version
if(DEFINED ENV{PADDLE_VERSION})
    add_definitions(-DPADDLE_VERSION=\"$ENV{PADDLE_VERSION}\")
else()
    if(EXISTS ${PROJ_ROOT}/.svn/)
        find_package(Subversion REQUIRED)
        if(SUBVERSION_FOUND)
            Subversion_WC_INFO(${PROJ_ROOT} Project)
            add_definitions(-DPADDLE_VERSION=${Project_WC_REVISION})
        endif()
    elseif(EXISTS ${PROJ_ROOT}/.git/)
        find_package(Git REQUIRED)
        execute_process(
            COMMAND ${GIT_EXECUTABLE} log -1 --format=%H
            WORKING_DIRECTORY ${PROJ_ROOT}
            OUTPUT_VARIABLE GIT_SHA1
            RESULT_VARIABLE GIT_RESULT
            ERROR_QUIET OUTPUT_STRIP_TRAILING_WHITESPACE)
        if(NOT ${GIT_RESULT})
            add_definitions(-DPADDLE_VERSION=\"${GIT_SHA1}\")
        else()
            message(WARNING "Cannot add paddle version from git tag")
        endif()
    endif()
endif()


if(NOT WITH_GPU)
    add_definitions(-DPADDLE_ONLY_CPU)
    add_definitions(-DHPPL_STUB_FUNC)
    list(APPEND CMAKE_CXX_SOURCE_FILE_EXTENSIONS cu)
else()
    if(${CUDA_VERSION_MAJOR} GREATER 6)
        if(COMPILER_SUPPORT_CXX11)
            LIST(APPEND CUDA_NVCC_FLAGS -std=c++11)
        endif()
    endif()

    # TODO(yuyang18): Change it to remove std=c++11 in cuda compile.
    set(CUDA_PROPAGATE_HOST_FLAGS OFF)
    if(NOT CUDNN_FOUND)
        message(FATAL_ERROR "Paddle need cudnn to compile")
    endif()
    set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS} "-g -O3 --use_fast_math")

    if(WITH_AVX)
        set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS} "-Xcompiler ${AVX_FLAG}")
    else(WITH_AVX)
        set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS} "-Xcompiler ${SSE3_FLAG}")
    endif(WITH_AVX)

    if(WITH_DSO)
        add_definitions(-DPADDLE_USE_DSO)
    endif(WITH_DSO)

    # Include cuda and cudnn
    include_directories(${CUDNN_INCLUDE_DIR})
    include_directories(${CUDA_TOOLKIT_INCLUDE})
endif(NOT WITH_GPU)

if(WITH_DOUBLE)
    add_definitions(-DPADDLE_TYPE_DOUBLE)
    set(ACCURACY double)
else(WITH_DOUBLE)
    set(ACCURACY float)
endif(WITH_DOUBLE)

if(NOT WITH_TIMER)
    add_definitions(-DPADDLE_DISABLE_TIMER)
endif(NOT WITH_TIMER)

if(NOT WITH_PROFILER)
    add_definitions(-DPADDLE_DISABLE_PROFILER)
endif(NOT WITH_PROFILER)

if(WITH_AVX)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${AVX_FLAG}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${AVX_FLAG}")
else(WITH_AVX)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${SSE3_FLAG}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${SSE3_FLAG}")
endif(WITH_AVX)

if(WITH_PYTHON)
    include_directories(${PYTHON_INCLUDE_DIR})
    include_directories(${PYTHON_NUMPY_INCLUDE_DIR})
else(WITH_PYTHON)
    add_definitions(-DPADDLE_NO_PYTHON)
endif(WITH_PYTHON)

if(WITH_RDMA)
  include_directories("${RDMA_INC_DIR}")
else(WITH_RDMA)
  add_definitions(-DPADDLE_DISABLE_RDMA)
endif(WITH_RDMA)

if(WITH_GLOG)
    add_definitions(-DPADDLE_USE_GLOG)
    include_directories(${LIBGLOG_INCLUDE_DIR})
endif()

if(WITH_GFLAGS)
    add_definitions(-DPADDLE_USE_GFLAGS)
    add_definitions(-DGFLAGS_NS=${GFLAGS_NAMESPACE})
    include_directories(${GFLAGS_INCLUDE_DIRS})
endif()

if(WITH_TESTING)
    enable_testing()
    include_directories(${GTEST_INCLUDE_DIRS})
endif()

include_directories("${CBLAS_INC_DIR}")
include_directories("${PROJ_ROOT}")
include_directories("${PROJ_ROOT}/paddle/cuda/include")
include_directories(${PROTOBUF_INCLUDE_DIRS})
include_directories("${CMAKE_CURRENT_BINARY_DIR}/proto")
if(EXISTS "${PROJ_ROOT}/paddle/internals/CMakeLists.txt")
    set(PADDLE_WITH_INTERNAL ON)
    include(paddle/internals/CMakeLists.txt)
else()
    set(PADDLE_WITH_INTERNAL OFF)
    set(INTERNAL_PROTO_PATH "")
endif()
add_subdirectory(proto)
add_subdirectory(paddle)
add_subdirectory(python)
if(WITH_DOC)
    add_subdirectory(doc)
    add_subdirectory(doc_cn)
endif()
