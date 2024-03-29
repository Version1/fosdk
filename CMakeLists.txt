cmake_minimum_required(VERSION 2.8.0)
project(fosdk CXX)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(SUBMODULE_FLAGS "")

if(PYTHON_LIBRARY AND PYTHON_INCLUDE_DIR AND PYTHON_EXECUTABLE)
    set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DPYTHON_LIBRARY=${PYTHON_LIBRARY})
    set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DPYTHON_INCLUDE_DIR=${PYTHON_INCLUDE_DIR})
    set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DPYTHON_EXECUTABLE=${PYTHON_EXECUTABLE})
endif()

if(SWIG_EXECUTABLE)
    set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DSWIG_EXECUTABLE=${SWIG_EXECUTABLE})
endif()

if(FLEX_EXECUTABLE AND FLEX_INCLUDE_DIR)
    set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DFLEX_EXECUTABLE=${FLEX_EXECUTABLE})
    set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DFLEX_INCLUDE_DIR=${FLEX_INCLUDE_DIR})
endif()

if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set (CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}/install" CACHE PATH "default install path" FORCE )
endif()

if(WIN32)
  set(CMAKE_SHARED_LINKER_FLAGS "/FORCE:UNRESOLVED")
  set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
  set(BUILD_SHARED_LIBS TRUE)
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MD")
  set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MDd")
endif()


if(APPLE)
  set(CMAKE_XCODE_ATTRIBUTE_CLANG_CXX_LIBRARY "libc++")
  set(CMAKE_XCODE_ATTRIBUTE_GCC_VERSION "com.apple.compilers.llvm.clang.1_0")
  set(CMAKE_SHARED_LINKER_FLAGS "-Wl,-undefined,dynamic_lookup")
  set(CMAKE_MACOSX_RPATH 1)
endif()

option(DEBUG "Enable debug build" OFF)
option(JAVA "Enable Java bindings" OFF)
option(PYTHON "Enable Python bindings" OFF)
option(CSHARP "Enable C# bindings" OFF)
option(COVERAGE "Enable gcov coverage" OFF)
option(EXAMPLES "Enable compilation of examples" OFF)
option(CDR_JSON_SUPPORT "Enable cdr json support" OFF)
set(PYTHON_CONFIG "python2-config" CACHE STRING "python-config for build env config")

# set version info
set(FOSDK_MAJOR_VERSION 0)
set(FOSDK_MINOR_VERSION 1)
set(FOSDK_PATCH_VERSION 0)
set(FOSDK_VERSION ${FOSDK_MAJOR_VERSION}.${FOSDK_MINOR_VERSION}.${FOSDK_PATCH_VERSION})

if (UNIX)
  macro (add_library _name)
    _add_library(${ARGV})
    if (NOT APPLE AND TARGET ${_name})
      SET_TARGET_PROPERTIES(${_name}
        PROPERTIES
        VERSION ${FOSDK_VERSION}
            # SOVERSION 0.0.0
        )
    endif()
  endmacro()
endif (UNIX)

add_compile_options(-Wall)
if (DEBUG)
  message(STATUS "fosdk DEBUG: ON")
  add_compile_options(-g3)
  set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DDEBUG=ON)
else ()
  message(STATUS "fosdk DEBUG: OFF")
  add_compile_options(-g -O2)
endif (DEBUG)

if (JAVA)
  message(STATUS "JAVA: ON")
  set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DJAVA=ON)
else ()
  message(STATUS "JAVA: OFF")
  set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DJAVA=OFF)
endif()
if (PYTHON)
  message(STATUS "PYTHON: ON")
  set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DPYTHON=ON -DPYTHON_CONFIG=${PYTHON_CONFIG})
else ()
  message(STATUS "PYTHON: OFF")
  set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DPYTHON=OFF)
endif()
if (CSHARP)
  message(STATUS "CSHARP: ON")
  set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DCSHARP=ON)
