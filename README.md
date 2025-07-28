# redis-IO-Multiplexing
为什么redis的核心业务处理（处理命令）是单线程，但是面对千万条客户端请求，依然有着每秒数万 QPS 的高处理能力？这归功于redis的网络模型-IO多路复用+事件派发。
## 一、在开始介绍 Redis 之前，我们先来简单介绍下IO多路复用的epoll，理解多路复用原理。
在传统的同步阻塞网络编程模型里（没有协程以前），性能上不来的根本原因在于进程线程都是笨重的家伙。让一个进(线)程只处理一个用户请求确确实实是有点浪费
就比如饭店的服务员和顾客。
一个进(线)程同时只能处理一个用户请求，就相当于一个服务员只能照顾一个顾客，服务员为这个顾客点完餐，才能去下一个顾客。如果同时来了 1000 个顾客，那就得 1000 个服务员，这样的人力成本很高。

如何高效的提升性能？很简单，就像现在扫码点餐一样，哪个顾客想点餐，只需要扫描桌子上的二维码，然后服务员就知道哪个顾客点了餐。

这就是多路复用。多路指的是许许多多个用户的网络连接。复用指的是对进(线)程的复用。换到餐厅的例子里，就是一群顾客只要一个服务员来处理就行了。

不过复用实现起来是需要特殊的 socket 事件管理机制的，一共有select，poll，epoll三种机制，最典型和高效的方案就是 epoll。放到餐厅的例子来，epoll 就相当于桌子上的二维码，顾客只需要扫描二维码，就可以通知到服务员。

关于这三种方式的底层实现，以及LUNIX如何处理， 这里不多赘述，可在主页的 **《》** 中查看。

在 epoll 的系列函数里， epoll_create 用于创建一个 epoll 对象，epoll_ctl 用来给 epoll 对象添加或者删除一个 socket。epoll_wait 就是查看它当前管理的这些 socket 上有没有可读可写事件发生。

当网卡接收到数据包后，Linux 内核通过中断或轮询的方式将数据拷贝到内核缓冲区，并触发网络协议栈进行处理。经过 IP 层、TCP 层等协议解析后，数据最终被送入对应 socket 的接收缓冲区（Receive Queue）。此时，内核会检测该 socket 是否已被某个 epoll 实例所监控。

如果该 socket 正在被 epoll 监听，并且事件类型（如 EPOLLIN）匹配，内核就会将对应的事件信息封装成 epoll_event 结构体，并添加到 epoll 的就绪队列（ready list）中。这样一来，用户线程在执行 epoll_wait() 时，内核只需检查就绪队列中是否已有事件即可，无需重新遍历所有 socket，从而实现了高效的 I/O 多路复用。

在基于 epoll 的事件驱动编程中，与传统“谁想处理、谁就直接调用”的函数调用模式不同，我们无法主动去调用某个 API 来处理事件。原因是：我们并不知道具体的事件（比如：网络连接、数据到达）会在什么时候发生。

因此，我们采用一种“被动响应”的方式：提前把感兴趣的事件（比如 socket 可读、可写）和对应的处理函数注册到一个“事件分发器”中（比如 epoll）。当事件真的发生时，操作系统通知我们，事件分发器就会自动调用我们事先写好的处理逻辑，这种被自动“回调”的处理方式就是所谓的事件驱动模型。

这种通过注册事件和回调函数、并由事件分发器统一调度的编程思想，被称为 Reactor 模型。它是一种高效处理高并发 I/O 的经典架构，在很多高性能服务器程序中广泛使用，比如 Nginx、Redis、Netty 等。

## 二、Redis 服务启动初始化
在理解了 epoll 原理后，我们从源码中看 Redis 具体是如何使用 epoll 的。直接在 Github 上就可以非常方便地获取 Redis 的源码。
其中整个 Redis 服务的代码总入口在 src/server.c 文件中，把入口函数的核心部分摘了出来，如下。

```c
//file: src/server.c
int main(int argc, char **argv) {
    ......
    // 启动初始化
    initServer();
    // 运行事件处理循环，一直到服务器关闭为止
    aeMain(server.el);
}
```
Redis 做了这么三件重要的事情。
1.创建一个 epoll 对象
2.对配置的监听端口进行 listen
3.把 listen socket 让 epoll 给管理起来

