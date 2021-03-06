cmake_minimum_required(VERSION 2.8)

project("opencv-plugin-sample")
message(STATUS "CMake project: ${PROJECT_NAME}")

# Test configuration
set(GENERATE_TESTS FALSE CACHE BOOL "Always build tests: add `make check_build` to normal `make` calls")
set(DISABLE_TESTS FALSE CACHE BOOL "Enable running `make check` during the building process")
set(VALGRIND_NUM_CALLERS 20 CACHE STRING "Valgrind option: maximum number of entries shown in stack traces")

message("If KurentoHelpers is not found, you need to install 'kms-cmake-utils' from the Kurento repository")
find_package(KurentoHelpers REQUIRED)

message("If CodeGenerator is not found, you need to install 'kms-core' from the Kurento repository")
include(CodeGenerator)
get_values_from_model(PREFIX VALUE MODELS ${CMAKE_CURRENT_SOURCE_DIR}/src/server/interface KEYS version)

include(KurentoGitHelpers)
install_git_hook(pre-commit ${CMAKE_CURRENT_SOURCE_DIR}/hooks/pre-commit.hook)
get_git_version(PROJECT_VERSION ${VALUE_VERSION})
message(STATUS "Project version: ${PROJECT_NAME}-${PROJECT_VERSION}")

# Set GCC flags common to all projects
# Note: Flags added from dpkg-buildflags (if installed):
# -g -O2 -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2
include(DpkgBuildFlags)
get_dpkg_buildflags_c(DPKG_BUILDFLAGS_C)
get_dpkg_buildflags_cxx(DPKG_BUILDFLAGS_CXX)
set(CMAKE_C_FLAGS   "${CMAKE_C_FLAGS}   ${DPKG_BUILDFLAGS_C}   -std=c11   -Wall -Werror -pthread")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${DPKG_BUILDFLAGS_CXX} -std=c++11 -Wall -Werror -pthread")
set(CMAKE_C_FLAGS_DEBUG   "-g -O0") # FIXME Ideal is '-Og' but a bug in GCC prevents this,
set(CMAKE_CXX_FLAGS_DEBUG "-g -O0") # causing "may be used uninitialized" errors: https://gcc.gnu.org/bugzilla/show_bug.cgi?id=58455
set(CMAKE_C_FLAGS_RELEASE   "-DNDEBUG -O2") # CMake adds '-O3' by default
set(CMAKE_CXX_FLAGS_RELEASE "-DNDEBUG -O2") # but currently only '-O2' is tested
set(CMAKE_POSITION_INDEPENDENT_CODE ON) # Use "-fPIC" for all targets, including static libs

# FIXME Disable error when macros __TIME__, __DATE__ or __TIMESTAMP__ are encountered
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-Wno-error=date-time" HAS_WARNING_DATE_TIME)
if(HAS_WARNING_DATE_TIME)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-error=date-time")
endif()

# Generate file "config.h"
set(VERSION ${PROJECT_VERSION})
set(PACKAGE ${PROJECT_NAME})
set(GETTEXT_PACKAGE "opencv-plugin-sample")
include(GNUInstallDirs)
set(KURENTO_MODULES_SO_DIR ${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}/${KURENTO_MODULES_DIR_INSTALL_PREFIX})
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/config.h)
set(CMAKE_C_FLAGS   "${CMAKE_C_FLAGS}   -DHAVE_CONFIG_H")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DHAVE_CONFIG_H")
include_directories(${CMAKE_CURRENT_BINARY_DIR})

set(GST_REQUIRED 1.5.0)
set(GLIB_REQUIRED 2.38)
set(OPENCV_REQUIRED 2.0.0)

find_package(PkgConfig)
pkg_check_modules(GSTREAMER REQUIRED gstreamer-1.5>=${GST_REQUIRED})
pkg_check_modules(GSTREAMER_BASE REQUIRED gstreamer-base-1.5>=${GST_REQUIRED})
pkg_check_modules(GSTREAMER_VIDEO REQUIRED gstreamer-video-1.5>=${GST_REQUIRED})
pkg_check_modules(GSTREAMER_CHECK REQUIRED gstreamer-check-1.5>=${GST_REQUIRED})
pkg_check_modules(KMSCORE REQUIRED kmscore)
pkg_check_modules(OPENCV REQUIRED opencv>=${OPENCV_REQUIRED})

set(CMAKE_INSTALL_GST_PLUGINS_DIR ${CMAKE_INSTALL_LIBDIR}/gstreamer-1.5)

add_subdirectory(src)
