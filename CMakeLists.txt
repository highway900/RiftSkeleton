CMAKE_MINIMUM_REQUIRED( VERSION 2.6 )
PROJECT( RiftSkeleton )

list(APPEND CMAKE_CXX_FLAGS "-std=c++11")
set (CMAKE_CXX_STANDARD 11)

INCLUDE(cmake_modules/InvokePython.cmake)
INVOKEPYTHON( "tools/hardcode_shaders.py" )
# Python script will dump generated headers to autogen/
INCLUDE_DIRECTORIES("autogen/")

# GLFW, SDL, SFML
SET( GL_FRAMEWORK "GLFW" CACHE STRING "Which OpenGL windowing framework to use" )

# Control linking to AntTweakBar and Sixense libraries by toggling a config flag.
# @todo Write or find a "FindXXX.cmake" script to search typical install locations
# on the build machine and link the libraries in.
SET( USE_ANTTWEAKBAR FALSE CACHE BOOL "Use AntTweakBar" )
SET( USE_SIXENSE FALSE CACHE BOOL "Use Sixense" )
SET( USE_OCULUSSDK TRUE CACHE BOOL "Use Oculus SDK" )

SET( USE_CORE_CONTEXT TRUE CACHE BOOL "Use Core Context" )
IF( USE_CORE_CONTEXT )
    ADD_DEFINITIONS(-DUSE_CORE_CONTEXT)
ENDIF()

ADD_DEFINITIONS( -DPROJECT_NAME="${PROJECT_NAME}" )

IF( UNIX )
    ADD_DEFINITIONS( -D_LINUX )
    SET( LIBS_HOME "/home/mattb/Dev/games/libs" )
    
    ADD_DEFINITIONS( -D_DEBUG )
    SET(CMAKE_CXX_FLAGS "-ggdb")
    SET(CMAKE_CXX_FLAGS_DEBUG "-ggdb")
    
    SET( GLM_ROOT "${LIBS_HOME}/glm" CACHE STRING "glm root" )
    INCLUDE_DIRECTORIES( "${GLM_ROOT}" )
    ADD_DEFINITIONS( -DGLM_FORCE_RADIANS )

    IF( GL_FRAMEWORK STREQUAL "GLFW" )
        MESSAGE("Using GLFW Framework.")
        INCLUDE_DIRECTORIES( "${LIBS_HOME}/glfw/include" )
        SET( FRAMEWORK_LIBS -lglfw3 )
    ELSEIF( GL_FRAMEWORK STREQUAL "SDL" )
        MESSAGE("Using SDL2 Framework.")
        INCLUDE_DIRECTORIES( "/usr/local/include/SDL2" )
        LINK_DIRECTORIES   ( "/usr/local/lib/" )
        SET( FRAMEWORK_LIBS -lSDL2 )
    ELSEIF( GL_FRAMEWORK STREQUAL "SFML" )
        MESSAGE("Using SFML Framework.")
        SET( FRAMEWORK_LIBS -lsfml-graphics -lsfml-window -lsfml-system )
    ELSE()
        MESSAGE("Unknown GL_FRAMEWORK: choose GLFW, SDL or SFML.")
    ENDIF()

    IF( USE_OCULUSSDK )
        MESSAGE("Using Oculus")
        SET( OCULUSSDK_ROOT "~/Downloads/software/games/rift/ovr_sdk_linux_0.5.0.1" CACHE STRING "OculusSDK root" )
        INCLUDE_DIRECTORIES( "${OCULUSSDK_ROOT}/LibOVR/Include" )
        INCLUDE_DIRECTORIES( "${OCULUSSDK_ROOT}/LibOVR/Src" )
        INCLUDE_DIRECTORIES( "${OCULUSSDK_ROOT}/LibOVRKernel/Src" )
        LINK_DIRECTORIES   ( "${OCULUSSDK_ROOT}/LibOVR/Lib/Linux/x86_64/Release/" )
        SET( OVR_LIBS -lOVR -ldl -ludev -lXinerama )
        ADD_DEFINITIONS( -DUSE_OCULUSSDK )
    ENDIF( USE_OCULUSSDK )

    IF( USE_SIXENSE )
        MESSAGE("Using Sixense SDK.")
        SET( SIXENSE_ROOT "${LIBS_HOME}/sixenseSDK_linux_OSX" CACHE STRING "SixenseSDK root" )
        INCLUDE_DIRECTORIES( "${SIXENSE_ROOT}/include" )
        LINK_DIRECTORIES   ( "${SIXENSE_ROOT}/lib/linux_x64/debug"
                             "${SIXENSE_ROOT}/lib/linux_x64/release" )
        SET( HYDRA_LIBS
            optimized -lsixense_x64
            optimized -lsixense_utils_x64
            debug -lsixense_x64d
            debug -lsixense_utils_x64d )
        ADD_DEFINITIONS( -DUSE_SIXENSE )
    ENDIF( USE_SIXENSE )

    IF( USE_ANTTWEAKBAR )
        MESSAGE("Using AntTweakBar.")
        SET( ANTTWEAKBAR_ROOT "${LIBS_HOME}/AntTweakBar" CACHE STRING "AntTweakBar root" )
        INCLUDE_DIRECTORIES( "${ANTTWEAKBAR_ROOT}/include" )
        LINK_DIRECTORIES   ( "${ANTTWEAKBAR_ROOT}/lib" )
        SET( ANT_LIBS libAntTweakBar.a )
        ADD_DEFINITIONS( -DUSE_ANTTWEAKBAR )
    ENDIF( USE_ANTTWEAKBAR )

    find_package(OpenGL REQUIRED)
    include_directories( ${OPENGL_INCLUDE_DIRS} )
    INCLUDE_DIRECTORIES( "/usr/include/SOIL" )

    SET( PLATFORM_LIBS
        ${OVR_LIBS}
        -lGLEW
        -lGL
        -lGLU
        -lSOIL
        ${FRAMEWORK_LIBS}
        -pthread
        -lrt
        -lXrandr
        -lXxf86vm
        -lXi
        -lX11
        -lXinerama # GLFW 3.1
        -lXcursor # GLFW 3.1
        -ldl
        -lassimp
        ${HYDRA_LIBS}
        ${ANT_LIBS}
        )
