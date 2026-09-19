# KV Storage

一个用于学习 Linux 网络编程的内存键值（Key-Value，KV）存储原型。服务端通过 TCP 接收简易文本命令，并分别使用顺序数组、红黑树和哈希表保存数据。项目还提供三种可选网络后端：基于 `epoll` 的 Reactor、基于 `io_uring` 的 Proactor，以及依赖 NtyCo 的协程实现。

> 这是教学原型，不是生产数据库。数据不持久化，进程退出即丢失；协议和网络层也尚未实现生产服务所需的完整分帧、背压与异常恢复。

## 功能与结构

| 组件 | 源文件 | 作用 |
| --- | --- | --- |
| 协议解析与程序入口 | `src/kvstore.c` | 解析命令并初始化、销毁 KV 引擎。 |
| 顺序数组 | `src/kvs_array.c` | 无前缀命令；查找通常为线性扫描。 |
| 红黑树 | `src/kvs_rbtree.c` | `R` 前缀命令；按键的有序树组织数据。 |
| 哈希表 | `src/kvs_hash.c` | `H` 前缀命令；平均查找效率较高。 |
| Reactor | `src/reactor.c` | Linux `epoll` 就绪通知模型。 |
| Proactor | `src/proactor.c` | Linux `io_uring` 完成通知模型，依赖 liburing。 |
| NtyCo | `src/ntyco.c` | 协程网络模型，依赖仓库外的 NtyCo 库。 |
| 测试客户端 | `src/testcase.c` | 向运行中的服务端发送功能/简单压力请求。 |

## 运行环境与依赖

服务端仅面向 Linux：三个网络后端分别依赖 `epoll`、`io_uring` 或 NtyCo。macOS 和 Windows 不能直接编译当前服务端实现。

默认的 Reactor 后端只需要 C 编译器、CMake 与 POSIX 线程库。Ubuntu/Debian 上可安装：

```bash
sudo apt update
sudo apt install -y build-essential cmake
```

可选后端的额外依赖：

| 后端 | CMake 值 | 额外依赖 |
| --- | --- | --- |
| Reactor（默认） | `REACTOR` | 无。 |
| Proactor | `PROACTOR` | `liburing-dev`，且内核需要支持 `io_uring`。 |
| NtyCo | `NTYCO` | `nty_coroutine.h` 与 `libntyco`；可放在 `NtyCo/core/` 与 `NtyCo/`。 |

## 使用 CMake 构建

以下命令在项目根目录执行。构建目录位于源码树外，便于清理且不会污染源码：

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel
```

默认生成两个程序：

| 可执行文件 | 用途 |
| --- | --- |
| `build/kvstore` | KV TCP 服务端。 |
| `build/kvstore_test_client` | 简单测试客户端；可用 `-DKVS_BUILD_TEST_CLIENT=OFF` 关闭构建。 |

选择 Proactor：

```bash
sudo apt install -y liburing-dev
cmake -S . -B build-proactor -DKVS_NETWORK_BACKEND=PROACTOR
cmake --build build-proactor --parallel
```

选择 NtyCo：

```bash
cmake -S . -B build-ntyco -DKVS_NETWORK_BACKEND=NTYCO
cmake --build build-ntyco --parallel
```

`KVS_NETWORK_BACKEND` 只接受 `REACTOR`、`PROACTOR`、`NTYCO`。CMake 会只编译所选后端，并在缺少对应依赖时给出配置错误；不会要求安装其他两个后端的库。

## 启动与验证

启动服务端时传入起始端口：

```bash
./build/kvstore 2000
```

当前 Reactor 实现会从传入端口开始监听一组端口，因此 `2000` 是其中第一个监听端口。另开一个终端运行测试客户端：

```bash
./build/kvstore_test_client 127.0.0.1 2000 3
```

测试客户端的第三个参数决定测试类型：

| 值 | 测试 |
| --- | --- |
| `0` | 红黑树重复读写测试。 |
| `1` | 红黑树多键读写测试。 |
| `2` | 顺序数组重复读写测试。 |
| `3` | 哈希表基本功能测试。 |

也可以使用 `nc` 手工验证（每次连接发送一条命令更符合当前演示实现）：

```bash
printf 'HSET Teacher Alice' | nc 127.0.0.1 2000
printf 'HGET Teacher' | nc 127.0.0.1 2000
```

## 文本协议

请求以空格分隔，响应以 `\r\n` 结束。数组命令没有前缀，红黑树使用 `R` 前缀，哈希表使用 `H` 前缀。

| 操作 | 数组 | 红黑树 | 哈希表 | 成功响应 |
| --- | --- | --- | --- | --- |
| 新增 | `SET key value` | `RSET key value` | `HSET key value` | `OK`；键已存在时为 `EXIST`。 |
| 查询 | `GET key` | `RGET key` | `HGET key` | 值；键不存在时为 `NO EXIST`。 |
| 删除 | `DEL key` | `RDEL key` | `HDEL key` | `OK` 或 `NO EXIST`。 |
| 修改 | `MOD key value` | `RMOD key value` | `HMOD key value` | `OK` 或 `NO EXIST`。 |
| 判断存在 | `EXIST key` | `REXIST key` | `HEXIST key` | `EXIST` 或 `NO EXIST`。 |

示例：

```text
请求：HSET Teacher Alice
响应：OK\r\n

请求：HGET Teacher
响应：Alice\r\n
```

## 注意与已知限制

- TCP 是可靠、有序的**字节流**，不保留应用层消息边界。一次 `recv()` 可能只得到半条命令，也可能包含多条命令；一次 `send()` 也可能只写出部分数据。
- 当前代码将一次读取直接当作一条完整请求处理，没有连接级读缓冲、消息分帧或部分发送队列。因此协议中的空格不能可靠地出现在值中，持久长连接、分包与粘连场景也不能可靠处理。
- 所谓“粘包、拆包”是应用层消息边界设计问题，并不是 TCP 传输错误。实际协议应采用长度字段或严格的行分隔规则，并维护读写缓冲区。
- Reactor 示例中的事件循环不应执行耗时业务；真实服务器还需要有界任务队列、连接/请求大小上限、超时和背压策略。
- 三种后端都是学习示例，API 错误处理、资源释放、并发安全和连接生命周期管理仍有待完善。

## 开发建议

构建时建议保留警告；CMake 已为 GCC 与 Clang 开启 `-Wall -Wextra -Wpedantic`。开发阶段可以使用 Debug 构建：

```bash
cmake -S . -B build-debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build-debug --parallel
```

后续可继续完善：应用层分帧与部分写处理、Socket 的 `EINTR`/`EAGAIN` 处理、RAII 风格的 C++17 重构、持久化、超时与容量控制，以及单元测试和基准测试。