```c
//file: src/server.c
void initServer() {
    // 2.1.1 创建 epoll
    server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);

    // 2.1.2 绑定监听服务端口
    listenToPort(server.port,server.ipfd,&server.ipfd_count);

    // 2.1.3 注册 accept 事件处理器
    for (j = 0; j < server.ipfd_count; j++) {
        aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL);
    }
    ...
}
```
**2.1 创建 epoll 对象：事件循环的初始化过程**

Redis 在启动过程中，会初始化事件驱动模块，构建一个完整的事件循环（event loop）结构。在这过程中，它内部封装了 epoll，并将其用于高效的 I/O 多路复用处理。

Redis 的事件循环初始化逻辑位于 `aeCreateEventLoop()` 函数中，调用路径为：

```
server.c → initServer() → aeCreateEventLoop()
```
###  创建 epoll 对象的核心数据结构

首先来看 `redisServer` 中保存 epoll 实例的成员变量：

```c
// file: src/server.c
struct redisServer {
    ...
    aeEventLoop *el;  // 这个变量叫 el，它保存了事件循环这套系统的地址，Redis 通过它来处理所有的 I/O 和定时事件。
    ...
};
```

Redis 使用 `aeEventLoop` 结构体封装了对 epoll 的调用细节，所有事件驱动相关的逻辑（事件注册、处理、分发）都围绕它展开。

---


以下是 `aeCreateEventLoop()` 函数的核心实现，它位于 `src/ae.c` 中：

```c
// file: src/ae.c
aeEventLoop *aeCreateEventLoop(int setsize) {
    aeEventLoop *eventLoop;

    // 分配事件循环结构体
    eventLoop = zmalloc(sizeof(*eventLoop));

    // 分配 I/O 事件数组，用于保存注册的事件（可读/可写）
    eventLoop->events = zmalloc(sizeof(aeFileEvent) * setsize);

    // 初始化 epoll（底层 I/O 多路复用机制）
    aeApiCreate(eventLoop);

    return eventLoop;
}
```

>  **说明：**
>
> * `setsize` 表示最多监听多少个文件描述符（FD）
> * `events` 是一个数组，存储每个 FD 上注册的事件（如 EPOLLIN / EPOLLOUT）
> * `aeApiCreate()` 是一个平台相关的初始化函数，底层对 epoll 进行封装

---

###  aeEventLoop 结构体概览

```c
// file: src/ae.h
typedef struct aeEventLoop {
    ...
    aeFileEvent *events;   // 注册的事件数组
    void *apidata;         // 指向 epoll 封装的私有数据（如 epoll fd）
    ...
} aeEventLoop;
```

---

 epoll 实例的创建（aeApiCreate）

Redis 针对不同平台提供了不同的实现，例如 epoll、kqueue、select。epoll 的实现位于 `ae_epoll.c` 中：

```c
// file: src/ae_epoll.c
static int aeApiCreate(aeEventLoop *eventLoop) {
    aeApiState *state;

    // 分配内部 epoll 封装状态结构
    state = zmalloc(sizeof(aeApiState));

    // 创建 epoll 文件描述符（大小 1024 已被内核忽略）
    state->epfd = epoll_create(1024);

    // 将 epoll 状态保存在事件循环结构中
    eventLoop->apidata = state;

    return 0;
}
```
>
> * `epoll_create(1024)` 创建了一个 epoll 实例，返回一个 epoll 文件描述符（epfd），用于后续调用 `epoll_ctl`、`epoll_wait`。
> * `aeApiState` 是对 epoll 的轻量封装，包含 `epfd` 和事件数组等字段。

* `aeCreateEventLoop()` 是 Redis 构建事件驱动框架的起点。
* 它封装了 epoll 的底层调用，构建了 `aeEventLoop` 对象，后续所有事件注册、触发与分发均基于该结构进行。
* epoll 的 `epfd` 会被保存在 `eventLoop->apidata` 中，供后续 `epoll_ctl` 和 `epoll_wait` 调用使用。

