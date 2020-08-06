cmake_minimum_required(VERSION 3.0.0)
include("cmake/HunterGate.cmake")
HunterGate(
    URL "https://github.com/cis-ncbj/hunter/archive/v0.23.263-cis.zip"
    SHA1 "f0a89769bab81e238d2595bfa5cea999e15a2b0d"
)

list (APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

include (DetectCPPZMQVersion)

project(cppzmq VERSION ${DETECTED_CPPZMQ_VERSION})

hunter_add_package(ZeroMQ)
find_package(ZeroMQ CONFIG REQUIRED)

# TODO "REQUIRED" above should already cause a fatal failure if not found, but this doesn't seem to work
if(NOT ZeroMQ_FOUND)
    message(FATAL_ERROR "ZeroMQ was not found, neither as a CMake package nor via pkg-config")
endif()

OPTION (ENABLE_DRAFTS "Build and install draft classes and methods" OFF)
if (ENABLE_DRAFTS)
    ADD_DEFINITIONS (-DZMQ_BUILD_DRAFT_API)
    set (pkg_config_defines "-DZMQ_BUILD_DRAFT_API=1")
else (ENABLE_DRAFTS)
    set (pkg_config_defines "")
endif (ENABLE_DRAFTS)

message(STATUS "cppzmq v${cppzmq_VERSION}")

set(CPPZMQ_HEADERS
    zmq.hpp
    zmq_addon.hpp
)

if(BUILD_SHARED_LIBS)
  add_library(cppzmq INTERFACE)
  target_include_directories(cppzmq INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
                                                 $<INSTALL_INTERFACE:include>)
  target_link_libraries(cppzmq INTERFACE ZeroMQ::libzmq)
  set(cppzmq_targets cppzmq)
else()
  add_library(cppzmq-static INTERFACE)
  target_include_directories(cppzmq-static INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
                                                 $<INSTALL_INTERFACE:include>)
  target_link_libraries(cppzmq-static INTERFACE ZeroMQ::libzmq-static)
  set(cppzmq_targets cppzmq-static)
endif()


include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

install(TARGETS ${cppzmq_targets}
        EXPORT ${PROJECT_NAME}-targets)

install(FILES ${CPPZMQ_HEADERS}
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

# GNUInstallDirs "DATADIR" wrong here; CMake search path wants "share".
set(CPPZMQ_CMAKECONFIG_INSTALL_DIR "share/cmake/${PROJECT_NAME}" CACHE STRING "install path for cppzmqConfig.cmake")

configure_file(libzmq-pkg-config/FindZeroMQ.cmake
               libzmq-pkg-config/FindZeroMQ.cmake
               COPYONLY)

export(EXPORT ${PROJECT_NAME}-targets
     FILE "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Targets.cmake")
configure_package_config_file(${PROJECT_NAME}Config.cmake.in
                              "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
                              INSTALL_DESTINATION ${CPPZMQ_CMAKECONFIG_INSTALL_DIR})
# Workaround until ARCH_INDEPENDENT flag can be used with cmake 3.14.
# The ConigVersion.cmake file contains checks for the architecture is was
# generated on, which can cause problems for header only libraries
# used with e.g. the Conan package manager. Since it is header only we
# can/should omit those checks.
set(CPPZMQ_SIZEOF_VOID_P ${CMAKE_SIZEOF_VOID_P})
set(CMAKE_SIZEOF_VOID_P "") # a simple unset is not sufficient
write_basic_package_version_file(${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
                                 VERSION ${CPPZMQ_VERSION}
                                 COMPATIBILITY AnyNewerVersion)
set(CMAKE_SIZEOF_VOID_P ${CPPZMQ_SIZEOF_VOID_P})

install(EXPORT ${PROJECT_NAME}-targets
        FILE ${PROJECT_NAME}Targets.cmake
        DESTINATION ${CPPZMQ_CMAKECONFIG_INSTALL_DIR}
        NAMESPACE ${PROJECT_NAME}::)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
              ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
              DESTINATION ${CPPZMQ_CMAKECONFIG_INSTALL_DIR})
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/libzmq-pkg-config/FindZeroMQ.cmake
              DESTINATION ${CPPZMQ_CMAKECONFIG_INSTALL_DIR}/libzmq-pkg-config)

option(CPPZMQ_BUILD_TESTS "Whether or not to build the tests" ON)

if (CPPZMQ_BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()
