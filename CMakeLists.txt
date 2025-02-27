cmake_minimum_required (VERSION 3.5)
include(CheckFunctionExists)
include(FetchContent)
project (vss2svg)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake/")

find_package(libemf2svg REQUIRED)
find_package(libwmf REQUIRED)
find_package(librevenge REQUIRED)
find_package(libvisio REQUIRED)
find_package(LibXml2 REQUIRED)
find_package(Freetype REQUIRED)

# Project version (sematic versionning)
set(vss2svg_VERSION_MAJOR 0)
set(vss2svg_VERSION_MINOR 5)
set(vss2svg_VERSION_PATCH 5)

set(vss2svg_VERSION ${vss2svg_VERSION_MAJOR}.${vss2svg_VERSION_MINOR}.${vss2svg_VERSION_PATCH})
# set version as a definition
set(CMAKE_C_FLAGS "${CMAKE_CXX_FLAGS} -DV2S_VERSION='\"${vss2svg_VERSION}\"'")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DV2S_VERSION='\"${vss2svg_VERSION}\"'")

add_custom_target(tag
	COMMAND git tag -a ${vss2svg_VERSION} -m "tagging version ${vss2svg_VERSION}"
	COMMAND git push origin ${vss2svg_VERSION}
)

# Options
option(DEBUG               "compile with debug symbol"                                          OFF)
option(STATIC              "compile statically"                                                 OFF)
option(USE_CLANG           "build application with clang"                                       OFF)
option(USE_GCC             "build application with gcc"                                         OFF)
option(UNSAFE_FILENAME     "disable safe file naming (if your stdlib doesn't support regex)"    OFF)

if(UNSAFE_FILENAME)
else(UNSAFE_FILENAME)
    set(CMAKE_C_FLAGS "${CMAKE_CXX_FLAGS} -DSAFE_FILENAME")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DSAFE_FILENAME")
endif(UNSAFE_FILENAME)

if(USE_CLANG)
    set(CMAKE_CXX_COMPILER "clang++")
    set(CMAKE_CC_COMPILER  "clang"  )
endif(USE_CLANG)

if(USE_GCC)
    set(CMAKE_CXX_COMPILER "g++")
    set(CMAKE_CC_COMPILER  "gcc")
endif(USE_GCC)
 
set(FMEM_NAME fmem${EXTERNAL_LIB_DIR_SUFFIX})

FetchContent_Declare(
    fmem
    GIT_REPOSITORY https://github.com/kreijstal/fmem.git
    GIT_TAG master
    PREFIX ${DEPS}
    CMAKE_CACHE_ARGS
        -DBUILD_TESTING:BOOL=FALSE
        -DCMAKE_INSTALL_PREFIX:PATH=${DEPS}
        -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
        ${PLATFORM_TOOLCHAIN_OPTION}
        ${CMAKE_OSX_ARCHITECTURES_OPTION}
)

FetchContent_MakeAvailable(fmem)
#set(EXTERNAL_FMEM "fmem")
message(STATUS "Configuring argp")
list(APPEND CMAKE_MESSAGE_INDENT "   ")
message(STATUS "Checking if argp is provided by system libraries")
CHECK_FUNCTION_EXISTS(argp_parse HAVE_BUNDLED_ARGP_PARSE_FUNCTION)

message(STATUS "Looking for standalone argp library")
set(CMAKE_REQUIRED_LIBRARIES argp)
CHECK_FUNCTION_EXISTS(argp_parse HAVE_EXTERNAL_ARGP_PARSE_FUNCTION)
list(POP_BACK CMAKE_MESSAGE_INDENT)
if(HAVE_BUNDLED_ARGP_PARSE_FUNCTION)
  message(STATUS "Using bundled argp")
elseif(HAVE_EXTERNAL_ARGP_PARSE_FUNCTION)
  message(STATUS "Using stand-alone argp")
  set(EXTERNAL_ARGP "argp")
