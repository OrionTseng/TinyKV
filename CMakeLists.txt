cmake_minimum_required(VERSION 3.16)

project(kv_storage LANGUAGES C)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

set(KVS_NETWORK_BACKEND "REACTOR" CACHE STRING
    "Network backend: REACTOR, PROACTOR, or NTYCO")
set_property(CACHE KVS_NETWORK_BACKEND PROPERTY STRINGS REACTOR PROACTOR NTYCO)
option(KVS_BUILD_TEST_CLIENT "Build the TCP test client" ON)

if(NOT CMAKE_SYSTEM_NAME STREQUAL "Linux")
    message(FATAL_ERROR
        "kv_storage server targets Linux: the available backends use epoll, io_uring, or NtyCo. "
        "Configure and build it on Linux.")
endif()

find_package(Threads REQUIRED)

set(KVS_SOURCES
    src/kvstore.c
    src/kvs_array.c
    src/kvs_rbtree.c
    src/kvs_hash.c)

string(TOUPPER "${KVS_NETWORK_BACKEND}" KVS_NETWORK_BACKEND)
if(KVS_NETWORK_BACKEND STREQUAL "REACTOR")
    list(APPEND KVS_SOURCES src/reactor.c)
    set(KVS_NETWORK_DEFINE NETWORK_REACTOR)
elseif(KVS_NETWORK_BACKEND STREQUAL "PROACTOR")
    find_path(LIBURING_INCLUDE_DIR NAMES liburing.h)
    find_library(LIBURING_LIBRARY NAMES uring)
    if(NOT LIBURING_INCLUDE_DIR OR NOT LIBURING_LIBRARY)
        message(FATAL_ERROR
            "PROACTOR requires liburing. On Debian/Ubuntu install: sudo apt install liburing-dev")
    endif()
    list(APPEND KVS_SOURCES src/proactor.c)
    set(KVS_NETWORK_DEFINE NETWORK_PROACTOR)
elseif(KVS_NETWORK_BACKEND STREQUAL "NTYCO")
    find_path(NTYCO_INCLUDE_DIR NAMES nty_coroutine.h HINTS "${CMAKE_CURRENT_SOURCE_DIR}/NtyCo/core")
    find_library(NTYCO_LIBRARY NAMES ntyco HINTS "${CMAKE_CURRENT_SOURCE_DIR}/NtyCo")
    if(NOT NTYCO_INCLUDE_DIR OR NOT NTYCO_LIBRARY)
        message(FATAL_ERROR
            "NTYCO requires the NtyCo header and libntyco library. Place them under NtyCo/ or set CMake search paths.")
    endif()
    list(APPEND KVS_SOURCES src/ntyco.c)
    set(KVS_NETWORK_DEFINE NETWORK_NTYCO)
else()
    message(FATAL_ERROR
        "Invalid KVS_NETWORK_BACKEND='${KVS_NETWORK_BACKEND}'. Choose REACTOR, PROACTOR, or NTYCO.")
endif()

add_executable(kvstore ${KVS_SOURCES})
target_include_directories(kvstore PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/include")
target_compile_definitions(kvstore PRIVATE NETWORK_SELECT=${KVS_NETWORK_DEFINE})
target_link_libraries(kvstore PRIVATE Threads::Threads)

if(KVS_NETWORK_BACKEND STREQUAL "PROACTOR")
    target_include_directories(kvstore PRIVATE "${LIBURING_INCLUDE_DIR}")
    target_link_libraries(kvstore PRIVATE "${LIBURING_LIBRARY}")
elseif(KVS_NETWORK_BACKEND STREQUAL "NTYCO")
    target_include_directories(kvstore PRIVATE "${NTYCO_INCLUDE_DIR}")
    target_link_libraries(kvstore PRIVATE "${NTYCO_LIBRARY}" ${CMAKE_DL_LIBS})
endif()

if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
    target_compile_options(kvstore PRIVATE -Wall -Wextra -Wpedantic)
endif()

if(KVS_BUILD_TEST_CLIENT)
    add_executable(kvstore_test_client src/testcase.c)
    if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
        target_compile_options(kvstore_test_client PRIVATE -Wall -Wextra -Wpedantic)
    endif()
endif()
