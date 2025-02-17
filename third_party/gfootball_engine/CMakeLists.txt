cmake_minimum_required(VERSION 3.4)

if(WIN32)
  # Check for required Environment variables
  if(NOT DEFINED CMAKE_GENERATOR_PLATFORM)
    message(FATAL_ERROR "CMAKE_GENERATOR_PLATFORM is not defined. Add -Ax64 or -AWin32 to cmake command")
  elseif(NOT DEFINED ENV{VCPKG_ROOT})
    message(FATAL_ERROR "VCPKG_ROOT is not defined. Set the location of vcpkg")
  elseif(NOT DEFINED ENV{PY_VERSION})  # PY_VERSION has to be X.Y, e.g. 3.9
    message(FATAL_ERROR "PY_VERSION is not defined. Define Python version that will be used, e.g. 3.9")
  endif()

  if(${CMAKE_GENERATOR_PLATFORM} STREQUAL Win32)
    set(VCPKG_TARGET_TRIPLET x86-windows)
  elseif(${CMAKE_GENERATOR_PLATFORM} STREQUAL x64)
    set(VCPKG_TARGET_TRIPLET x64-windows)
  else()
    message(FATAL_ERROR "Unsupported Generator Platform, only Win32 or x64 are supported")
  endif()

  set(CMAKE_TOOLCHAIN_FILE "$ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake")
  set(VCPKG_MANIFEST_DIR "${CMAKE_SOURCE_DIR}/vcpkg_manifests/py$ENV{PY_VERSION}")
  set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
else()  # UNIX
  set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/CMakeModules")
endif()

project(gameplayfootball)
set(PACKAGE gameplayfootball)

set (CMAKE_CXX_STANDARD 14)

if(UNIX)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${GCC_COVERAGE_COMPILE_FLAGS} -fPIC -g -O3")
  SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC -O3 -g")
endif(UNIX)

# Find needed libraries
FIND_PACKAGE(OpenGL REQUIRED)
include_directories(${OPENGL_INCLUDE_DIR})

FIND_PACKAGE(SDL2 REQUIRED)
include_directories(${SDL2_INCLUDE_DIR})

if(UNIX AND NOT APPLE)
  FIND_PACKAGE(EGL REQUIRED)
  include_directories(${EGL_INCLUDE_DIR})
endif()

if(WIN32)
  FIND_PACKAGE(SDL2-image REQUIRED)
  FIND_PACKAGE(SDL2-ttf REQUIRED)
  FIND_PACKAGE(SDL2-gfx REQUIRED)
else()  # Unix
  FIND_PACKAGE(SDL2_image REQUIRED)
  FIND_PACKAGE(SDL2_ttf REQUIRED)
  FIND_PACKAGE(SDL2_gfx REQUIRED)
endif()

include_directories(${SDL2_IMAGE_DIRS})
include_directories(${SDL2_TTF_DIRS})
include_directories(${SDL2_GFX_DIRS})

if(UNIX)
  find_package(PythonLibs 3 REQUIRED)
else()
  set(PYTHONLIBS_VERSION_STRING $ENV{PY_VERSION})
endif()

message("Using Python: ${PYTHONLIBS_VERSION_STRING}")
string(REPLACE "." ";" VERSION_LIST ${PYTHONLIBS_VERSION_STRING})
list(GET VERSION_LIST 0 PYTHON_MAJOR)
list(GET VERSION_LIST 1 PYTHON_MINOR)

if(WIN32)
  # find_package (Python COMPONENTS Development)
  # FindPython package searches for system-wide python installations first.
  # It was improved only recently (Jan 2021), so we can't use it for earlier py versions:
  # (see https://github.com/microsoft/vcpkg/pull/15221 )
  # To avoid version mismatch we point to vcpkg's libs directly
  set(PYTHON_LIBRARIES "${CMAKE_BINARY_DIR}/vcpkg_installed/${VCPKG_TARGET_TRIPLET}/lib/python${PYTHON_MAJOR}${PYTHON_MINOR}.lib")
  set(PYTHON_INCLUDE_DIR "${CMAKE_BINARY_DIR}/vcpkg_installed/${VCPKG_TARGET_TRIPLET}/include/python${PYTHON_MAJOR}.${PYTHON_MINOR}")
endif()

# Include Boost Python.
if(WIN32 OR APPLE)
  set(BOOST_PYTHON_VERSION python${PYTHON_MAJOR}${PYTHON_MINOR})
