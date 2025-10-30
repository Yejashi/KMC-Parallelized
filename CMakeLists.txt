cmake_minimum_required(VERSION 3.15)
project(KMC3_PARALLEL LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_BUILD_TYPE Release)

option(ENABLE_CALIPER "Enable Caliper instrumentation" OFF)
option(ENABLE_MPI "Enable MPI parallelism" OFF)
option(ENABLE_RAJA "Enable RAJA parallelism" OFF)

set(KMC_MAIN_DIR kmc_core)
set(KMC_CLI_DIR kmc_CLI)
set(KMC_API_DIR kmc_api)
set(KMC_DUMP_DIR kmc_dump)
set(KMC_TOOLS_DIR kmc_tools)
set(PY_KMC_API_DIR py_kmc_api)
set(OUT_BIN_DIR ${CMAKE_BINARY_DIR}/bin)
set(OUT_INCLUDE_DIR ${CMAKE_BINARY_DIR}/include)

include_directories(
    ${KMC_MAIN_DIR}
    ${KMC_API_DIR}
    3rd_party/cloudflare
)

if(CMAKE_SYSTEM_PROCESSOR MATCHES "arm" OR CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64")
    add_compile_options(-march=armv8-a)
else()
    add_compile_options(-m64)
endif()

add_compile_options(-Wall -O3 -fsigned-char -pthread)

if(ENABLE_CALIPER)
    find_package(Caliper REQUIRED)
    add_definitions(-DENABLE_CALIPER)
endif()

if(ENABLE_MPI)
    find_package(MPI REQUIRED)
    add_definitions(-DENABLE_MPI)
endif()

if(ENABLE_RAJA)
    find_package(RAJA REQUIRED)
    add_definitions(-DENABLE_RAJA)
endif()

file(GLOB KMC_CORE_SRC
    ${KMC_MAIN_DIR}/*.cpp
)
file(GLOB KMC_CLI_SRC
    ${KMC_CLI_DIR}/*.cpp
)
file(GLOB KMC_DUMP_SRC
    ${KMC_DUMP_DIR}/*.cpp
)
file(GLOB KMC_API_SRC
    ${KMC_API_DIR}/*.cpp
)
file(GLOB KMC_TOOLS_SRC
    ${KMC_TOOLS_DIR}/*.cpp
)

add_subdirectory(3rd_party/cloudflare)

add_library(kmc_core STATIC ${KMC_CORE_SRC} ${KMC_API_SRC})
target_include_directories(kmc_core PUBLIC ${OUT_INCLUDE_DIR})
target_link_libraries(kmc_core PRIVATE z)

if(ENABLE_CALIPER)
    target_link_libraries(kmc_core PRIVATE caliper)
endif()
if(ENABLE_MPI)
    target_link_libraries(kmc_core PRIVATE MPI::MPI_CXX)
endif()
if(ENABLE_RAJA)
    target_link_libraries(kmc_core PRIVATE RAJA)
endif()

add_executable(kmc ${KMC_CLI_SRC})
target_link_libraries(kmc PRIVATE kmc_core)

add_executable(kmc_dump ${KMC_DUMP_SRC})
target_link_libraries(kmc_dump PRIVATE kmc_core)

add_executable(kmc_tools ${KMC_TOOLS_SRC})
target_link_libraries(kmc_tools PRIVATE kmc_core z)

set_target_properties(kmc kmc_dump kmc_tools PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY ${OUT_BIN_DIR}
)

