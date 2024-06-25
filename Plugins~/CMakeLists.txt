CMAKE_MINIMUM_REQUIRED(VERSION 3.18)

if(APPLE)
    set(CMAKE_OSX_DEPLOYMENT_TARGET "10.15" CACHE STRING "Minimum OS X deployment version")    
endif()

PROJECT(MeshSyncDCCPlugins)

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/Build/cmake_modules)
set(SRC_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/Src)
set(DIST_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/Dist)
set(EXTERNAL_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/External)

# allow CMake to organize files by folders
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

# ----------------------------------------------------------------------------------------------------------------------

# For other available options, check MeshSync/msConfig.h and MeshUtils/muConfig.h
# Example:
#   -DmuEnablePPL
#   -DmuEnableTBB

# CXX Setup
if(APPLE)
    set(PLATFORM "Mac")
    set(MAC ON)
elseif(ANDROID)
    set(PLATFORM "Android")
elseif(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
    set(PLATFORM "Linux")
    set(LINUX ON)

    #--fpermissive is required on Linux for building MotionBuilder plugins.
    #The error: ‘GetData’ was not declared in this scope, and no declarations were found by argument-dependent 
    #    lookup at the point of instantiation [-fpermissive]
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fpermissive")

elseif(WIN32) 
    set(PLATFORM "Windows")
    add_definitions(-DmuEnablePPL)
    
    # Additional Poco Net dependencies on Windows
    list(APPEND Poco_LIBRARIES "ws2_32.lib;Iphlpapi.lib" )       

    #Compile with MultiThreaded DLL flag to use dynamically-linked runtime library (Poco)
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MD")
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MDd")        
   
    #Set /fp. We need so that -infinity == infinity == NaN. See Transform::setupDataFlags()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /fp:fast")
endif()

# Custom scripts
include(AddPlugin)
include(UnityPackage)

setup_unity_package_version()
add_definitions(-DMESHSYNC_DCC_PLUGIN_VER="${UNITY_PACKAGE_VERSION}")

if(WIN32)
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /std:c++17")
else()
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17")
endif()

# Force Unicode character set
add_definitions(-DUNICODE -D_UNICODE)

# Use Poco static lib
add_definitions(-DPOCO_STATIC)

# ----------------------------------------------------------------------------------------------------------------------


# prerequisites. 
find_package(Poco COMPONENTS Net)
include(ISPC)
set(ENABLE_ZSTD ON CACHE BOOL "Use ZSTD for scene cache compression")
add_definitions(-DmsEnableZSTD)

#CCache variable to hold MeshSync version. This is overwritten by the passed argument
SET(MESHSYNC_VER "0.0.1-preview" CACHE STRING "MeshSync package version")

# MeshSync
include(MeshSync)
get_meshsync(${MESHSYNC_VER})
setup_meshsync(${MESHSYNC_VER})


add_subdirectory("${SRC_ROOT}/MeshSyncClient")

# ----------------------------------------------------------------------------------------------------------------------
# Call add_subdirectory using symbolic link
function(add_subdirectory_link to from)
    if(NOT EXISTS "${SRC_ROOT}/${to}")
        if(WIN32) 
            file(TO_NATIVE_PATH "${SRC_ROOT}/${from}" native_src)
            file(TO_NATIVE_PATH "${SRC_ROOT}/${to}"   native_dest)
            execute_process(WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}" COMMAND cmd.exe /c mklink /D ${native_dest} ${native_src})
        else()   
            execute_process(WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}" COMMAND ln -s "${SRC_ROOT}/${from}" "${SRC_ROOT}/${to}" )
        endif()
        
    endif()
    add_subdirectory("${SRC_ROOT}/${to}")
endfunction()

# ----------------------------------------------------------------------------------------------------------------------

# Blender
list(APPEND BLENDER_VERSIONS
    BLENDER_2.90.1
    BLENDER_2.91.2
    BLENDER_2.92.0
    BLENDER_2.93.7
    BLENDER_3.0.1
    BLENDER_3.1.0
    BLENDER_3.2.0
    BLENDER_3.3.0
    BLENDER_3.4.0
) 

if(BUILD_BLENDER_ALL)
    foreach(CUR_BLENDER_VER IN LISTS BLENDER_VERSIONS)
        option(BUILD_${CUR_BLENDER_VER}_PLUGIN "Build MeshSync Plugin for ${CUR_BLENDER_VER}" ON)
    endforeach(CUR_BLENDER_VER)   
endif()

foreach(CUR_BLENDER_VER IN LISTS BLENDER_VERSIONS)
    option(BUILD_${CUR_BLENDER_VER}_PLUGIN "Build MeshSync Plugin for ${CUR_BLENDER_VER}" OFF)
    if (BUILD_${CUR_BLENDER_VER}_PLUGIN)
        message(STATUS "[PROJECT] Setup: : ${CUR_BLENDER_VER}") 
        add_subdirectory_link(MeshSyncClient${CUR_BLENDER_VER} MeshSyncClientBlender)
    endif()
endforeach(CUR_BLENDER_VER)   