else()
  # Unfortunately different linux distributions are using different naming for boost-python
  # list of prefixes: Ubuntu, Debian, Some other distributions
  set(BOOST_PYTHON_PREFIXES python3-py python-py python)
  foreach(BP_PREFIX ${BOOST_PYTHON_PREFIXES})
    set(BOOST_PYTHON_NAME ${BP_PREFIX}${PYTHON_MAJOR}${PYTHON_MINOR})
    string(TOUPPER ${BOOST_PYTHON_NAME} BOOST_PYTHON_NAME_UPPER)
    FIND_PACKAGE(Boost COMPONENTS ${BOOST_PYTHON_NAME})
    if(Boost_${BOOST_PYTHON_NAME_UPPER}_FOUND)
      set(BOOST_PYTHON_VERSION ${BOOST_PYTHON_NAME})
      break()
    endif()
    message("${BOOST_PYTHON_NAME} not found. Trying other names.")
  endforeach()
  if(NOT DEFINED BOOST_PYTHON_VERSION)
    message(FATAL_ERROR "Boost Python not found")
  endif()
endif()
message("Using python_boost: ${BOOST_PYTHON_VERSION}")

FIND_PACKAGE(Boost REQUIRED COMPONENTS thread system filesystem ${BOOST_PYTHON_VERSION})
include_directories(${Boost_INCLUDE_DIR})

include_directories(${PYTHON_INCLUDE_DIR})

include_directories(${PROJECT_SOURCE_DIR}/src)
include_directories(${PROJECT_SOURCE_DIR}/src/cmake)

# Include the sources
include(sources.cmake)

add_library(baselib OBJECT ${BASE_SOURCES} ${BASE_HEADERS}
   ${BASE_GEOMETRY_HEADERS} ${BASE_MATH_HEADERS})
add_library(systemsgraphicslib OBJECT ${SYSTEMS_GRAPHICS_SOURCES} ${SYSTEMS_COMMON_HEADERS}
   ${SYSTEMS_GRAPHICS_HEADERS} ${SYSTEMS_GRAPHICS_OBJECTS_HEADERS}
   ${SYSTEMS_GRAPHICS_RESOURCES_HEADERS} ${SYSTEMS_GRAPHICS_RENDERING_HEADERS})
add_library(loaderslib OBJECT ${LOADERS_SOURCES} ${LOADERS_HEADERS} ${MANAGERS_HEADERS})
add_library(typeslib OBJECT ${TYPES_SOURCES} ${TYPES_HEADERS})
add_library(scenelib OBJECT ${SCENE_SOURCES} ${SCENE_HEADERS}
   ${SCENE2D_HEADERS} ${SCENE3D_HEADERS} ${SCENE_OBJECTS_HEADERS}
   ${SCENE_RESOURCES_HEADERS})
add_library(utilslib OBJECT ${UTILS_SOURCES} ${UTILS_HEADERS}
   ${UTILS_EXT_HEADERS})
add_library(gui2lib OBJECT ${UTILS_GUI2_SOURCES} ${UTILS_GUI2_HEADERS}
   ${UTILS_GUI2_WIDGETS_HEADERS})

# Join all created static libraries into a single static or shared one.
set(OWN_LIBRARIES $<TARGET_OBJECTS:baselib> # $<TARGET_OBJECTS:systemscommonlib>
   $<TARGET_OBJECTS:systemsgraphicslib> $<TARGET_OBJECTS:loaderslib>
   $<TARGET_OBJECTS:typeslib> $<TARGET_OBJECTS:scenelib>
   $<TARGET_OBJECTS:utilslib> $<TARGET_OBJECTS:gui2lib>)

# Compile it as multiple static libraries
add_library(blunted2 ${BLUNTED_CORE_SOURCES} ${BLUNTED_CORE_HEADERS} ${OWN_LIBRARIES})
add_library(gamelib ${GAME_SOURCES} ${GAME_HEADERS})
add_library(menulib ${MENU_SOURCES} ${MENU_HEADERS})
add_library(datalib ${DATA_SOURCES} ${DATA_HEADERS})

if(UNIX)
  set(SDL2_LIBS_ALL ${SDL2_IMAGE_LIBRARIES} ${SDL2_TTF_LIBRARIES} ${SDL2_GFX_LIBRARIES} ${SDL2_LIBRARIES})
elseif(WIN32)
  set(SDL2_LIBS_ALL SDL2::SDL2 SDL2::SDL2_image SDL2::SDL2_ttf SDL2::SDL2_gfx SDL2::SDL2main)
endif()

set(LIBRARIES gamelib menulib datalib blunted2
   Boost::filesystem Boost::system Boost::thread Boost::${BOOST_PYTHON_VERSION}
   ${PYTHON_LIBRARIES} ${SDL2_LIBS_ALL} ${EGL_LIBRARIES} ${OPENGL_LIBRARY})

if(UNIX)
  set(LIB_NAME game)
else()  # WIN32
  # Windows requires the library to have the same name as the module
  set(LIB_NAME _gameplayfootball)
endif()

add_library(${LIB_NAME} SHARED ${CORE_SOURCES} ${CORE_HEADERS} ${AI_HEADERS} ${AI_SOURCES})
target_link_libraries(${LIB_NAME} ${LIBRARIES})

if(WIN32)
  set_target_properties(${LIB_NAME} PROPERTIES SUFFIX ".pyd")
endif()
