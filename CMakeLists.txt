cmake_minimum_required(VERSION 2.8)

project(masbcpp)

# C++11
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
if(COMPILER_SUPPORTS_CXX11)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
else()
  message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()

# Figure out if we can enable OpenMP
set(LINK_LIBS)
set(COMPILE_OPTIONS "-funroll-loops -ffast-math")

# disable openmp with xcode, doesn't work
if(NOT CMAKE_GENERATOR MATCHES "Xcode")
if(APPLE)
	if(EXISTS "/usr/local/bin/clang-omp++")
		message(STATUS "Building with OMP support")

		set(CMAKE_CXX_COMPILER "/usr/local/bin/clang-omp++" )
		add_definitions(-DWITH_OPENMP)
		#add_definitions(-DCLANG_OMP)

		set(LINK_LIBS ${LINK_LIBS} -liomp5)
		set(COMPILE_OPTIONS "${COMPILE_OPTIONS} -fopenmp")
		include_directories(/usr/local/include/libiomp)
		link_directories(/usr/local/lib)
	endif()
else()
	find_package(OpenMP)
	if(OPENMP_FOUND)
	
		add_definitions(-DWITH_OPENMP)
		set(COMPILE_OPTIONS "${COMPILE_OPTIONS} ${OpenMP_CXX_FLAGS}")
	endif()
endif()
endif()

# MSVC settings
if (MSVC)
# searching for zlib include and library directory
find_path(ZLIB_INCLUDE_DIR zlib.h)
find_library(ZLIB_LIBRARY zlib)

if (ZLIB_INCLUDE_DIR AND ZLIB_LIBRARY)
    set(ZLIB_FOUND TRUE)
    add_definitions(-DHAVE_ZLIB=1)

    include_directories(${ZLIB_INCLUDE_DIR})
    set(LINK_LIBS ${LINK_LIBS} ${ZLIB_LIBRARY})
endif ()

# Adding winmm.lib for Timer
set(LINK_LIBS ${LINK_LIBS} winmm.lib)

# Avoiding min/max macros in windows.h
add_definitions(-DNOMINMAX)

# Enabling M_PI macro
add_definitions(-D_USE_MATH_DEFINES)
endif (MSVC)

# enable verbose printing by default
add_definitions(-DVERBOSEPRINT)

# global
set(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} ${COMPILE_OPTIONS}")
set(CMAKE_BUILD_TYPE Release)
set_property(GLOBAL PROPERTY LINKER_LANGUAGE CXX)

# include directories
include_directories(${CMAKE_SOURCE_DIR}/thirdparty ${CMAKE_SOURCE_DIR}/thirdparty/vrui)

# bundle all thirdparty stuff as a static library
FILE(GLOB_RECURSE THIRDPARTY thirdparty/*.cpp)
add_library(thirdparty STATIC ${THIRDPARTY})

# build a library from the masbpcpp processing functions
add_library(libmasbcpp STATIC src/compute_ma_processing.cpp src/compute_normals_processing.cpp src/simplify_processing.cpp)

set(LINK_LIBS ${LINK_LIBS} thirdparty libmasbcpp)

# set targets
add_executable(compute_ma src/compute_ma.cpp)
add_executable(compute_normals src/compute_normals.cpp)
add_executable(simplify src/simplify.cpp)

# link targets
target_link_libraries(compute_ma ${LINK_LIBS})
target_link_libraries(compute_normals ${LINK_LIBS})
target_link_libraries(simplify ${LINK_LIBS})

install(TARGETS compute_ma compute_normals simplify DESTINATION bin)
