cmake_minimum_required(VERSION 3.9)
cmake_policy(SET CMP0042 NEW)
# project(MyApplication)
project (corrade-js)

set(CORRADEJS_MAJOR_VERSION 0)
set(CORRADEJS_MINOR_VERSION 1)
set(CORRADEJS_PATCH_VERSION 0)
set(CORRADEJS_VERSION
  ${CORRADEJS_MAJOR_VERSION}.${CORRADEJS_MINOR_VERSION}.${CORRADEJS_PATCH_VERSION})

  # Offer the user the choice of overriding the installation directories
set(INSTALL_LIB_DIR lib CACHE PATH "Installation directory for libraries")
set(INSTALL_BIN_DIR bin CACHE PATH "Installation directory for executables")
set(INSTALL_INCLUDE_DIR include CACHE PATH
  "Installation directory for header files")

if(WIN32 AND NOT CYGWIN)
  set(DEF_INSTALL_CMAKE_DIR cmake)
else()
  set(DEF_INSTALL_CMAKE_DIR lib/cmake/MinMagApi)
endif()
set(INSTALL_CMAKE_DIR ${DEF_INSTALL_CMAKE_DIR} CACHE PATH
  "Installation directory for CMake files")

# Make relative paths absolute (needed later on)
foreach(p LIB BIN INCLUDE CMAKE)
set(var INSTALL_${p}_DIR)
if(NOT IS_ABSOLUTE "${${var}}")
  set(${var} "${CMAKE_INSTALL_PREFIX}/${${var}}")
endif()
endforeach()

set (CMAKE_CXX_STANDARD 14)
include_directories(${CMAKE_JS_INC})

# FetchContent added in CMake 3.11, downloads during the configure step
include(FetchContent)
# FetchContent_MakeAvailable was not added until CMake 3.14; use our shim
if(${CMAKE_VERSION} VERSION_LESS 3.14)
    include(cmake/add_FetchContent_MakeAvailable.cmake)
endif()

add_library(${PROJECT_NAME} SHARED) 
set_target_properties(${PROJECT_NAME} PROPERTIES PREFIX "" SUFFIX ".node")
set(CMAKE_INSTALL_PREFIX ${CMAKE_SOURCE_DIR}/bin)

FetchContent_Declare(
  corrade
  GIT_REPOSITORY https://github.com/mosra/corrade.git
  GIT_TAG aa69de7edd60d3f20ebe7e4c9e4e137890d22851
)

FetchContent_GetProperties(corrade)
if(NOT corrade_POPULATED)
  message(STATUS "Fetching CORRADE v${CORRADE_VERSION}")

  FetchContent_Populate(corrade)
  set(WITH_INTERCONNECT TRUE CACHE BOOL "" FORCE)
  set(WITH_PLUGINMANAGER  TRUE CACHE BOOL "" FORCE)
  set(WITH_UTILITY   TRUE CACHE BOOL "" FORCE)
  set(WITH_RC    TRUE CACHE BOOL "" FORCE)

  set(BUILD_MULTITHREADED TRUE CACHE BOOL "" FORCE)
  set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CORRADE_SOURCE_DIR}/modules)
  add_subdirectory(${corrade_SOURCE_DIR})# ${corrade_BINARY_DIR})
  # configure_file(${corrade_BINARY_DIR}/src/Corrade/configure.h ${corrade_SOURCE_DIR}/src/Corrade/configure.h COPYONLY)
  # we copy the configure.h from build to source 

  set(CORRADE_INCLUDE_DIR 
    ${corrade_SOURCE_DIR}/src
    ${corrade_BINARY_DIR}/src
    )
endif()

target_compile_definitions(
        ${PROJECT_NAME}
        PRIVATE
            # ENTT_ID_TYPE=${ENTT_ID_TYPE}
            # _ENABLE_EXTENDED_ALIGNED_STORAGE
            NOMINMAX
            # ENTT_API_EXPORT
    )

file(GLOB SRC_MAIN
    "src/lib/*.cpp")

target_sources(${PROJECT_NAME} PRIVATE
    ${SRC_MAIN}
    ${EASYLOGGING}
    ${EMPTY_RESOURCES}
    ${CMAKE_JS_SRC}
    )

target_link_libraries(${PROJECT_NAME} PRIVATE
    ${CMAKE_JS_LIB} 
    ${CORRADE_LIBRARIES}
) 

# Include Node-API wrappers
execute_process(COMMAND node -p "require('node-addon-api').include"
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE NODE_ADDON_API_DIR
        )

string(REPLACE "\n" "" NODE_ADDON_API_DIR ${NODE_ADDON_API_DIR})
string(REPLACE "\"" "" NODE_ADDON_API_DIR ${NODE_ADDON_API_DIR})
string(REGEX REPLACE "[\r\n\"]" "" NODE_ADDON_API_DIR ${NODE_ADDON_API_DIR})

target_include_directories(${PROJECT_NAME} PRIVATE 
    ./
    src
    ../external/cr
    ${CORRADE_INCLUDE_DIR}
    ${MAGNUM_INCLUDE_DIR}
    ${NODE_ADDON_API_DIR})

# define NPI_VERSION
add_definitions(-DNAPI_VERSION=7)
