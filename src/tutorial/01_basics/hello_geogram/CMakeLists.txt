# Instructions to compile this program
# are explained in tutorials/README.txt

cmake_minimum_required(VERSION 3.10)
project(hello_geogram)

if (NOT CMAKE_BUILD_TYPE)
  message (STATUS "No build type selected, default to Release")
  set (CMAKE_BUILD_TYPE "Release")
endif ()

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR})
find_package(Geogram REQUIRED)

add_executable(hello_geogram main.cpp)
target_link_libraries(hello_geogram Geogram::geogram)


