# Most widely used distributions have cmake 3.5 or greater available as of March
# 2019.  A notable exception is RHEL-7 (CentOS7).  You can install a current
# version of CMake by first installing Extra Packages for Enterprise Linux
# (https://fedoraproject.org/wiki/EPEL#Extra_Packages_for_Enterprise_Linux_.28EPEL.29)
# and then issuing `yum install cmake3` on the command line.
cmake_minimum_required(VERSION 3.5)

project(SimpleAmqpClient LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 98)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Follow all steps below in order to calculate new ABI version when updating the library
# NOTE: THIS IS UNRELATED to the actual project version
#
# 1. If the library source code has changed at all since the last update, then increment revision
# 2. If any interfaces have been added, removed, or changed since the last update, increment
#    current and set revision to 0.
# 3. If any interfaces have been added since the last public release, then increment age.
# 4. If any interfaces have been removed since the last public release, then set age to 0.

set(SAC_SOVERSION_CURRENT   7)
set(SAC_SOVERSION_REVISION  1)
set(SAC_SOVERSION_AGE       0)

math(EXPR SAC_SOVERSION_MAJOR "${SAC_SOVERSION_CURRENT} - ${SAC_SOVERSION_AGE}")
math(EXPR SAC_SOVERSION_MINOR "${SAC_SOVERSION_AGE}")
math(EXPR SAC_SOVERSION_PATCH "${SAC_SOVERSION_REVISION}")

set(SAC_VERSION ${SAC_SOVERSION_MAJOR}.${SAC_SOVERSION_MINOR}.${SAC_SOVERSION_PATCH})
set(SAC_SOVERSION ${SAC_SOVERSION_MAJOR})

file(STRINGS src/SimpleAmqpClient/Version.h _API_VERSION_MAJOR REGEX "^#define SIMPLEAMQPCLIENT_VERSION_MAJOR [0-9]+$")
file(STRINGS src/SimpleAmqpClient/Version.h _API_VERSION_MINOR REGEX "^#define SIMPLEAMQPCLIENT_VERSION_MINOR [0-9]+$")
file(STRINGS src/SimpleAmqpClient/Version.h _API_VERSION_PATCH REGEX "^#define SIMPLEAMQPCLIENT_VERSION_PATCH [0-9]+$")

string(REGEX MATCH "[0-9]+" _API_VERSION_MAJOR ${_API_VERSION_MAJOR})
string(REGEX MATCH "[0-9]+" _API_VERSION_MINOR ${_API_VERSION_MINOR})
string(REGEX MATCH "[0-9]+" _API_VERSION_PATCH ${_API_VERSION_PATCH})

set(SAC_APIVERSION ${_API_VERSION_MAJOR}.${_API_VERSION_MINOR}.${_API_VERSION_PATCH})

option(BUILD_SHARED_LIBS "Build SimpleAmqpClient as a shared library" ON)

# Force the use of static boost library for static libraries
include(CMakeDependentOption)

cmake_dependent_option(
    Boost_Dynamic_Linking_ENABLED
    "Enable boost dynamic linking"
    ON
    "BUILD_SHARED_LIBS"
    OFF
    )

if(Boost_Dynamic_Linking_ENABLED)
  set(Boost_USE_STATIC_LIBS OFF)
else()
  set(Boost_USE_STATIC_LIBS ON)
endif()
set(Boost_USE_STATIC_RUNTIME OFF)

find_package(Boost 1.47.0 COMPONENTS chrono system REQUIRED)
include_directories(SYSTEM ${Boost_INCLUDE_DIRS})
link_directories(${Boost_LIBRARY_DIRS})

# Try using the CMake config modules first
find_package(rabbitmq-c CONFIG QUIET)
if (rabbitmq-c_FOUND)
    if (BUILD_SHARED_LIBS)
        set(Rabbitmqc_LIBRARY rabbitmq::rabbitmq)
    else()
        set(Rabbitmqc_LIBRARY rabbitmq::rabbitmq-static)
    endif()
    get_target_property(Rabbitmqc_INCLUDE_DIRS ${Rabbitmqc_LIBRARY} INTERFACE_INCLUDE_DIRECTORIES)
else()
    set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/Modules)
    find_package(Rabbitmqc REQUIRED)
    INCLUDE_DIRECTORIES(SYSTEM ${Rabbitmqc_INCLUDE_DIRS})
endif()

option(ENABLE_SSL_SUPPORT "Enable SSL support." ${Rabbitmqc_SSL_ENABLED})

if (ENABLE_SSL_SUPPORT)
  add_definitions(-DSAC_SSL_SUPPORT_ENABLED)
endif()

