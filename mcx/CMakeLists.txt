########################################################################################################################
#  CMake configure file for Monte Carlo eXtreme (MCX)
#  Qianqian Fang <q.fang at neu.edu>
#  2018/08/26
########################################################################################################################

# Minimum CMake version required to support CUDA as a language
cmake_minimum_required(VERSION 3.18)

project(mcx LANGUAGES C CXX CUDA)

set(CMAKE_CXX_STANDARD 23)

find_package(CUDAToolkit REQUIRED)

find_package(OpenMP REQUIRED)

option(MCX_BUILD_MEX "Build MCX mex file for MATLab" OFF)
option(MCX_BUILD_PYTHON "Build the pmcx Python package" OFF)
option(MCX_CUDA_USE_FAST_MATH "Build the MCX executable/bindings with fast math enabled" ON)
option(MCX_ENABLE_SAVE_DETECTORS "Enables saving detectors in the the MCX executable/bindings" ON)


if (MCX_BUILD_PYTHON)
    add_subdirectory(pybind11)
    find_package(Python3 COMPONENTS Interpreter Development)
    include_directories(${PYTHON_INCLUDE_DIRS})
endif ()

if (MCX_BUILD_MEX)
    find_package(Matlab)
endif ()

# NVCC Options
set(
        CUDA_NVCC_FLAGS
        ${CUDA_NVCC_FLAGS};
        -g -lineinfo -Xcompiler -Wall -Xcompiler -O3 -arch=sm_50
        -DMCX_TARGET_NAME="Fermi MCX" -DUSE_ATOMIC -use_fast_math
        -DSAVE_DETECTORS -Xcompiler -fPIC ${OMPFLAG}
)

# C Options
set(CMAKE_C_FLAGS "-g -Wall -std=c99 -fPIC")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/../bin)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/../lib)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

add_subdirectory(zmat)

# Add include directories
include_directories(cjson ubj zmat zmat/easylzma)

# Add link directories
link_directories(zmat)

# Create mcx library
add_library(mcx STATIC
        mcx_core.cu
        mcx_utils.c
        mcx_shapes.c
        mcx_bench.c
        mcx_lang.c
        mcx_mie.cpp
        mcx_tictoc.c
        mcx_neurojson.cpp
        cjson/cJSON.c
        ubj/ubjw.c
)

if (MCX_ENABLE_SAVE_DETECTORS)
    target_compile_definitions(mcx PUBLIC MCX_SAVE_DETECTORS=1)
endif ()

target_include_directories(mcx PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)

target_link_libraries(mcx PUBLIC CUDA::curand)

# Add all project units
add_executable(
        mcx-exe
        mcx.c
)

set_target_properties(mcx-exe
        PROPERTIES OUTPUT_NAME mcx)

# Link options
target_link_libraries(
        mcx-exe
        mcx OpenMP::OpenMP_CXX
        zmat
)

if (BUILD_PYTHON)
    cuda_add_library(_pmcx MODULE
            mcx_core.cu
            mcx_core.h
            mcx_utils.cpp
            mcx_utils.h
            shapes.cpp
            mcx_shapes.h
            mcx_bench.cpp
            mcx_bench.h
            mcx_lang.cpp
            mcx_lang.h
            mcx_mie.cpp
            mcx_mie.h
            tictoc.c
            mcx_tictoc.h
            cjson/cJSON.c
            cjson/cJSON.h
            pmcx.cpp
    )
    target_compile_definitions(_pmcx PUBLIC MCX_CONTAINER PYBIND11_VERSION_MAJOR)

    target_link_libraries(_pmcx OpenMP::OpenMP_CXX pybind11::module pybind11::lto pybind11::windows_extras)

    pybind11_extension(_pmcx)
    pybind11_strip(_pmcx)

    set_target_properties(_pmcx PROPERTIES CXX_VISIBILITY_PRESET "hidden"
            CUDA_VISIBILITY_PRESET "hidden")
endif ()

# Build mex file
if (BUILD_MEX AND Matlab_FOUND)
    # Create mcx-matlab library
    cuda_add_library(mcx-matlab STATIC
            mcx_core.cuh
            mcx_core.h
            mcx_utils.cpp
            shapes.cpp
            mcx_shapes.h
            mcx_bench.cpp
            mcx_bench.h
            mcx_lang.cpp
            mcx_lang.h
            mcx_mie.cpp
            mcx_mie.h
            tictoc.cpp
            mcx_tictoc.h
            cjson/cJSON.c
            cjson/cJSON.h
    )

    target_compile_definitions(mcx-matlab PUBLIC MCX_CONTAINER MATLAB_MEX_FILE)

    if (${CMAKE_VERSION} VERSION_LESS "3.24.0")
        matlab_add_mex(
                NAME mcxlab
                SRC mcxlab.cpp
                LINK_TO OpenMP::OpenMP_CXX mcx-matlab
        )
    else ()
        matlab_add_mex(
                NAME mcxlab
                SRC mcxlab.cpp
                NO_IMPLICIT_LINK_TO_MATLAB_LIBRARIES
                LINK_TO ${Matlab_MEX_LIBRARY} ${Matlab_MX_LIBRARY} OpenMP::OpenMP_CXX mcx-matlab
        )
    endif ()


    target_compile_definitions(mcxlab PUBLIC MCX_CONTAINER MATLAB_MEX_FILE)

    set_target_properties(mcxlab
            PROPERTIES OUTPUT_NAME ${CMAKE_SOURCE_DIR}/../mcxlab/mcx)
endif ()