else()
  message(STATUS "Building argp")
  if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(EXTERNAL_ARGP "-Wl,-force_load,${DEPS}/lib/libargp-standalone.a")
  else()
    set(EXTERNAL_ARGP "argp-standalone")
  endif()

  FetchContent_Declare(
    argp_standalone
    GIT_REPOSITORY https://github.com/tom42/argp-standalone.git
    GIT_TAG 238d83d6fb4fbdbb3e0893f51698d8d54696bfb0
    # Optionally specify where to download the content
    # SOURCE_DIR "${CMAKE_BINARY_DIR}/_deps/argp-src"
    # BINARY_DIR "${CMAKE_BINARY_DIR}/_deps/argp-build"
  )
  FetchContent_MakeAvailable(argp_standalone)

  # Assuming argp_standalone creates a target, you can link it directly.
  # If not, you might still need to set up include directories or link libraries manually.
endif(HAVE_BUNDLED_ARGP_PARSE_FUNCTION)


if(DEBUG)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O0 -g -DDEBUG")
    set(CMAKE_BUILD_TYPE Debug)
endif(DEBUG)

if(STATIC)
    set(SHARED "")
    set(CMAKE_FIND_LIBRARY_SUFFIXES ".a")
    set(BUILD_SHARED_LIBRARIES OFF)
    set(CMAKE_EXE_LINKER_FLAGS "-static")
else(STATIC)
    set(SHARED "SHARED")
    set(BUILD_SHARED_LIBRARIES ON)
endif(STATIC)

LINK_DIRECTORIES(
    /usr/local/lib
    /usr/lib/
)

include_directories(
    inc/
    ${LIBEMF2SVG_INCLUDE_DIR}
    ${LIBWMF_INCLUDE_DIR}
    ${LIBREVENGE_INCLUDE_DIR}
    ${LIBVISIO_INCLUDE_DIR}
    ${FREETYPE_INCLUDE_DIRS}
    ${FREETYPE_INCLUDE_DIR_ft2build}
    ${FREETYPE_INCLUDE_DIR_freetype2}
    ${LIBXML2_INCLUDE_DIR}
)

add_executable(vss2svg-conv src/conv/vss2svg.cpp)
add_executable(vsd2svg-conv src/conv/vsd2svg.cpp)

add_library(TitleGenerator
    ${SHARED}
    src/lib/visio2svg/TitleGenerator.cpp
)

add_library(Visio2Svg
    ${SHARED}
    src/lib/visio2svg/Visio2Svg.cpp
)
#add_dependencies(Visio2Svg ${FMEM_NAME})
set_target_properties(Visio2Svg
    PROPERTIES
    VERSION ${vss2svg_VERSION}
    SOVERSION ${vss2svg_VERSION_MAJOR}
)

set_target_properties(TitleGenerator
    PROPERTIES
    VERSION ${vss2svg_VERSION}
    SOVERSION ${vss2svg_VERSION_MAJOR}
)

target_link_libraries(TitleGenerator
    ${LIBREVENGE_LIBRARIES}
    ${LIBVISIO_LIBRARIES}
)

target_link_libraries(Visio2Svg
    ${LIBREVENGE_LIBRARIES}
    ${LIBVISIO_LIBRARIES}
    ${LIBEMF2SVG_LIBRARIES}
    ${LIBWMF_LIBRARIES}
    ${LIBWMFLITE_LIBRARIES}
    ${LIBREVENGE_STREAM_LIBRARY}
    ${LIBXML2_LIBRARIES}
    TitleGenerator
    ${EXTERNAL_ARGP}
    fmem
)

target_link_libraries(vss2svg-conv
    Visio2Svg
)

target_link_libraries(vsd2svg-conv
    Visio2Svg
)

if (NOT LIB_INSTALL_DIR)
    set(LIB_INSTALL_DIR lib)
endif ()

if (NOT BIN_INSTALL_DIR)
    set(BIN_INSTALL_DIR bin)
endif ()

if (NOT INCLUDE_INSTALL_DIR)
    set(INCLUDE_INSTALL_DIR include)
endif ()


SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
INSTALL(TARGETS vss2svg-conv vsd2svg-conv TitleGenerator Visio2Svg
  RUNTIME DESTINATION ${BIN_INSTALL_DIR}
  LIBRARY DESTINATION ${LIB_INSTALL_DIR}
  ARCHIVE DESTINATION ${LIB_INSTALL_DIR}
)

INSTALL(DIRECTORY inc/visio2svg DESTINATION ${INCLUDE_INSTALL_DIR})