if (CMAKE_GENERATOR MATCHES ".*(Make|Ninja).*"
    AND NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel" FORCE)
  message(STATUS "CMAKE_BUILD_TYPE not specified. Using ${CMAKE_BUILD_TYPE} build")
endif ()

if (CMAKE_CXX_FLAGS STREQUAL ""
    AND NOT DEFINED SAC_CXX_FLAGS_SET)
  if (CMAKE_COMPILER_IS_GNUCXX
      OR CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    SET(CMAKE_CXX_FLAGS "-Wall -Wextra" CACHE STRING "Flags used by the compiler during all build types." FORCE)
  endif ()
  set(SAC_CXX_FLAGS_SET TRUE CACHE INTERNAL "Have the SAC default compiler flags been set?")
endif ()

include_directories(BEFORE src
    ${CMAKE_CURRENT_BINARY_DIR})

if (WIN32)
    set(SOCKET_LIBRARY ws2_32)
endif ()

set(SAC_LIB_SRCS
    src/SimpleAmqpClient/SimpleAmqpClient.h

    src/SimpleAmqpClient/AmqpException.h
    src/AmqpException.cpp

    src/SimpleAmqpClient/Bytes.h

    src/SimpleAmqpClient/Channel.h
    src/Channel.cpp

    src/SimpleAmqpClient/ChannelImpl.h
    src/ChannelImpl.cpp

    src/SimpleAmqpClient/BasicMessage.h
    src/BasicMessage.cpp

    src/SimpleAmqpClient/Util.h

    src/SimpleAmqpClient/AmqpLibraryException.h
    src/AmqpLibraryException.cpp

    src/SimpleAmqpClient/AmqpResponseLibraryException.h
    src/AmqpResponseLibraryException.cpp

    src/SimpleAmqpClient/BadUriException.h
    src/SimpleAmqpClient/ConnectionClosedException.h
    src/SimpleAmqpClient/ConsumerTagNotFoundException.h
    src/SimpleAmqpClient/MessageRejectedException.h

    src/SimpleAmqpClient/Envelope.h
    src/Envelope.cpp

    src/SimpleAmqpClient/MessageReturnedException.h
    src/MessageReturnedException.cpp

    src/SimpleAmqpClient/Table.h
    src/Table.cpp

    src/SimpleAmqpClient/TableImpl.h
    src/TableImpl.cpp
    )


add_library(SimpleAmqpClient ${SAC_LIB_SRCS})
target_link_libraries(SimpleAmqpClient ${Rabbitmqc_LIBRARY} ${SOCKET_LIBRARY} ${Boost_LIBRARIES} $<$<BOOL:${Boost_Dynamic_Linking_ENABLED}>:Boost::dynamic_linking>)

if (WIN32)
    if (NOT BUILD_SHARED_LIBS)
        target_compile_definitions(SimpleAmqpClient PUBLIC SimpleAmqpClient_STATIC)
    endif ()

    set_target_properties(SimpleAmqpClient PROPERTIES VERSION ${SAC_VERSION} OUTPUT_NAME SimpleAmqpClient.${SAC_SOVERSION})
else ()
    set_target_properties(SimpleAmqpClient PROPERTIES VERSION ${SAC_VERSION} SOVERSION ${SAC_SOVERSION})
endif ()

# Some smoke tests:

option(ENABLE_TESTING "Enable smoke tests" OFF)

if (ENABLE_TESTING)
    enable_testing()

    set(BUILD_GTEST ON CACHE BOOL "" FORCE)
    set(BUILD_GMOCK OFF CACHE BOOL "" FORCE)
    set(INSTALL_GTEST OFF CACHE BOOL "" FORCE)

    # This only affects targets declared after this.
    set(BUILD_SHARED_LIBS OFF)

    mark_as_advanced(BUILD_GMOCK)
    mark_as_advanced(BUILD_GTEST)
    mark_as_advanced(INSTALL_GTEST)
    mark_as_advanced(gmock_build_tests)
    mark_as_advanced(gtest_build_samples)
    mark_as_advanced(gtest_build_tests)
    mark_as_advanced(gtest_disable_pthreads)
    mark_as_advanced(gtest_force_shared_crt)
    mark_as_advanced(gtest_hide_internal_symbols)

    add_subdirectory(third-party/googletest)
    add_subdirectory(testing)
endif (ENABLE_TESTING)


# Documentation generation
find_package(Doxygen COMPONENTS dot)
option(BUILD_API_DOCS "Build Doxygen API docs" ${DOXYGEN_FOUND})

if (BUILD_API_DOCS)
  if (NOT DOXYGEN_FOUND)
    message(FATAL_ERROR "Doxygen is required to build the API documentation")
  endif ()
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.in ${CMAKE_CURRENT_BINARY_DIR}/docs/Doxyfile @ONLY)

  add_custom_target(docs ALL
    COMMAND ${DOXYGEN_EXECUTABLE}
    VERBATIM
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/docs
    DEPENDS SimpleAmqpClient
    COMMENT "Generating API documentation"
    SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.in
    )
endif ()

include(GNUInstallDirs)

include(CMakePackageConfigHelpers)
set(SAC_CMAKE_DIR ${CMAKE_INSTALL_LIBDIR}/cmake/SimpleAmqpClient)
set(project_config "${CMAKE_CURRENT_BINARY_DIR}/SimpleAmqpClientConfig.cmake")
set(version_config "${CMAKE_CURRENT_BINARY_DIR}/SimpleAmqpClientConfig-version.cmake")
set(targets_export_name SimpleAmqpClient-targets)
list(APPEND INSTALL_TARGETS SimpleAmqpClient)

write_basic_package_version_file(
    "${version_config}"
    VERSION ${SAC_VERSION}
    COMPATIBILITY AnyNewerVersion
	)

configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/SimpleAmqpClientConfig.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${SAC_CMAKE_DIR}"
	)