else ()
  message(STATUS "CSHARP: OFF")
  set(SUBMODULE_FLAGS ${SUBMODULE_FLAGS} -DCSHARP=OFF)
endif()

# allow external projects
include(${CMAKE_ROOT}/Modules/ExternalProject.cmake)

if (COVERAGE)
    set(COVERAGE_COMPILE_FLAGS "-g -O0 -coverage -fprofile-arcs -ftest-coverage")
    set(COVERAGE_LINK_FLAGS    "-coverage -lgcov")
    set(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} ${COVERAGE_COMPILE_FLAGS}" )
    set(CMAKE_EXE_LINKER_FLAGS  "${CMAKE_EXE_LINKER_FLAGS} ${COVERAGE_LINK_FLAGS}" )
endif(COVERAGE)

# deps
if(NOT SUBMODULE_ROOT_DIR)
    set(SUBMODULE_ROOT_DIR ${PROJECT_SOURCE_DIR})
endif()
if(NOT SUBMODULE_PREFIX_DIR)
    set(SUBMODULE_PREFIX_DIR ${CMAKE_BINARY_DIR})
endif()

find_library(EVENT_LIB
  NAMES event
  HINTS "${CMAKE_INSTALL_PREFIX}/lib"
)
find_library(EVENT_PTHREAD_LIB
  NAMES event_pthreads
  HINTS "${CMAKE_INSTALL_PREFIX}/lib"
)
if(EVENT_LIB AND EVENT_PTHREAD_LIB)
  message(STATUS "event-found: " ${EVENT_LIB})
  message(STATUS "event-pthread-found: " ${EVENT_PTHREAD_LIB})
 else()
  set(EVENT_CMAKE ${SUBMODULE_ROOT_DIR}/ext/libevent CACHE PATH "Location of libevent-cmake" FORCE)
  message(STATUS "libevent-location: " ${EVENT_CMAKE})
  ExternalProject_Add(LIBEVENT_PROJECT
      PREFIX ${SUBMODULE_PREFIX_DIR}
      SOURCE_DIR ${PROJECT_SOURCE_DIR}/ext/libevent
      CMAKE_ARGS -DEVENT__DISABLE_TESTS=ON -DEVENT__DISABLE_REGRESS=ON -DEVENT__DISABLE_OPENSSL=ON -DEVENT__DISABLE_SAMPLES=ON -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX})
  set(LIBEVENT_PROJECT_DIR ${binary_dir}/lib/cmake/libevent CACHE PATH "location of libevent cmake files" FORCE)
endif()

#sbf
set(SBF_CMAKE ${SUBMODULE_ROOT_DIR}/ext/sbf CACHE PATH "Location of sbf-cmake" FORCE)
message(STATUS "sbf-location: " ${SBF_CMAKE})
ExternalProject_Add(
  SBF_PROJECT
  PREFIX ${SUBMODULE_PREFIX_DIR}
  SOURCE_DIR ${SBF_CMAKE}
  CMAKE_ARGS -DSUBMODULE_PREFIX_DIR=${SUBMODULE_PREFIX_DIR} -DSUBMODULE_ROOT_DIR=${SUBMODULE_ROOT_DIR} -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} ${SUBMODULE_FLAGS}
  )
ExternalProject_Get_Property(SBF_PROJECT binary_dir)
set(SBF_PROJECT_DIR ${binary_dir}/lib/cmake/sbf CACHE PATH "location of sbf cmake files" FORCE)
# needs libevent
if (NOT EVENT_LIB)
  add_dependencies(SBF_PROJECT LIBEVENT_PROJECT)
endif()
set(CPACK_INSTALL_CMAKE_PROJECTS ${CPACK_INSTALL_CMAKE_PROJECTS} "${binary_dir};sbf;ALL;/")

