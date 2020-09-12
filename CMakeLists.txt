#
# LibTomMath, a free open source portable number theoretic multiple-precision
# integer (MPI) library written entirely in C.
#

cmake_minimum_required(VERSION 3.7)
project(libtommath VERSION 1.2.0 LANGUAGES C)

#-----------------------------------------------------------------------------
# Include cmake modules
#-----------------------------------------------------------------------------
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)
include(sources.cmake)

#-----------------------------------------------------------------------------
# library target
#-----------------------------------------------------------------------------
add_library(libtommath
    ${SOURCES}
)
if(WIN32)
    target_sources(libtommath PRIVATE tommath.def)
    set_target_properties(libtommath PROPERTIES PREFIX "")
endif()

target_include_directories(libtommath PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

set_target_properties(libtommath PROPERTIES OUTPUT_NAME tommath)
set_target_properties(libtommath PROPERTIES VERSION ${PROJECT_VERSION})
set_target_properties(libtommath PROPERTIES SOVERSION ${PROJECT_VERSION_MAJOR})

add_library(libtommath::libtommath ALIAS libtommath)

#-----------------------------------------------------------------------------
# demo target
#-----------------------------------------------------------------------------

enable_testing()

add_executable(demo-libtommath EXCLUDE_FROM_ALL
    ${CMAKE_CURRENT_SOURCE_DIR}/demo/shared.c
    ${CMAKE_CURRENT_SOURCE_DIR}/demo/test.c
)

target_include_directories(demo-libtommath PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}
)

target_link_libraries(demo-libtommath PRIVATE libtommath::libtommath)

add_test(NAME demo-libtommath
    COMMAND demo-libtommath
)

#-----------------------------------------------------------------------------
# demo target
#-----------------------------------------------------------------------------
add_custom_target(check
    COMMAND demo-libtommath
    DEPENDS demo-libtommath
)

#-----------------------------------------------------------------------------
# Install/export targets and files
#-----------------------------------------------------------------------------
set(CONFIG_INSTALL_DIR "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}")
set(PC_INSTALL_DIR "${CMAKE_INSTALL_LIBDIR}/pkgconfig")
set(PROJECT_VERSION_FILE "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake")
set(PROJECT_CONFIG_FILE libtommath-config.cmake)
set(TARGETS_EXPORT_NAME libtommath)

install(TARGETS libtommath
    EXPORT ${TARGETS_EXPORT_NAME}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

install(FILES ${HEADERS}
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

#generate .pc file
set(VERSION "${PROJECT_VERSION}")
set(PREFIX "${CMAKE_INSTALL_PREFIX}")
configure_file(libtommath.pc.in libtommath.pc @ONLY)

# install .pc file
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/libtommath.pc"
        DESTINATION "${PC_INSTALL_DIR}"
)

# generate package version file
write_basic_package_version_file(
    ${PROJECT_VERSION_FILE}
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

# install version file
install(FILES ${PROJECT_VERSION_FILE}
    DESTINATION "${CONFIG_INSTALL_DIR}"
)

# build directory package config
export(EXPORT ${TARGETS_EXPORT_NAME}
    FILE ${PROJECT_CONFIG_FILE}
)

# installed package config
install(EXPORT ${TARGETS_EXPORT_NAME}
    NAMESPACE libtommath::
    DESTINATION "${CONFIG_INSTALL_DIR}"
    FILE ${PROJECT_CONFIG_FILE}
)

# add to CMake registry
export(PACKAGE ${PROJECT_NAME})

#---------------------------------------------------------------------------------------
# Create release packages
#---------------------------------------------------------------------------------------
# package release version
set(PACKAGE_RELEASE_VERSION 1)

# determine distribution and architecture
find_program(LSB_RELEASE lsb_release)

execute_process(COMMAND uname -m OUTPUT_VARIABLE MACHINE_ARCH OUTPUT_STRIP_TRAILING_WHITESPACE)

if(LSB_RELEASE)
    execute_process(COMMAND lsb_release -sr OUTPUT_VARIABLE LINUX_DISTRO_VERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
    execute_process(COMMAND lsb_release -si OUTPUT_VARIABLE LINUX_DISTRO OUTPUT_STRIP_TRAILING_WHITESPACE)

    string(TOLOWER ${LINUX_DISTRO} LINUX_DISTRO)
endif()

# default CPack generators
set(CPACK_GENERATOR TGZ STGZ)

# extra CPack generators
if(LINUX_DISTRO STREQUAL "debian" OR LINUX_DISTRO STREQUAL "ubuntu" OR LINUX_DISTRO STREQUAL "linuxmint")
    list(APPEND CPACK_GENERATOR DEB)
elseif(LINUX_DISTRO STREQUAL "fedora" OR LINUX_DISTRO STREQUAL "opensuse" OR LINUX_DISTRO STREQUAL "centos")
    list(APPEND CPACK_GENERATOR RPM)
endif()

# general CPack config
set(CPACK_PACKAGE_DIRECTORY ${CMAKE_BINARY_DIR}/packages)
message(STATUS "CPack: packages will be generated under ${CPACK_PACKAGE_DIRECTORY}")
set(CPACK_PACKAGE_NAME "${PROJECT_NAME}")
set(CPACK_PACKAGE_VERSION ${PROJECT_VERSION})
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "LibTomMath")
set(CPACK_PACKAGE_VENDOR "LibTomMath")
set(CPACK_PACKAGE_CONTACT "libtom@googlegroups.com")
set(CPACK_RESOURCE_FILE_LICENSE "${PROJECT_SOURCE_DIR}/LICENSE")
set(CPACK_PACKAGE_FILE_NAME ${CPACK_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}-${PACKAGE_RELEASE_VERSION}_${LINUX_DISTRO}-${LINUX_DISTRO_VERSION}_${MACHINE_ARCH})
set(CPACK_STRIP_FILES ON)

# deb specific CPack config
set(CPACK_DEBIAN_PACKAGE_RELEASE ${PACKAGE_RELEASE_VERSION})
set(CPACK_DEBIAN_FILE_NAME DEB-DEFAULT)
set(CPACK_DEBIAN_PACKAGE_SHLIBDEPS ON)
set(CPACK_DEBIAN_PACKAGE_HOMEPAGE "https://www.libtom.net/LibTomMath")
set(CPACK_DEBIAN_PACKAGE_SECTION "devel")

# rpm specific CPack config
set(CPACK_RPM_PACKAGE_URL "https://www.libtom.net/LibTomMath")
set(CPACK_RPM_PACKAGE_RELEASE ${PACKAGE_RELEASE_VERSION})
set(CPACK_RPM_PACKAGE_ARCHITECTURE ${MACHINE_ARCH})
set(CPACK_RPM_PACKAGE_NAME "${PROJECT_NAME}-${PROJECT_VERSION}")
set(CPACK_RPM_FILE_NAME "${PROJECT_NAME}_${PROJECT_VERSION}-${CPACK_RPM_PACKAGE_RELEASE}_${LINUX_DISTRO}-${LINUX_DISTRO_VERSION}_${CPACK_RPM_PACKAGE_ARCHITECTURE}.rpm")
set(CPACK_RPM_PACKAGE_LICENSE "WTFPL")

include(CPack)