ENDIF()


#
# Platform-independent section
#
FILE( GLOB_RECURSE UTIL_SOURCE_FILES
    src/Util/*.cpp
    src/Util/*.h
    )

FILE( GLOB_RECURSE APPSKELETON_SOURCE_FILES
    src/AppSkeleton/*.cpp
    src/AppSkeleton/*.h
    )

FILE( GLOB_RECURSE SCENE_SOURCE_FILES
    src/Scene/*.cpp
    src/Scene/*.h
    )

FILE( GLOB_RECURSE MODEL_SOURCE_FILES
    src/Model/*.cpp
    src/Model/*.h
    )

FILE( GLOB_RECURSE FLYINGMOUSE_SOURCE_FILES
    src/FlyingMouse/*.cpp
    src/FlyingMouse/*.h
    )

# Manually exclude the OVR-dependent files from file lists.
IF( NOT USE_OCULUSSDK )
    MESSAGE( "Removing OVR-dependent files..." )

    LIST(REMOVE_ITEM APPSKELETON_SOURCE_FILES
        ${CMAKE_CURRENT_SOURCE_DIR}/src/AppSkeleton/RiftAppSkeleton.cpp )

    LIST(REMOVE_ITEM SCENE_SOURCE_FILES
        ${CMAKE_CURRENT_SOURCE_DIR}/src/Scene/OVRScene.cpp )
ENDIF()

INCLUDE_DIRECTORIES("src/Util")
INCLUDE_DIRECTORIES("src/Util/GL")
INCLUDE_DIRECTORIES("src/Scene")
INCLUDE_DIRECTORIES("src/AppSkeleton")
INCLUDE_DIRECTORIES("src/Model")
INCLUDE_DIRECTORIES("src/FlyingMouse")

ADD_LIBRARY( Util        ${UTIL_SOURCE_FILES} )
ADD_LIBRARY( Scene       ${SCENE_SOURCE_FILES} )
ADD_LIBRARY( AppSkeleton ${APPSKELETON_SOURCE_FILES} )
ADD_LIBRARY( Model       ${MODEL_SOURCE_FILES} )
ADD_LIBRARY( FlyingMouse ${FLYINGMOUSE_SOURCE_FILES} )

IF( GL_FRAMEWORK STREQUAL "GLFW" )
    ADD_EXECUTABLE( ${PROJECT_NAME} src/glfw_main.cpp )
ELSEIF( GL_FRAMEWORK STREQUAL "SDL" )
    ADD_EXECUTABLE( ${PROJECT_NAME} src/sdl_main.cpp )
ELSEIF( GL_FRAMEWORK STREQUAL "SFML" )
    ADD_EXECUTABLE( ${PROJECT_NAME} src/sfml_main.cpp )
ENDIF()

TARGET_LINK_LIBRARIES( ${PROJECT_NAME}
    AppSkeleton
    Model
    Scene
    Util
    FlyingMouse
    ${PLATFORM_LIBS}
    )
