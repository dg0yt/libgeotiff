###############################################################################
#
# CMake main configuration file to build GeoTIFF library and utilities.
#
# Author: Mateusz Loskot <mateusz@loskot.net>
#
###############################################################################
PROJECT(GeoTIFF)

include(GNUInstallDirs)

SET(GEOTIFF_LIB_NAME geotiff)
SET(GEOTIFF_LIBRARY_TARGET geotiff_library)

##############################################################################
# CMake settings
CMAKE_MINIMUM_REQUIRED(VERSION 3.0.0)

SET(CMAKE_COLOR_MAKEFILE ON)

# Version information
set(PROJECT_VERSION_MAJOR 1)
set(PROJECT_VERSION_MINOR 7)
set(PROJECT_VERSION_PATCH 1)
set(PROJECT_VERSION
  "${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH}")
set(GeoTIFF_VERSION ${PROJECT_VERSION})

# Set library version to match that of autoconf:
# libgeotiff.so -> libgeotiff.so.2.2.0
# libgeotiff.so.2 -> libgeotiff.so.2.2.0
# libgeotiff.so.2.2.0
set(LINK_SOVERSION "5")
set(LINK_VERSION "5.2.0")

string (TOLOWER ${PROJECT_NAME} PROJECT_NAME_LOWER)
string (TOUPPER ${PROJECT_NAME} PROJECT_NAME_UPPER)

# Allow advanced users to generate Makefiles printing detailed commands
MARK_AS_ADVANCED(CMAKE_VERBOSE_MAKEFILE)

# Path to additional CMake modules
SET(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

###############################################################################
# General build settings

IF(NOT CMAKE_BUILD_TYPE)
    SET(CMAKE_BUILD_TYPE Debug CACHE STRING
        "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel"
        FORCE)
ENDIF()

SET(GEOTIFF_BUILD_PEDANTIC FALSE CACHE BOOL "Choose compilation in pedantic or relaxed mode")
IF(CMAKE_BUILD_TYPE MATCHES Debug)
    SET(GEOTIFF_BUILD_PEDANTIC TRUE)
ENDIF()

# TODO: Still testing the output paths --mloskot
SET(GEOTIFF_BUILD_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)

# Output directory in which to build RUNTIME target files.
# SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${GEOTIFF_BUILD_OUTPUT_DIRECTORY})
SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)

# Output directory in which to build LIBRARY target files
# SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${GEOTIFF_BUILD_OUTPUT_DIRECTORY})
SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)

# Output directory in which to build ARCHIVE target files.
# SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${GEOTIFF_BUILD_OUTPUT_DIRECTORY})
SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)

###############################################################################
# Platform and compiler specific settings

IF(WIN32)
    IF(BUILD_SHARED_LIBS)
        ADD_DEFINITIONS(-DBUILD_AS_DLL=1)
    ENDIF()
    IF(MSVC)
        ADD_DEFINITIONS(/DW4)
        if (NOT (MSVC_VERSION VERSION_LESS 1400))
            ADD_DEFINITIONS(-D_CRT_SECURE_NO_DEPRECATE)
            ADD_DEFINITIONS(-D_CRT_SECURE_NO_WARNINGS)
            ADD_DEFINITIONS(-D_CRT_NONSTDC_NO_WARNING)
            ADD_DEFINITIONS(-D_SCL_SECURE_NO_WARNINGS)
        endif()
    ENDIF(MSVC)
ENDIF()

IF(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX)
    SET(COMPILE_FLAGS "-fPIC -Wall -Wno-long-long")
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${COMPILE_FLAGS} -std=c99")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${COMPILE_FLAGS} -std=c++98")
    IF(GEOTIFF_BUILD_PEDANTIC)
        SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pedantic")
        SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pedantic")
    ENDIF()
ENDIF()

# Check required standard headers
INCLUDE(CheckIncludeFiles)
CHECK_INCLUDE_FILES(strings.h GEOTIFF_HAVE_STRINGS_H)

###############################################################################
# User-defined build settings

SET(WITH_UTILITIES TRUE CACHE BOOL "Choose if GeoTIFF utilities should be built")

###############################################################################
# Search for dependencies

INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR})
INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR}/libxtiff)

# TIFF support - required, default=ON
SET(WITH_TIFF TRUE CACHE BOOL "Choose if TIFF support should be built")

FIND_PACKAGE(PROJ NO_MODULE QUIET)
if (NOT PROJ_FOUND)
    FIND_PACKAGE(PROJ)