install(TARGETS ${INSTALL_TARGETS} EXPORT ${targets_export_name}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    )
    
install(FILES ${project_config} ${version_config}
    DESTINATION ${SAC_CMAKE_DIR}
    )

install(EXPORT ${targets_export_name}
    DESTINATION ${SAC_CMAKE_DIR}
    NAMESPACE SimpleAmqpClient::
    )
install(FILES
    src/SimpleAmqpClient/AmqpException.h
    src/SimpleAmqpClient/AmqpLibraryException.h
    src/SimpleAmqpClient/AmqpResponseLibraryException.h
    src/SimpleAmqpClient/BadUriException.h
    src/SimpleAmqpClient/BasicMessage.h
    src/SimpleAmqpClient/Channel.h
    src/SimpleAmqpClient/ConnectionClosedException.h
    src/SimpleAmqpClient/ConsumerCancelledException.h
    src/SimpleAmqpClient/ConsumerTagNotFoundException.h
    src/SimpleAmqpClient/Envelope.h
    src/SimpleAmqpClient/MessageReturnedException.h
    src/SimpleAmqpClient/MessageRejectedException.h
    src/SimpleAmqpClient/SimpleAmqpClient.h
    src/SimpleAmqpClient/Table.h
    src/SimpleAmqpClient/Util.h
    src/SimpleAmqpClient/Version.h
    DESTINATION include/SimpleAmqpClient
    )

set(prefix ${CMAKE_INSTALL_PREFIX})
set(exec_prefix "\${prefix}")
set(libdir "\${prefix}/${CMAKE_INSTALL_LIBDIR}")
set(includedir "\${prefix}/include")
if(WIN32)
    get_target_property(SIMPLEAMQPCLIENT_LIB SimpleAmqpClient OUTPUT_NAME)
else(WIN32)
    set(SIMPLEAMQPCLIENT_LIB SimpleAmqpClient)
endif(WIN32)

# Propagate package dependencies
if (BUILD_SHARED_LIBS)
    set(requires_private "librabbitmq")
else (BUILD_SHARED_LIBS)
    set(requires_public "librabbitmq")
endif (BUILD_SHARED_LIBS)

# Propagate interface compile definitions
set(SIMPLEAMQPCLIENT_DEFINITIONS "")
get_target_property(propagated_definitions SimpleAmqpClient INTERFACE_COMPILE_DEFINITIONS)
if (propagated_definitions)
    foreach(_def ${propagated_definitions})
        set(SIMPLEAMQPCLIENT_DEFINITIONS "${SIMPLEAMQPCLIENT_DEFINITIONS} -D${_def}")
    endforeach()
endif(propagated_definitions)

# Propagate library dependencies
set(libs_private "")
set(libs_public "")

if (BUILD_SHARED_LIBS)
    set(populate_libs "libs_private")
else (BUILD_SHARED_LIBS)
    set(populate_libs "libs_public")
    set(extra_win32_targets "${Rabbitmqc_LIBRARY};${SOCKET_LIBRARY}")
endif (BUILD_SHARED_LIBS)

foreach(_lib ${Boost_LIBRARIES} ${extra_win32_targets})

    # Check if FindBoost.cmake provided actual library paths or targets
    if(TARGET ${_lib})
        get_target_property(_lib ${_lib} LOCATION)
        message(WARNING "Using target ${_lib} as a library")
    endif()

    get_filename_component(_LIBPATH ${_lib} PATH)
    if (NOT _LIBPATH STREQUAL _LASTLIBPATH AND NOT _LIBPATH STREQUAL "")
        set(${populate_libs} "${${populate_libs}} -L\"${_LIBPATH}\"")
        set(_LASTLIBPATH ${_LIBPATH})
    endif()

    get_filename_component(_LIBNAME ${_lib} NAME_WLE)
    if (NOT _LIBNAME STREQUAL "debug" AND NOT _LIBNAME STREQUAL "optimized")
        if (NOT WIN32)
            string(REGEX REPLACE "^lib" "" _LIBNAME ${_LIBNAME})
        endif()
        set(_LIBNAME "-l${_LIBNAME}")
        set(${populate_libs} "${${populate_libs}} ${_LIBNAME}")
    endif()
endforeach()

configure_file(libSimpleAmqpClient.pc.in ${CMAKE_CURRENT_BINARY_DIR}/libSimpleAmqpClient.pc @ONLY)

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/libSimpleAmqpClient.pc
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
    )
