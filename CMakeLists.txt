# Copyright (C) 2017 itemis AG
# This Source Code Form is subject to the terms of the Mozilla Public
# License, v. 2.0. If a copy of the MPL was not distributed with this
# file, You can obtain one at http://mozilla.org/MPL/2.0/.

cmake_minimum_required (VERSION 2.8)

project (CommonAPI-WAMP)

set (CMAKE_VERBOSE_MAKEFILE off)

set (LIBCOMMONAPI_WAMP_MAJOR_VERSION 0)
set (LIBCOMMONAPI_WAMP_MINOR_VERSION 7)
set (LIBCOMMONAPI_WAMP_PATCH_VERSION 0)

message(STATUS "Project name: ${PROJECT_NAME}")

set (COMPONENT_VERSION ${LIBCOMMONAPI_WAMP_MAJOR_VERSION}.${LIBCOMMONAPI_WAMP_MINOR_VERSION}.${LIBCOMMONAPI_WAMP_PATCH_VERSION})
set (COMMONAPI_API_HEADER_VERSION ${LIBCOMMONAPI_WAMP_MAJOR_VERSION}.${LIBCOMMONAPI_WAMP_MINOR_VERSION})

SET(PACKAGE_VERSION "${COMPONENT_VERSION}") # used in *.cmake.in

OPTION(USE_FILE "Set to OFF to disable file logging" OFF )
message(STATUS "USE_FILE is set to value: ${USE_FILE}")

OPTION(USE_CONSOLE "Set to OFF to disable console logging" OFF )
message(STATUS "USE_CONSOLE is set to value: ${USE_CONSOLE}")

SET(DEFAULT_SEND_TIMEOUT "5000" CACHE STRING "default send timeout")
message(STATUS "DEFAULT_SEND_TIMEOUT is set to value: ${DEFAULT_SEND_TIMEOUT} ms")

IF(USE_FILE)
  add_definitions(-DUSE_FILE)
ENDIF(USE_FILE)
IF(USE_CONSOLE)
  add_definitions(-DUSE_CONSOLE)
ENDIF(USE_CONSOLE)

add_definitions(-DDEFAULT_SEND_TIMEOUT=${DEFAULT_SEND_TIMEOUT})

if (MSVC) 
# Boost
find_package( Boost 1.56 COMPONENTS system thread log REQUIRED )
if(Boost_FOUND)
  if(Boost_LIBRARY_DIR)
    MESSAGE( STATUS "Boost_LIBRARY_DIR not empty using it: ${Boost_LIBRARY_DIR}" )
  else()
    if(BOOST_LIBRARYDIR)
      MESSAGE( STATUS "Boost_LIBRARY_DIR empty but BOOST_LIBRARYDIR is set setting Boost_LIBRARY_DIR to: ${BOOST_LIBRARYDIR}" )
      set(Boost_LIBRARY_DIR ${BOOST_LIBRARYDIR})
    endif()
  endif()
else()
  MESSAGE( STATUS "Boost was not found!")
endif()
include_directories( ${Boost_INCLUDE_DIR} )
set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_CRT_SECURE_NO_WARNINGS /wd4503")
link_directories(${Boost_LIBRARY_DIR})
else()
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x -Wall -Wextra -Wformat -Wformat-security -Wconversion -fexceptions -fstrict-aliasing -fstack-protector -fasynchronous-unwind-tables -fno-omit-frame-pointer -DCOMMONAPI_INTERNAL_COMPILATION -D_GLIBCXX_USE_NANOSLEEP")
endif()

SET(MAX_LOG_LEVEL "DEBUG" CACHE STRING "maximum log level")
message(STATUS "MAX_LOG_LEVEL is set to value: ${MAX_LOG_LEVEL}")

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCOMMONAPI_INTERNAL_COMPILATION")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCOMMONAPI_WAMP_VERSION_MAJOR=${LIBCOMMONAPI_WAMP_MAJOR_VERSION}") 
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCOMMONAPI_WAMP_VERSION_MINOR=${LIBCOMMONAPI_WAMP_MINOR_VERSION}") 
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCOMMONAPI_LOGLEVEL=COMMONAPI_LOGLEVEL_${MAX_LOG_LEVEL}")