#cdr
set(CDR_CMAKE ${SUBMODULE_ROOT_DIR}/ext/cdr CACHE PATH "Location of cdr-cmake" FORCE)
message(STATUS "cdr-location: " ${CDR_CMAKE})
ExternalProject_Add(
  CDR_PROJECT
  PREFIX ${SUBMODULE_PREFIX_DIR}
  SOURCE_DIR ${CDR_CMAKE}
  CMAKE_ARGS -DSUBMODULE_PREFIX_DIR=${SUBMODULE_PREFIX_DIR} -DSUBMODULE_ROOT_DIR=${SUBMODULE_ROOT_DIR} -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} -DJSON=${CDR_JSON_SUPPORT} ${SUBMODULE_FLAGS}
  )
ExternalProject_Get_Property(CDR_PROJECT binary_dir)
set(CDR_PROJECT_DIR ${binary_dir}/lib/cmake/cdr CACHE PATH "location of cdr cmake files" FORCE)
set(CPACK_INSTALL_CMAKE_PROJECTS ${CPACK_INSTALL_CMAKE_PROJECTS} "${binary_dir};cdr;ALL;/")

#utils
set(UTILS_CMAKE ${SUBMODULE_ROOT_DIR}/ext/utils CACHE PATH "Location of utils-cmake" FORCE)
message(STATUS "utils-location: " ${UTILS_CMAKE})
ExternalProject_Add(
  UTILS_PROJECT
  PREFIX ${SUBMODULE_PREFIX_DIR}
  SOURCE_DIR ${UTILS_CMAKE}
  CMAKE_ARGS -DSUBMODULE_PREFIX_DIR=${SUBMODULE_PREFIX_DIR} -DSUBMODULE_ROOT_DIR=${SUBMODULE_ROOT_DIR} -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} ${SUBMODULE_FLAGS}
  )
ExternalProject_Get_Property(UTILS_PROJECT binary_dir)
set(UTILS_PROJECT_DIR ${binary_dir}/lib/cmake/utils CACHE PATH "location of utils cmake files" FORCE)
set(CPACK_INSTALL_CMAKE_PROJECTS ${CPACK_INSTALL_CMAKE_PROJECTS} "${binary_dir};utils;ALL;/")
add_dependencies(UTILS_PROJECT SBF_PROJECT)

#properties
set(PROPERTIES_CMAKE ${SUBMODULE_ROOT_DIR}/ext/properties CACHE PATH "Location of properties-cmake" FORCE)
message(STATUS "properties-location: " ${PROPERTIES_CMAKE})
ExternalProject_Add(
  PROPERTIES_PROJECT
  PREFIX ${SUBMODULE_PREFIX_DIR}
  SOURCE_DIR ${PROPERTIES_CMAKE}
  CMAKE_ARGS -DSUBMODULE_PREFIX_DIR=${SUBMODULE_PREFIX_DIR} -DSUBMODULE_ROOT_DIR=${SUBMODULE_ROOT_DIR} -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} ${SUBMODULE_FLAGS}
  )
ExternalProject_Get_Property(PROPERTIES_PROJECT binary_dir)
set(PROPERTIES_PROJECT_DIR ${binary_dir}/lib/cmake/config CACHE PATH "location of properties cmake files" FORCE)
set(CPACK_INSTALL_CMAKE_PROJECTS ${CPACK_INSTALL_CMAKE_PROJECTS} "${binary_dir};properties;ALL;/")
add_dependencies(PROPERTIES_PROJECT UTILS_PROJECT)

#logger
set(LOGGER_CMAKE ${SUBMODULE_ROOT_DIR}/ext/logger CACHE PATH "Location of logger-cmake" FORCE)
message(STATUS "logger-location: " ${LOGGER_CMAKE})
ExternalProject_Add(
  LOGGER_PROJECT
  PREFIX ${SUBMODULE_PREFIX_DIR}
  SOURCE_DIR ${LOGGER_CMAKE}
  CMAKE_ARGS -DSUBMODULE_PREFIX_DIR=${SUBMODULE_PREFIX_DIR} -DSUBMODULE_ROOT_DIR=${SUBMODULE_ROOT_DIR} -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} ${SUBMODULE_FLAGS}
  )