endif ()

IF(PROJ_FOUND)
    INCLUDE_DIRECTORIES(${PROJ_INCLUDE_DIR})
ELSE()
    MESSAGE(FATAL_ERROR "Failed to detect PROJ >= 6")
ENDIF()

# Zlib support - optional, default=OFF
SET(WITH_ZLIB FALSE CACHE BOOL "Choose if zlib support should be built")

IF(WITH_ZLIB)
    FIND_PACKAGE(ZLIB NO_MODULE QUIET)
    if (NOT ZLIB_FOUND)
      FIND_PACKAGE(ZLIB)
    endif ()

    IF(ZLIB_FOUND)
        SET(HAVE_ZIP 1)
        INCLUDE_DIRECTORIES(${ZLIB_INCLUDE_DIR})
        ADD_DEFINITIONS(-DHAVE_ZIP=${HAVE_ZIP})
    ENDIF()
ENDIF()

# JPEG support - optional, default=OFF
SET(WITH_JPEG FALSE CACHE BOOL "Choose if JPEG support should be built")

IF(WITH_JPEG)
    FIND_PACKAGE(JPEG NO_MODULE QUIET)
    if (NOT JPEG_FOUND)
      FIND_PACKAGE(JPEG)
    endif ()

    IF(JPEG_FOUND)
        SET(HAVE_JPEG 1)
        INCLUDE_DIRECTORIES(${JPEG_INCLUDE_DIR})
        ADD_DEFINITIONS(-DHAVE_JPEG=${HAVE_JPEG})
    ENDIF()
ENDIF()

IF(WITH_TIFF)
    FIND_PACKAGE(TIFF NO_MODULE QUIET)
    if (NOT TIFF_FOUND)
      FIND_PACKAGE(TIFF REQUIRED)
    endif ()

    IF(TIFF_FOUND)
        # Confirm required API is available
        INCLUDE(CheckFunctionExists)
        SET(CMAKE_REQUIRED_LIBRARIES ${TIFF_LIBRARIES})

        CHECK_FUNCTION_EXISTS(TIFFOpen HAVE_TIFFOPEN)
        IF(NOT HAVE_TIFFOPEN)
            SET(TIFF_FOUND) # ReSET to NOT found for TIFF library
            MESSAGE(FATAL_ERROR "Failed to link with libtiff - TIFFOpen function not found")
        ENDIF()

        CHECK_FUNCTION_EXISTS(TIFFMergeFieldInfo HAVE_TIFFMERGEFIELDINFO)
        IF(NOT HAVE_TIFFMERGEFIELDINFO)
            SET(TIFF_FOUND) # ReSET to NOT found for TIFF library
            MESSAGE(FATAL_ERROR "Failed to link with libtiff - TIFFMergeFieldInfo function not found. libtiff 3.6.0 Beta or later required. Please upgrade or use an older version of libgeotiff")
        ENDIF()

        INCLUDE_DIRECTORIES(${TIFF_INCLUDE_DIR})
        ADD_DEFINITIONS(-DHAVE_TIFF=1)
    ENDIF(TIFF_FOUND)
ENDIF(WITH_TIFF)

# Turn off TOWGS84 support
SET(WITH_TOWGS84 TRUE CACHE BOOL "Build with TOWGS84 support")
IF (NOT WITH_TOWGS84)
   SET(GEO_NORMALIZE_DISABLE_TOWGS84 1)
endif()

###############################################################################
# Generate geo_config.h with compile-time configuration

MESSAGE(STATUS "Generating geo_config.h header")

CONFIGURE_FILE (${CMAKE_CURRENT_SOURCE_DIR}/cmake/geo_config.h.in
  geo_config.h)
INCLUDE_DIRECTORIES(BEFORE ${CMAKE_CURRENT_BINARY_DIR})

MESSAGE(STATUS "Generating geo_config.h header - done")


###############################################################################
# Installation settings

SET(GEOTIFF_LIB_HEADERS
    geotiff.h
    geotiffio.h
    geovalues.h
    geonames.h
    geokeys.h
    ${CMAKE_CURRENT_BINARY_DIR}/geo_config.h
    geo_tiffp.h
    geo_keyp.h
    geo_normalize.h
    cpl_serv.h
    geo_simpletags.h
    epsg_datum.inc
    epsg_gcs.inc
    epsg_pm.inc
    epsg_units.inc
    geo_ctrans.inc
    epsg_ellipse.inc
    epsg_pcs.inc
    epsg_proj.inc
    epsg_vertcs.inc
    geokeys.inc
    geokeys_v1_1.inc
    libxtiff/xtiffio.h)

