#
# Copyright (c) 2016-2017 Vinnie Falco (vinnie dot falco at gmail dot com)
#
# Distributed under the Boost Software License, Version 1.0. (See accompanying
# file LICENSE_1_0.txt or copy at http://www.boost.org/LICENSE_1_0.txt)
#
# Official repository: https://github.com/boostorg/beast
#

cmake_minimum_required (VERSION 3.5.1)

include(cmake/HunterGate.cmake)
HunterGate(
    URL "https://github.com/ruslo/hunter/archive/v0.23.202.tar.gz"
    SHA1 "931b1cb6b36a7a96948d2cfc8ec5e553bf3160f3"
)
project (Beast VERSION 189)


set_property (GLOBAL PROPERTY USE_FOLDERS ON)
option (Beast_BUILD_EXAMPLES "Build examples" OFF)
option (Beast_BUILD_TESTS "Build tests" OFF)

if (MSVC)
    set (CMAKE_VERBOSE_MAKEFILE FALSE)

    add_definitions (-D_WIN32_WINNT=0x0601)
    add_definitions (-D_SCL_SECURE_NO_WARNINGS=1)
    add_definitions (-D_CRT_SECURE_NO_WARNINGS=1)

    add_compile_options(
        /bigobj       # large object file format
        /permissive-  # strict C++
        #/wd4503      # decorated name length exceeded, name was truncated
        /W4           # enable all warnings
        /MP           # Multi-processor compilation
        )

    set (Boost_USE_STATIC_LIBS ON)
    set (Boost_USE_STATIC_RUNTIME ON)

    set (CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd")
    set (CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Ob2 /Oi /Ot /GL /MT")
    set (CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} /Oi /Ot /MT")

    set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /SAFESEH:NO")
    set (CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /LTCG")

    # for RelWithDebInfo builds, disable incremental linking
    # since CMake sets it ON by default for that build type and it
    # causes warnings
    #
    string (REPLACE "/INCREMENTAL" "/INCREMENTAL:NO" replacement_flags
        ${CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO})
    set (CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO ${replacement_flags})

else()
    set (THREADS_PREFER_PTHREAD_FLAG ON)
    find_package (Threads)

    set( CMAKE_CXX_FLAGS
      "${CMAKE_CXX_FLAGS} -std=c++11 -Wall -Wextra -Wpedantic -Wno-unused-parameter")

    if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wrange-loop-analysis")
    endif ()
endif()

# Must come before Boost includes, otherwise the
# IDE sees the wrong file due to boost/ symlinks.
include_directories (include)

#-------------------------------------------------------------------------------
#
# Boost
#

get_filename_component (BOOST_ROOT ../../ ABSOLUTE)

# <<<<<<< HEAD
# # VFALCO I want static but "b2 stage" builds a minimal set which excludes static
# add_definitions (-DBOOST_ALL_STATIC_LINK=1)
# 
# # Some of the tests use deprecated APIs
# #add_definitions (-DBOOST_ASIO_NO_DEPRECATED=1)
# =======
set(BOOST_COMPONENTS system)
if (Beast_BUILD_EXAMPLES OR Beast_BUILD_TESTS)
    list(APPEND BOOST_COMPONENTS coroutine context filesystem program_options thread)
endif()
hunter_add_package(Boost COMPONENTS ${BOOST_COMPONENTS})
find_package (Boost CONFIG REQUIRED ${BOOST_COMPONENTS})

add_definitions (-DBOOST_ASIO_DISABLE_BOOST_ARRAY=1)
add_definitions (-DBOOST_ASIO_DISABLE_BOOST_BIND=1)
add_definitions (-DBOOST_ASIO_DISABLE_BOOST_DATE_TIME=1)
add_definitions (-DBOOST_ASIO_DISABLE_BOOST_REGEX=1)
add_definitions (-DBOOST_COROUTINES_NO_DEPRECATION_WARNING=1)

include_directories (${BOOST_ROOT})

link_directories(${BOOST_ROOT}/stage/lib)

if (MINGW)
    link_libraries(ws2_32 mswsock)
endif()

#-------------------------------------------------------------------------------
#
# OpenSSL
#

# Only used for examples and tests
if(Beast_BUILD_EXAMPLES OR Beast_BUILD_TESTS)
    hunter_add_package(OpenSSL)
    find_package(OpenSSL REQUIRED)
    add_definitions (-DBOOST_BEAST_USE_OPENSSL=1)
else()
    add_definitions (-DBOOST_BEAST_USE_OPENSSL=0)
endif()

#
#-------------------------------------------------------------------------------

function (DoGroupSources curdir rootdir folder)
    file (GLOB children RELATIVE ${PROJECT_SOURCE_DIR}/${curdir} ${PROJECT_SOURCE_DIR}/${curdir}/*)
    foreach (child ${children})
        if (IS_DIRECTORY ${PROJECT_SOURCE_DIR}/${curdir}/${child})
            DoGroupSources (${curdir}/${child} ${rootdir} ${folder})
        elseif (${child} STREQUAL "CMakeLists.txt")
            source_group("" FILES ${PROJECT_SOURCE_DIR}/${curdir}/${child})
        else()
            string (REGEX REPLACE ^${rootdir} ${folder} groupname ${curdir})
            string (REPLACE "/" "\\" groupname ${groupname})
            source_group (${groupname} FILES ${PROJECT_SOURCE_DIR}/${curdir}/${child})
        endif()
    endforeach()
endfunction()

function (GroupSources curdir folder)
    DoGroupSources (${curdir} ${curdir} ${folder})
endfunction()

#-------------------------------------------------------------------------------

if ("${VARIANT}" STREQUAL "coverage")
    if (MSVC)
    else()
        set (CMAKE_BUILD_TYPE DEBUG)
        set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse4.2 --coverage")
        set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --coverage")
    endif()

elseif ("${VARIANT}" STREQUAL "ubasan")
    if (MSVC)
    else()
        set (CMAKE_BUILD_TYPE RELWITHDEBINFO)
        set (CMAKE_CXX_FLAGS
          "${CMAKE_CXX_FLAGS} -DBOOST_BEAST_NO_SLOW_TESTS=1 -msse4.2 -funsigned-char -fno-omit-frame-pointer -fsanitize=address,undefined -fno-sanitize-recover=address,undefined -fsanitize-blacklist=${PROJECT_SOURCE_DIR}/tools/blacklist.supp")
        set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=address,undefined -fno-sanitize-recover=address,undefined")
    endif()

elseif ("${VARIANT}" STREQUAL "debug")
    set (CMAKE_BUILD_TYPE DEBUG)

elseif ("${VARIANT}" STREQUAL "release")
    set (CMAKE_BUILD_TYPE RELEASE)

endif()

#-------------------------------------------------------------------------------
#
# Library interface
#

add_library (${PROJECT_NAME} INTERFACE)
target_link_libraries (${PROJECT_NAME} INTERFACE Boost::system)
if (NOT MSVC)
    target_link_libraries (${PROJECT_NAME} INTERFACE Threads::Threads)
endif()
target_compile_definitions (${PROJECT_NAME} INTERFACE BOOST_COROUTINES_NO_DEPRECATION_WARNING=1)

# Install

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(include_install_dir "include")

set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(targets_export_name "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)

# Note: use 'targets_export_name'
configure_file("cmake/Config.cmake.in" "${project_config}" @ONLY)

install(
    TARGETS ${PROJECT_NAME}
    EXPORT "${targets_export_name}"
    INCLUDES DESTINATION "${include_install_dir}"
)

install(
    DIRECTORY "include/"
    DESTINATION "${include_install_dir}"
)

install(
    FILES "${project_config}" "${version_config}"
    DESTINATION "${config_install_dir}"
)

install(
    EXPORT "${targets_export_name}"
    NAMESPACE "${namespace}"
    DESTINATION "${config_install_dir}"
)

#-------------------------------------------------------------------------------
#
# Tests and examples
#

include_directories (.)

if (OPENSSL_FOUND)
    include_directories (${OPENSSL_INCLUDE_DIR})
endif()

file (GLOB_RECURSE BOOST_BEAST_FILES
    ${PROJECT_SOURCE_DIR}/include/boost/beast/*.hpp
    ${PROJECT_SOURCE_DIR}/include/boost/beast/*.ipp
)

file (GLOB_RECURSE COMMON_FILES
    ${PROJECT_SOURCE_DIR}/example/common/*.hpp
  )

if (Beast_BUILD_EXAMPLES AND
    (NOT "${VARIANT}" STREQUAL "coverage") AND
    (NOT "${VARIANT}" STREQUAL "ubasan"))
    add_subdirectory (example)
endif()

if (Beast_BUILD_TESTS)
    add_subdirectory (test)
endif()