ExternalProject_Get_Property(LOGGER_PROJECT binary_dir)
set(LOGGER_PROJECT_DIR ${binary_dir}/lib/cmake/logger CACHE PATH "location of logger cmake files" FORCE)
set(CPACK_INSTALL_CMAKE_PROJECTS ${CPACK_INSTALL_CMAKE_PROJECTS} "${binary_dir};logger;ALL;/")
add_dependencies(LOGGER_PROJECT SBF_PROJECT PROPERTIES_PROJECT)

# codec
set(CODEC_CMAKE ${SUBMODULE_ROOT_DIR}/ext/codec CACHE PATH "Location of codec-cmake" FORCE)
message(STATUS "codec-location: " ${CODEC_CMAKE})
ExternalProject_Add(
  CODEC_PROJECT
  PREFIX ${SUBMODULE_PREFIX_DIR}
  SOURCE_DIR ${CODEC_CMAKE}
  CMAKE_ARGS -DSUBMODULE_PREFIX_DIR=${SUBMODULE_PREFIX_DIR} -DSUBMODULE_ROOT_DIR=${SUBMODULE_ROOT_DIR} -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} ${SUBMODULE_FLAGS}
  )
ExternalProject_Get_Property(CODEC_PROJECT binary_dir)
set(CODEC_PROJECT_DIR ${binary_dir}/lib/cmake/codec CACHE PATH "location of codec cmake files" FORCE)
set(CPACK_INSTALL_CMAKE_PROJECTS ${CPACK_INSTALL_CMAKE_PROJECTS} "${binary_dir};codec;ALL;/")
add_dependencies(CODEC_PROJECT SBF_PROJECT PROPERTIES_PROJECT LOGGER_PROJECT CDR_PROJECT UTILS_PROJECT)

if (EXAMPLES)
    add_subdirectory(examples)
endif()

# add source
add_subdirectory(src)
add_dependencies(gwc CDR_PROJECT UTILS_PROJECT LOGGER_PROJECT SBF_PROJECT CODEC_PROJECT)

# unit-tests
option(TESTS "Enable unit-tests" OFF)
if(TESTS)
  if(WIN32)
    ExternalProject_Add(
      googletest
      SOURCE_DIR ${PROJECT_SOURCE_DIR}/ext/googletest
      CMAKE_ARGS -DCMAKE_CXX_FLAGS="/w" -DBUILD_GMOCK=ON -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}/googletest -DCMAKE_INSTALL_LIBDIR=${CMAKE_BINARY_DIR}/googletest/lib
      )
  else()
    ExternalProject_Add(
      googletest
      SOURCE_DIR ${PROJECT_SOURCE_DIR}/ext/googletest
      CMAKE_ARGS -DBUILD_GMOCK=ON -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}/googletest -DCMAKE_INSTALL_LIBDIR=${CMAKE_BINARY_DIR}/googletest/lib
      )
  endif()

  include(CTest)
  enable_testing()

  add_subdirectory(test)
endif()

# packaging
set(CPACK_INSTALL_CMAKE_PROJECTS ${CPACK_INSTALL_CMAKE_PROJECTS} "${CMAKE_BINARY_DIR};fosdk;ALL;/")
set(CPACK_GENERATOR "TGZ")
set(CPACK_PACKAGE_VERSION_MAJOR "${FOSDK_MAJOR_VERSION}")
set(CPACK_PACKAGE_VERSION_MINOR "${FOSDK_MINOR_VERSION}")
set(CPACK_PACKAGE_VERSION_PATCH "${FOSDK_PATCH_VERSION}")
set(CPACK_PACKAGE_NAME "fosdk")
set(CPACK_PACKAGE_VERSION "${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}")
if(DEBUG)
    set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME}_dbg-${CPACK_PACKAGE_VERSION}")
else()
    set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}")
endif()

INCLUDE(CPack)