SET(GEOTIFF_MAN_PAGES
    man/man1/applygeo.1
    man/man1/geotifcp.1
    man/man1/listgeo.1)

#    ${PROJECT_BINARY_DIR}/geotiff_version.h

# Install doc files
INSTALL(FILES
    AUTHORS ChangeLog COPYING INSTALL LICENSE README README_BIN README.WIN
    DESTINATION ${CMAKE_INSTALL_DOCDIR})

# Install man pages
INSTALL(FILES ${GEOTIFF_MAN_PAGES} DESTINATION ${CMAKE_INSTALL_MANDIR}/man1)


# Install header files for development distribution
INSTALL(FILES ${GEOTIFF_LIB_HEADERS} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

###############################################################################
# Build libxtiff library

ADD_SUBDIRECTORY(libxtiff)

###############################################################################
# Build libgeotiff library

SET(GEOTIFF_LIB_SOURCES
    cpl_serv.c
    geo_extra.c
    geo_free.c
    geo_get.c
    geo_names.c
    geo_new.c
    geo_normalize.c
    geo_print.c
    geo_set.c
    geo_simpletags.c
    geo_tiffp.c
    geo_trans.c
    geo_write.c
    geotiff_proj4.c)

SET(XTIFF_SOURCES libxtiff/xtiff.c)

if (MSVC OR CMAKE_CONFIGURATION_TYPES)
  # For multi-config systems and for Visual Studio, the debug versions
  # of the libraries have a _d suffix.
  set (CMAKE_DEBUG_POSTFIX _d)
endif ()

ADD_LIBRARY(${GEOTIFF_LIBRARY_TARGET}
    ${GEOTIFF_LIB_SOURCES} ${XTIFF_SOURCES})

IF(WIN32 AND MSVC)
    SET_TARGET_PROPERTIES(${GEOTIFF_LIBRARY_TARGET} PROPERTIES IMPORT_SUFFIX "_i.lib")
ENDIF(WIN32 AND MSVC)

# Unix, linux:
if(UNIX)
   SET_TARGET_PROPERTIES(
      ${GEOTIFF_LIBRARY_TARGET}
      PROPERTIES
      VERSION ${LINK_VERSION}
      SOVERSION ${LINK_SOVERSION}
      CLEAN_DIRECT_OUTPUT 1 )
   if (APPLE)
      set_target_properties(
        ${GEOTIFF_LIBRARY_TARGET}
        PROPERTIES
        INSTALL_NAME_DIR "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}")
   endif()
endif()

SET_TARGET_PROPERTIES(${GEOTIFF_LIBRARY_TARGET} PROPERTIES
   OUTPUT_NAME ${GEOTIFF_LIB_NAME})

set(CONFIG_DEPENDENCIES "")
if(TARGET TIFF::TIFF)
    set(TIFF_LIBRARIES TIFF::TIFF)
    string(APPEND CONFIG_DEPENDENCIES "  find_dependency(TIFF)\n")
endif()
target_link_libraries(${GEOTIFF_LIBRARY_TARGET} PUBLIC
    ${TIFF_LIBRARIES})

if(TARGET PROJ::proj)
    set(PROJ_LIBRARIES PROJ::proj)
    string(APPEND CONFIG_DEPENDENCIES "  find_dependency(PROJ CONFIG)\n")
endif()
target_link_libraries(${GEOTIFF_LIBRARY_TARGET} PRIVATE
    ${PROJ_LIBRARIES})

target_include_directories(
  ${GEOTIFF_LIBRARY_TARGET}
  PUBLIC $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}>
         $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)

INSTALL( TARGETS ${GEOTIFF_LIBRARY_TARGET}
     EXPORT depends
     RUNTIME DESTINATION ${CMAKE_INSTALLL_BINDIR}
     LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
     PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
     ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR} )

# Define grouping of source files in PROJECT file (e.g. Visual Studio)
SOURCE_GROUP("CMake Files" FILES CMakeLists.txt)
SOURCE_GROUP("Header Files" FILES ${GEOTIFF_LIB_HEADERS})
SOURCE_GROUP("Source Files" FILES ${GEOTIFF_LIB_SOURCES})

###############################################################################
# Build GeoTIFF utilities

IF(WITH_UTILITIES)
    ADD_SUBDIRECTORY(bin)
ENDIF()

ADD_SUBDIRECTORY(cmake)
