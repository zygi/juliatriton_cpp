cmake_minimum_required(VERSION 3.18)

project(TritonJll)

# TODO remove and pass somewhere else
set(LLVM_ROOT_DIR "${CMAKE_CURRENT_SOURCE_DIR}/build_mlir")
# debug print the value
message(STATUS "LLVM_ROOT_DIR: ${LLVM_ROOT_DIR}")

if(NOT WIN32)
  list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/triton/cmake")
endif()

# include(${CMAKE_CURRENT_SOURCE_DIR}/triton/CMakeLists.txt)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/triton)
# option(BUILD_SHARED_LIBS "Build using shared libraries" ON)
add_subdirectory(triton)

## COPIED OVER FROM TRITON

find_package(LLVM 11 REQUIRED COMPONENTS "nvptx;amdgpu")
set(LLVM_LIBRARY_DIR ${LLVM_LIBRARY_DIRS})
set(MLIR_DIR ${LLVM_LIBRARY_DIR}/cmake/mlir)
# message(STATUS "MLIR_DIR: ${MLIR_DIR}")

list(APPEND CMAKE_MODULE_PATH ${MLIR_DIR})
find_package(MLIR REQUIRED CONFIG PATHS ${MLIR_DIR})

list(APPEND CMAKE_MODULE_PATH "${MLIR_CMAKE_DIR}")
list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_DIR}")

include(TableGen) # required by AddMLIR
include(AddLLVM)
include(AddMLIR)

# Disable warnings that show up in external code (gtest;pybind11)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror -Wno-covered-switch-default")

include_directories(${MLIR_INCLUDE_DIRS})
include_directories(${LLVM_INCLUDE_DIRS})
include_directories(${PROJECT_SOURCE_DIR}/triton/include)
include_directories(${PROJECT_BINARY_DIR}/triton/include) # Tablegen'd files

set(TRITON_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}")
set(TRITON_BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}")

get_property(dialect_libs GLOBAL PROPERTY MLIR_DIALECT_LIBS)
get_property(conversion_libs GLOBAL PROPERTY MLIR_CONVERSION_LIBS)

set(TRITON_LIBRARIES
  TritonAnalysis
  TritonTransforms
  TritonGPUTransforms
  TritonNvidiaGPUTransforms
  TritonLLVMIR
  TritonPTX
  TritonHSACO
  ${dialect_libs}
  ${conversion_libs}

  # optimizations
  MLIRBytecodeWriter
  MLIRPass
  MLIRTransforms
  MLIRLLVMDialect
  MLIRSupport
  MLIRTargetLLVMIRExport
  MLIRExecutionEngine
  MLIRMathToLLVM
  MLIRNVVMToLLVMIRTranslation
  MLIRROCDLToLLVMIRTranslation
  MLIRIR
)

## END COPIED OVER FROM TRITON

execute_process(COMMAND julia -e "using Pkg; Pkg.activate(\".\"); Pkg.add(\"CxxWrap\"); using CxxWrap; print(CxxWrap.prefix_path())" OUTPUT_VARIABLE CxxWrap_prefix_path OUTPUT_STRIP_TRAILING_WHITESPACE)
set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH} ${CxxWrap_prefix_path})
find_package(JlCxx)
get_target_property(JlCxx_location JlCxx::cxxwrap_julia LOCATION)

message(STATUS "Adding Julia module")
set(JULIA_SRC_PATH ${CMAKE_CURRENT_SOURCE_DIR}/src)
set(JULIA_SRC ${JULIA_SRC_PATH}/main.cc)
add_library(triton_julia SHARED ${JULIA_SRC})
target_include_directories(triton_julia PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/triton/include)
# target_include_directories(triton_julia PRIVATE ${JULIA_SRC_PATH} ${JlCxx_prefix_path}/include )

target_link_libraries(triton_julia
JlCxx::cxxwrap_julia
${LLVM_LIBRARIES}
z
${TRITON_LIBRARIES}
)

# add -Wl,--no-undefined only for this target
target_link_options(triton_julia PRIVATE ${LLVM_LDFLAGS})
set_target_properties(triton_julia PROPERTIES LINK_FLAGS "-Wl,--no-undefined")

# include debug symbols
set_target_properties(triton_julia PROPERTIES
COMPILE_FLAGS "-g"
LINK_FLAGS "-g"
)