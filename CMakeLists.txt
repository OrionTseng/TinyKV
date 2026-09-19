# 要求 CMake 3.16 或更新版本；该版本足够支持本项目使用的基本命令。
cmake_minimum_required(VERSION 3.16)

# 声明项目名称和使用的语言。源码为 C，而不是 C++。
project(kv_storage LANGUAGES C)

# 所有目标使用标准 C11；关闭 GNU 等编译器的语言扩展，避免无意依赖非标准语法。
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

# 当前默认服务端使用 epoll，而 epoll 是 Linux 专有 API。
# 尽早终止配置，可避免在 macOS/Windows 上出现难以理解的头文件错误。
if(NOT CMAKE_SYSTEM_NAME STREQUAL "Linux")
    message(FATAL_ERROR "kv_storage uses Linux epoll and must be built on Linux.")
endif()

# kvstore 由协议入口、三种内存索引实现和默认 Reactor 网络层组成。
# 如果要试验 io_uring 或 NtyCo，请在此列表中将 reactor.c 换成对应源码，
# 并自行添加 liburing 或 NtyCo 的头文件路径和链接库。
add_executable(kvstore
    src/kvstore.c
    src/reactor.c
    src/kvs_array.c
    src/kvs_rbtree.c
    src/kvs_hash.c)

# kvstore.h 与 server.h 位于 include/，只供本目标编译时查找。
target_include_directories(kvstore PRIVATE include)

# 明确选择 Reactor，避免头文件中的默认值以后被修改时影响此构建目标。
target_compile_definitions(kvstore PRIVATE NETWORK_SELECT=NETWORK_REACTOR)

# 为 GCC 和 Clang 启用常用警告；不会把警告升级为错误，便于学习阶段查看问题。
if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
    target_compile_options(kvstore PRIVATE -Wall -Wextra -Wpedantic)
endif()

# 测试客户端是独立可执行程序：启动 kvstore 后，使用它发送预置测试命令。
add_executable(kvstore_test_client src/testcase.c)

if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
    target_compile_options(kvstore_test_client PRIVATE -Wall -Wextra -Wpedantic)
endif()