# Package config module not found message macro
macro (pkg_config_module_not_found_message PKG_CONFIG_MODULE)
    message (FATAL_ERROR "pkg-config could not find the required module ${PKG_CONFIG_MODULE}!"
      " Please adjust your PKG_CONFIG_PATH environment variable accordingly.")
endmacro ()

# BEGIN TEMPORARY WORKAROUND #
# OS
if (${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    set (OS "LINUX")
    set (NO_DEPRECATED_REGISTER "")
endif (${CMAKE_SYSTEM_NAME} MATCHES "Linux")

if (${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD")
    set (OS "FREEBSD")
    set (NO_DEPRECATED_REGISTER "-Wno-deprecated-register")
endif (${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD")

if (NOT MSVC)
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D${OS} ${NO_DEPRECATED_REGISTER} -DBOOST_LOG_DYN_LINK -pthread -g -std=c++0x -O0")
endif()
# END TEMPORARY WORKAROUND #

###################################################################################################
# see http://www.cmake.org/Wiki/CMake/Tutorials/How_to_create_a_ProjectConfig.cmake_file
###################################################################################################

# Offer the user the choice of overriding the installation directories
set (INSTALL_LIB_DIR lib CACHE PATH "Installation directory for libraries")
set (INSTALL_BIN_DIR bin CACHE PATH "Installation directory for executables")
set (INSTALL_INCLUDE_DIR include/CommonAPI-${COMMONAPI_API_HEADER_VERSION} CACHE PATH "Installation directory for header files")

if (WIN32 AND NOT CYGWIN)
    set (DEF_INSTALL_CMAKE_DIR cmake)
else ()
    set (DEF_INSTALL_CMAKE_DIR lib/cmake/CommonAPI-WAMP-${COMPONENT_VERSION})
endif ()

set (INSTALL_CMAKE_DIR ${DEF_INSTALL_CMAKE_DIR} CACHE PATH "Installation directory for CMake files")

# Make relative paths absolute (needed later on)
foreach (p LIB BIN INCLUDE CMAKE)
    set (var INSTALL_${p}_DIR)
    if (NOT IS_ABSOLUTE "${${var}}")
        set (ABSOLUTE_${var} "${CMAKE_INSTALL_PREFIX}/${${var}}")
    endif ()
endforeach ()

###################################################################################################

if ("${USE_INSTALLED_COMMONAPI}" STREQUAL "ON")
    FIND_PACKAGE(CommonAPI 3.1.12 REQUIRED CONFIG NO_CMAKE_PACKAGE_REGISTRY)
else()
    FIND_PACKAGE(CommonAPI 3.1.12 REQUIRED CONFIG NO_SYSTEM_ENVIRONMENT_PATH NO_CMAKE_SYSTEM_PATH)
endif()

message(STATUS "CommonAPI_CONSIDERED_CONFIGS: ${CommonAPI_CONSIDERED_CONFIGS}")
message(STATUS "COMMONAPI_INCLUDE_DIRS: ${COMMONAPI_INCLUDE_DIRS}")
message(STATUS "CommonAPI Version: ${CommonAPI_VERSION}")

# Boost
find_package( Boost 1.54 COMPONENTS system thread log REQUIRED )
include_directories( ${Boost_INCLUDE_DIR} )

# find_package (vsomeip 2.7.0 REQUIRED)
# message(STATUS "vsomeip version: ${vsomeip_VERSION}")

include_directories (
    include
    ${Boost_INCLUDE_DIR}
    ${COMMONAPI_INCLUDE_DIRS}
)

file (GLOB CommonAPI-WAMP_SRC "src/CommonAPI/Wamp/*.cpp")
list (SORT CommonAPI-WAMP_SRC
)

# CommonAPI
add_library (CommonAPI-WAMP SHARED ${CommonAPI-WAMP_SRC})
set_target_properties (CommonAPI-WAMP PROPERTIES VERSION ${COMPONENT_VERSION} SOVERSION ${COMPONENT_VERSION})
#target_link_libraries (CommonAPI-WAMP CommonAPI vsomeip)
target_link_libraries (CommonAPI-WAMP CommonAPI boost_thread)

if (MSVC)
    target_link_libraries(CommonAPI-WAMP ws2_32 Rpcrt4)
endif()

###################################################################################################

file (GLOB_RECURSE CommonAPI-WAMP_INCLUDE_INSTALL_FILES "include/*.hpp")
list (SORT CommonAPI-WAMP_INCLUDE_INSTALL_FILES)

set_target_properties (CommonAPI-WAMP PROPERTIES PUBLIC_HEADER "${CommonAPI-WAMP_INCLUDE_INSTALL_FILES}")

install (
    TARGETS CommonAPI-WAMP
    # IMPORTANT: Add the CommonAPI-WAMP library to the "export-set"
    EXPORT CommonAPI-WAMPTargets
	LIBRARY DESTINATION ${INSTALL_LIB_DIR}
	RUNTIME DESTINATION ${INSTALL_BIN_DIR}
	ARCHIVE DESTINATION ${INSTALL_LIB_DIR}
    PUBLIC_HEADER DESTINATION "${INSTALL_INCLUDE_DIR}/CommonAPI/Wamp"
)

##############################################################################
# exporting, configuring and installing of cmake files

# Add all targets to the build-tree export set
export (TARGETS CommonAPI-WAMP
        FILE "${PROJECT_BINARY_DIR}/CommonAPI-WAMPTargets.cmake")

# Export the package for use from the build-tree
# (this registers the build-tree with a global CMake-registry)
export (PACKAGE CommonAPI-WAMP)

# Create the CommonAPI-WAMPConfig.cmake and CommonAPI-WAMPConfigVersion files
file (RELATIVE_PATH REL_INCLUDE_DIR "${ABSOLUTE_INSTALL_CMAKE_DIR}" "${ABSOLUTE_INSTALL_INCLUDE_DIR}")

# ... for the build tree
set (CONF_INCLUDE_DIRS "${PROJECT_SOURCE_DIR}/include")
configure_file (${CMAKE_CURRENT_SOURCE_DIR}/cmake/CommonAPI-WAMPConfig.cmake.in
    "${PROJECT_BINARY_DIR}/CommonAPI-WAMPConfig.cmake" @ONLY)

# ... for the install tree
set (CONF_INCLUDE_DIRS "\${COMMONAPI_WAMP_CMAKE_DIR}/${REL_INCLUDE_DIR}")
configure_file (${CMAKE_CURRENT_SOURCE_DIR}/cmake/CommonAPI-WAMPConfig.cmake.in
    "${PROJECT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/CommonAPI-WAMPConfig.cmake" @ONLY)

# ... for both
configure_file (${CMAKE_CURRENT_SOURCE_DIR}/cmake/CommonAPI-WAMPConfigVersion.cmake.in
    "${PROJECT_BINARY_DIR}/CommonAPI-WAMPConfigVersion.cmake" @ONLY)

# Install the CommonAPI-WAMPConfig.cmake and CommonAPI-WAMPConfigVersion.cmake
install (
    FILES
    "${PROJECT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/CommonAPI-WAMPConfig.cmake"
    "${PROJECT_BINARY_DIR}/CommonAPI-WAMPConfigVersion.cmake"
    DESTINATION "${INSTALL_CMAKE_DIR}"
)

# Install the export set for use with the install-tree
install (
    EXPORT CommonAPI-WAMPTargets
    DESTINATION "${INSTALL_CMAKE_DIR}"
)

##############################################################################
# create pkg-config file
if(NOT WIN32)
        configure_file(CommonAPI-WAMP.pc.in ${PROJECT_BINARY_DIR}/CommonAPI-WAMP.pc @ONLY)
        install(FILES ${PROJECT_BINARY_DIR}/CommonAPI-WAMP.pc DESTINATION lib/pkgconfig)
endif()

##############################################################################
# maintainer-clean
add_custom_target(maintainer-clean COMMAND rm -rf *)
