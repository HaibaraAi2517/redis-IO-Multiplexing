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
## 2.1 创建 epoll 对象：事件循环的初始化过程

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

  当然可以，下面是对你提供的这段 Redis 监听服务端口过程的专业化、系统化描述，适合用于简历项目经验或面试答辩中，让面试官感受到你对底层细节的理解：

---

## 2.2 Redis 监听服务端口的底层实现流程解析

在 Redis 启动阶段，监听端口的核心逻辑集中在 `listenToPort` 函数中，其主要作用是初始化服务器监听套接字，为后续客户端连接建立基础。

### 2.2.1. 支持多端口绑定

```c
int listenToPort(int port, int *fds, int *count);
```

Redis 支持多网卡地址绑定（例如 `127.0.0.1` 和 `0.0.0.0` 同时监听），因此在 `listenToPort` 中通过一个 `for` 循环，遍历所有配置的 `bindaddr`，并多次调用 `anetTcpServer`，为每个地址创建监听套接字。成功创建的 `fd` 会记录在传入的 `fds` 数组中。

### 2.2.2. 抽象封装 `anetTcpServer`

调用链如下：

```
listenToPort -> anetTcpServer -> _anetTcpServer -> anetListen
```

其中，`anetTcpServer` 是对网络操作的封装，内部实际调用 `_anetTcpServer` 来完成 TCP 套接字创建和监听。

```c
int anetTcpServer(char *err, int port, char *bindaddr, int backlog) {
    return _anetTcpServer(err, port, bindaddr, AF_INET, backlog);
}
```

### 2.2.3. 关键系统调用封装

在 `_anetTcpServer` 中，完成以下核心系统调用：

* **创建套接字**（socket）
  `socket(AF_INET, SOCK_STREAM, 0)`：创建一个 TCP 套接字。

* **设置端口复用选项**
  `setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, ...)`：避免端口被 TIME\_WAIT 占用导致服务重启失败。

* **绑定地址**

  ```c
  bind(s, sa, len);
  ```

  将套接字绑定到指定 IP 和端口。`sa` 为 `sockaddr_in` 类型地址结构体。

* **监听连接**

  ```c
  listen(s, backlog);
  ```

  设置监听套接字的连接请求队列长度，开启监听状态。

这些操作最终在 `anetListen` 中实现，该函数是对底层系统调用 `bind()` 和 `listen()` 的封装，并处理可能的错误信息。

### 4. 安全性与健壮性设计

整个监听流程中，Redis 通过封装 `anet.c` 模块，提供了统一的错误处理、跨平台支持、参数校验、日志记录等机制，提升了网络模块的健壮性与可维护性。

---

* Redis 通过抽象封装网络模块，将复杂的系统调用封装在 `anet.c`，提高代码可读性和跨平台兼容性。
* 支持多网卡绑定与端口复用，适用于高可用、容灾部署需求。
* 所有监听套接字最终会注册到 `aeEventLoop`（事件循环）中，实现基于 `epoll` 的 I/O 多路复用，配合事件驱动模型，高效处理并发连接。

  当然可以，以下是润色后的内容，使其更专业、更具逻辑性和条理性，同时保留原意，突出对系统底层机制的深入理解：

---

### 2.3 注册事件回调函数

我们再次回顾 `initServer` 函数的执行流程。在前2.2中，`initServer` 依次完成了以下关键步骤：

1. 通过 `aeCreateEventLoop` 创建了事件循环对象（底层封装了 epoll 实例），用于后续的 I/O 多路复用管理。
2. 通过 `listenToPort` 完成了对服务端口的 `bind` 与 `listen` 操作，初始化监听 socket。
3. 最关键的一步是，调用 `aeCreateFileEvent` 注册了 `accept` 事件的处理回调。

```c
// file: src/server.c
void initServer(void) {
    // 2.1.1 创建 epoll 实例并初始化事件循环
    server.el = aeCreateEventLoop(server.maxclients + CONFIG_FDSET_INCR);

    // 2.1.2 监听指定服务端口
    listenToPort(server.port, server.ipfd, &server.ipfd_count);

    // 2.1.3 注册 accept 事件处理器
    for (j = 0; j < server.ipfd_count; j++) {
        aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
                          acceptTcpHandler, NULL);
    }
}
```

可以看到，在调用 `aeCreateFileEvent` 时，传入了一个非常关键的回调函数指针 `acceptTcpHandler`。这个函数将在监听 socket 上有新连接到来时被触发调用，处理连接的 `accept` 操作。下面我们来看 `aeCreateFileEvent` 的实现细节：

```c
// file: src/ae.c
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
                      aeFileProc *proc, void *clientData)
{
    // 获取指定 fd 对应的事件结构体
    aeFileEvent *fe = &eventLoop->events[fd];

    // 封装底层 epoll_ctl 调用，添加感兴趣的事件
    aeApiAddEvent(eventLoop, fd, mask);

    // 设置事件的类型和处理函数
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;

    // 设置私有数据指针，便于回调中使用
    fe->clientData = clientData;

    return AE_OK;
}
```

此函数的核心逻辑如下：

* 首先从 `eventLoop->events` 数组中获取了 fd 对应的 `aeFileEvent` 实例。在前面我们已介绍，`eventLoop->events` 是 Redis 用于存储所有文件描述符相关事件状态的数组。
* 接着调用 `aeApiAddEvent` 函数，封装对 `epoll_ctl` 的调用。具体代码如下：

```c
// file: src/ae_epoll.c
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    int op = (eventLoop->events[fd].mask == AE_NONE) ?
             EPOLL_CTL_ADD : EPOLL_CTL_MOD;

    // 构造 epoll_event 并调用 epoll_ctl
    epoll_ctl(state->epfd, op, fd, &ee);

    return 0;
}
```

通过这一操作，Redis 成功将 fd 注册到 epoll 实例中，监听对应的读/写事件。epoll 在后续调用 `epoll_wait` 时就可以检测这些事件。

此后，每一个 `aeFileEvent` 实例上，都可能包含以下三项关键信息：

* `rfileProc`：可读事件触发时的回调函数。
* `wfileProc`：可写事件触发时的回调函数。
* `clientData`：用于回调时传入的私有上下文数据。

当某个 fd 上有事件触发时，Redis 会从 `eventLoop->events[fd]` 中读取对应的 `aeFileEvent`，并根据事件类型调用相应的回调处理函数（例如 `rfileProc`）。

回到 `initServer` 中注册监听 fd 的事件逻辑：

```c
// file: src/server.c
for (j = 0; j < server.ipfd_count; j++) {
    aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
                      acceptTcpHandler, NULL);
}
```

这里 Redis 为监听 socket 注册了 `AE_READABLE` 事件，回调函数是 `acceptTcpHandler`，也就是说，当有新的客户端连接请求到达时，epoll 检测到该 fd 可读，随后触发 `acceptTcpHandler` 来执行实际的连接建立操作。

注意：

* 只设置了读事件（AE\_READABLE），未设置写事件（AE\_WRITABLE），因为监听 socket 通常只关注“是否有新连接请求到达”这一事件。
* `clientData` 被设置为 `NULL`，说明此阶段没有传递额外上下文数据。

通过这一流程，Redis 的事件驱动模型得以运行起来。当后续调用 `aeMain` 启动事件循环后，所有注册的事件处理器将随 epoll 的通知被触发执行，实现高效的非阻塞 I/O 通信机制。

---

1. **epoll 的封装思路**：Redis 将 epoll 封装为独立模块，便于跨平台扩展。
2. **事件驱动模型解耦性强**：事件注册和事件触发逻辑分离，利于维护。
3. **注册回调机制简洁高效**：通过设置 `rfileProc` / `wfileProc` 实现不同事件类型的响应。


## 三、redis的aeMain循环事件处理

---

###  Redis 启动后正式进入事件循环 —— `aeMain`

在前一节中，我们已经完成了 Redis 启动初始化的核心流程：

* 创建了 epoll 实例（通过 `aeCreateEventLoop`），
* 绑定并监听了服务端口（`listenToPort`），
* 并为监听 socket 注册了 `accept` 事件处理器（`acceptTcpHandler`）。

这些准备工作完成后，Redis 会进入主循环函数 `aeMain`，真正开始响应客户端请求。

```c
// file: src/server.c
int main(int argc, char **argv) {
    ...
    // 初始化服务器（创建 epoll、绑定端口、注册回调）
    initServer();

    // 启动事件循环，直到服务器关闭
    aeMain(server.el);
}
```

### aeMain：Redis 核心事件驱动循环

`aeMain` 是 Redis 的主事件循环。它本质上是一个无限循环结构，每一轮循环负责处理一次 I/O 多路复用轮询 + 用户逻辑的触发。其执行逻辑概括如下：

```c
// file: src/ae.c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {

        // 3.4 3.4 beforesleep 处理写任务队列并实际发送（后面会重点讲到）
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);

        // 3.1-3.3 调用 epoll_wait 等待并处理所有事件
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}
```

Redis 在每轮循环中要完成以下 **四件关键任务**：

---

### 3.1 epoll_wait 事件发现机制

在 Redis 中，无论有多少客户端连接，所有的网络事件（包括连接请求的 accept 事件、数据可读事件以及数据可写事件）都统一通过 `epoll_wait` 进行检测和管理。甚至包括定时器等时间事件，也被整合进 `epoll_wait` 的等待时长中，实现了对所有事件的统一调度与高效处理。


当 `epoll_wait` 监测到某个文件描述符（fd）上的事件发生时，Redis 会触发相应的、事先注册好的回调函数来处理这些事件。具体来看，`aeProcessEvents` 函数对 `epoll_wait` 进行了封装，负责事件的等待和分发，代码逻辑如下：

```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    // 计算最近一个时间事件的触发时间，作为阻塞等待的超时时间
    tvp = /* 计算时间事件超时时间 */;

    // 通过底层封装函数调用 epoll_wait，等待事件发生，等待时间由时间事件决定
    numevents = aeApiPoll(eventLoop, tvp);

    // 遍历所有已就绪的事件
    for (j = 0; j < numevents; j++) {
        // 取出对应 fd 的事件结构体
        aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];

        // 如果该事件是读事件且有对应的读事件处理函数，则调用它
        if (fe->mask & AE_READABLE && fe->rfileProc)
            fe->rfileProc(eventLoop, eventLoop->fired[j].fd, fe->clientData, AE_READABLE);

        // 如果该事件是写事件且有对应的写事件处理函数，则调用它
        if (fe->mask & AE_WRITABLE && fe->wfileProc)
            fe->wfileProc(eventLoop, eventLoop->fired[j].fd, fe->clientData, AE_WRITABLE);
    }
}
```

`aeProcessEvents` 核心在于调用 `aeApiPoll`，后者对底层系统调用 `epoll_wait` 进行了封装：

```c
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;

    // 将 epoll_wait 的超时时间转换为毫秒，若无时间限制则传 -1（阻塞）
    int timeout = tvp ? (tvp->tv_sec * 1000 + tvp->tv_usec / 1000) : -1;

    // 调用 epoll_wait，等待事件发生
    int numevents = epoll_wait(state->epfd, state->events, eventLoop->setsize, timeout);

    // 事件的就绪状态会存储在 state->events 中，后续由 aeProcessEvents 处理
    return numevents;
}
```

总结来说，Redis 利用 `epoll_wait` 高效地监听多个文件描述符上的事件，结合事件循环机制统一调度读写事件和时间事件，保证单线程下的高并发性能。这种设计不仅实现了 IO 多路复用，更保证了事件处理的响应及时和资源的合理利用，体现了 Redis 网络模型设计的高效与优雅。

---


### 3.2 新连接请求如何处理

假设有一个新的用户连接到达 Redis 服务器。此前我们看到，Redis 在监听 socket 上注册的读事件回调函数是 `acceptTcpHandler`，也就是说，一旦有新的连接请求到来，`acceptTcpHandler` 就会被触发执行。

在 `acceptTcpHandler` 中，Redis 主要完成了三项关键操作：

1. 调用 `accept` 系统调用，接收客户端的连接请求，拿到新的连接 socket；
2. 为这个新连接创建一个唯一的 `redisClient` 对象，用于管理该客户端的所有状态和数据；
3. 将新连接添加到事件循环（epoll）中，并注册读事件的处理函数，以便后续读取客户端发来的请求。

接下来，我们详细拆解这三个步骤具体是如何实现的。

首先，在 `acceptTcpHandler` 函数中调用了 `anetTcpAccept`，它内部最终执行的就是 Linux 的 `accept` 系统调用，用于接收新连接并返回新 socket 文件描述符：

```c
int anetTcpAccept(...) {
    return anetGenericAccept(...);
}
static int anetGenericAccept(...) {
    return accept(s, (struct sockaddr*)&sa, &salen);
}
```

拿到新连接 socket 后，`acceptTcpHandler` 调用 `acceptCommonHandler`，在这里为该 socket 创建一个对应的 `redisClient` 对象：

```c
static void acceptCommonHandler(int fd, int flags) {
    redisClient *c = createClient(fd);
    ...
}
```

`createClient` 是创建客户端连接对象的核心函数。在这里，它通过动态分配内存创建一个 `redisClient`，并且最关键的是，**它会为该连接注册一个读事件处理函数 `readQueryFromClient`**，这样一旦客户端发送数据，Redis 能够及时读取并处理：

```c
redisClient *createClient(int fd) {
    redisClient *c = zmalloc(sizeof(redisClient));
    if (fd != -1) {
        aeCreateFileEvent(server.el, fd, AE_READABLE, readQueryFromClient, c);
    }
    ...
}
```

这里的 `aeCreateFileEvent` 会把新连接 socket 的读事件与 `readQueryFromClient` 函数绑定，并把当前 `redisClient` 对象作为私有数据传递，从而完成事件驱动模型下的客户端请求处理准备。

总结来看，Redis 在新连接到达时，通过 `acceptTcpHandler` 接收连接，创建对应客户端对象，并将该连接注册到事件循环中，完成了从连接建立到请求准备的完整链路。这种设计保证了 Redis 能高效且高并发地处理大量客户端连接。

---


### 3.3 处理客户端连接上的可读事件

当已经连接的客户端通过连接发送命令（比如 `GET XXXXX_KEY`）时，Redis 的事件循环会检测到该连接的 socket 上出现了可读事件。此时，之前为该连接注册的读事件处理器 `readQueryFromClient` 会被触发执行。

在 `readQueryFromClient` 中，Redis 完成了以下几步关键操作：

1. **解析命令内容**
2. **查找并执行命令**
3. **将响应结果写入输出缓冲区**
4. **将当前 client 添加到待写队列，等待数据返回**

我们来看具体实现流程：

---

#### 3.3.1. 触发读事件，读取并处理命令

```c
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, ...) {
    redisClient *c = (redisClient*) privdata;
    processInputBufferAndReplicate(c);
}
```

这个函数会进一步调用 `processInputBuffer` 来解析并处理客户端命令：

```c
void processInputBuffer(redisClient *c) {
    processCommand(c);  // 执行命令
}
```

---

####  3.3.2. 查找命令并执行

`processCommand` 是命令处理的核心函数，它首先根据客户端输入查找命令定义，并检查合法性：

```c
int processCommand(redisClient *c) {
    c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);

    // 事务模式下入队，否则直接调用处理函数
    if (c->flags & CLIENT_MULTI && ...) {
        queueMultiCommand(c);
    } else {
        call(c, CMD_CALL_FULL);  // 直接处理命令
    }
    return C_OK;
}
```

---

#### 3.3.3. 调用具体命令处理函数

在 `call` 函数中，Redis 会调用命令所对应的处理函数，例如 `GET` 命令会对应 `getCommand`：

```c
void call(client *c, int flags) {
    c->cmd->proc(c);  // 调用命令对应函数，如 getCommand
}
```

```c
void getCommand(client *c) {
    getGenericCommand(c);
}
```

---

#### 3.3.4. 执行 getGenericCommand 查找数据并写入缓冲区

`getGenericCommand` 是实际处理逻辑，它会尝试从内存中查找 key，并将结果写入输出缓冲区：

```c
int getGenericCommand(client *c) {
    robj *o;
    if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.null[c->resp])) == NULL)
        return C_OK;

    addReplyBulk(c, o);  // 将响应数据写入缓冲区
    return C_OK;
}
```

---

#### 3.3.5. 将响应数据写入输出缓冲区

`addReplyBulk` 会调用 `addReply` 来完成核心的缓冲区写入逻辑：

```c
void addReply(client *c, robj *obj) {
    if (prepareClientToWrite(c) != C_OK) return;

    if (sdsEncodedObject(obj)) {
        if (_addReplyToBuffer(c, obj->ptr, sdslen(obj->ptr)) != C_OK)
            _addReplyStringToList(c, obj->ptr, sdslen(obj->ptr));
    }
}
```

这个函数完成了两个关键任务：

1. **将 client 添加到等待写入任务队列中（`clients_pending_write`）**
2. **将响应写入输出缓冲区buf（response buffer）**

其中 `prepareClientToWrite` 的作用就是将当前客户端标记为等待写操作，并加入到 `server.clients_pending_write` 队列中：

```c
int prepareClientToWrite(client *c) {
    if (!clientHasPendingReplies(c) && !(c->flags & CLIENT_PENDING_READ))
        clientInstallWriteHandler(c);
}

void clientInstallWriteHandler(client *c) {
    c->flags |= CLIENT_PENDING_WRITE;
    listAddNodeHead(server.clients_pending_write, c);
}
```

---

#### 3.3.6. 写入固定大小的 response buffer

实际的响应内容写入是通过 `_addReplyToBuffer` 完成的：

```c
int _addReplyToBuffer(client *c, const char *s, size_t len) {
    memcpy(c->buf + c->bufpos, s, len);  // 拷贝到输出缓冲区
    c->bufpos += len;
    return C_OK;
}
```

如果缓冲区满了，还可以退回到链表式的缓冲机制 `_addReplyStringToList` 中继续写入 reply，确保大响应也能正确发送。

---

### 整个流程从客户端发送命令开始，到命令处理、生成响应并写入缓冲区，再加入写队列等待返回，展现了 Redis 高效的事件驱动架构：

* **事件触发精准（epoll）**
* **命令处理高效（函数指针 + 命令表）**
* **响应回写机制完善（buffer + pending\_write）**

这样的设计既保证了性能，又具备良好的扩展性

 ## 这时，只是将client添加到任务队列中，并没有真正的处理。真正的处理过程在beforesleep这个关键的函数里



### 3.4 beforesleep 关键操作：处理写任务队列

在 Redis 的主事件循环 `aeMain` 中，每次调用 `aeProcessEvents` 处理事件之前，都会先执行一个函数：`beforesleep`。
虽然名字看起来像是“睡觉前的准备”，但它所承担的责任却十分关键，尤其是**处理客户端写任务队列并将数据真正写回客户端**。

#### 3.4.1.`aeMain` 主循环中的 `beforesleep`

```c
// file: src/ae.c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        // beforesleep: 处理写任务队列并实际发送
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);

        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}
```

每次进入 `aeProcessEvents` 之前都会调用 `beforesleep`，而它在实际中负责处理很多关键性操作，其中一个重要任务就是**将响应数据写回客户端**。

---

#### 3.4.2.`beforeSleep` 的核心：`handleClientsWithPendingWrites`

```c
// file: src/server.c
void beforeSleep(struct aeEventLoop *eventLoop) {
    ...
    handleClientsWithPendingWrites();
}
```

接下来我们看看 `handleClientsWithPendingWrites` 的实现：

```c
// file: src/networking.c
int handleClientsWithPendingWrites(void) {
    listIter li;
    listNode *ln;
    int processed = listLength(server.clients_pending_write);

    listRewind(server.clients_pending_write, &li);
    while ((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_WRITE;
        listDelNode(server.clients_pending_write, ln);

        // 实际将响应数据发送到客户端
        writeToClient(c->fd, c, 0);

        // 如果没发送完，注册写事件等待 epoll 回调
        if (clientHasPendingReplies(c)) {
            aeCreateFileEvent(server.el, c->fd, AE_WRITABLE,
                              sendReplyToClient, c);
        }
        ...
    }
}
```

这个函数的核心逻辑是：
* 遍历 `server.clients_pending_write` 写任务队列；
* 调用 `writeToClient` 尝试把缓冲区中的响应数据发送到客户端；
* 如果一次写不完，就注册 `AE_WRITABLE` 写事件，让 epoll 下次可写时继续发送。
* 注册一个写事件处理器sendReplyToClient，等待 epoll_wait 发现可写后再处理 

#### 3.4.3.核心写操作：`writeToClient`

```c
// file: src/networking.c
int writeToClient(int fd, client *c, int handler_installed) {
    while (clientHasPendingReplies(c)) {
        // 发送固定缓冲区
        if (c->bufpos > 0) {
            nwritten = write(fd, c->buf + c->sentlen, c->bufpos - c->sentlen);
            if (nwritten <= 0) break;
            ...
        }
        // 发送链表缓冲中的剩余数据
        else {
            o = listNodeValue(listFirst(c->reply));
            nwritten = write(fd, o->buf + c->sentlen, objlen - c->sentlen);
            ...
        }
    }
}
```

在 `writeToClient` 中，Redis 会根据当前的响应缓冲状态：

* 先尝试写出固定缓冲区 `c->buf` 的数据；
* 如果固定缓冲写完，就进入动态链表 `c->reply` 中，逐条写出数据对象。

writeToClient 的核心逻辑是通过 write 系统调用，将 Redis 命令的处理结果写入客户端对应的 socket，由内核负责将数据实际发送出去。
由于每条命令的响应数据大小不确定，Redis 采用了**固定大小的缓冲区（c->buf）与可变长度的回复链表（c->reply）**相结合的方式来存储待发送数据。
发送时，Redis 会优先发送固定缓冲区中的内容，当该缓冲区发送完毕后，再继续发送链表中的回复对象。这样设计既兼顾了小响应的发送效率，也支持了大数据量的分段输出，从而实现了高性能的响应机制

---

### 为什么写操作可能写不完？

一次 `write()` 调用并不保证数据全部写出，原因如下：

* 操作系统对每个 socket 的发送缓冲区有限制；
* 如果发送内容太大、或者客户端读取太慢，就会导致**写入阻塞或中断**。

因此，Redis 判断 `clientHasPendingReplies` 返回 true 时，就意味着还有数据未发送完，这时 Redis 会使用 `epoll` 注册写事件等待下一轮发送，避免阻塞主线程。

### 这是具体流程：

客户端发送命令
    ↓
Redis 处理命令，生成响应
    ↓
将响应写入 c->buf / c->reply
    ↓
调用 writeToClient 尝试发送
    ↓
[是否全部发送成功？]
 ├─ 是：不注册写事件，任务完成
 └─ 否：注册 AE_WRITABLE，绑定 sendReplyToClient
            ↓
    epoll 检测到 socket 可写
            ↓
    sendReplyToClient 被调用 → 继续发送

 ## 对于AE_WRITABLE和sendReplyToClient的关系，可以这样理解：“如果 socket 可写了（AE_WRITABLE），请调用 sendReplyToClient() 来处理它。”


## 四、总结 在 Redis 中，尽管服务器端处理是单线程的，但却能轻松支撑每秒数万 QPS 的高并发性能

在 Redis 中，尽管服务器端处理是单线程的，但却能轻松支撑**每秒数万 QPS** 的高并发性能。这背后的关键，就是对 **Linux 提供的 I/O 多路复用机制——`epoll` 的精妙运用**。

其实，Redis 的核心逻辑可以浓缩为两个关键函数：

### 1. `initServer` —— 初始化服务器

### 2. `aeMain` —— 启动事件循环

只要理解了这两个函数，Redis 的运行机制就掌握了大半。

---

### 入口：main 函数

```c
// file: src/server.c
int main(int argc, char **argv) {
    ...
    initServer();             // 初始化服务，监听端口等
    aeMain(server.el);        // 启动事件循环，直到 Redis 关闭
}
```

`main` 函数是 Redis 的入口，调用 `initServer` 完成服务初始化，然后进入 `aeMain`，启动事件循环系统。

---

### 服务初始化：`initServer`

在 `initServer` 函数中，Redis 完成了三件非常关键的准备工作：

1. **创建 epoll 实例（通过 `aeCreateEventLoop`）**
   这是 Redis 对 epoll 多路复用机制的封装，负责后续所有 fd 的事件注册与监听。

2. **监听配置的端口（调用 `listen` 创建 listen socket）**
   Redis 支持多个端口监听，会创建一个或多个 server socket。

3. **将 listen socket 注册到 epoll 中，监听连接事件**
   使用 `aeCreateFileEvent`，将 listen fd 与 `AE_READABLE` 事件绑定，当有新连接到来时被 epoll 通知。

---

### 事件主循环：`aeMain`

这是 Redis 最核心的运行机制，采用单线程死循环：

```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep)
            eventLoop->beforesleep(eventLoop);  // 发送前准备，比如处理写任务队列

        aeProcessEvents(eventLoop, AE_ALL_EVENTS);  // 处理所有事件（读/写/定时等）
    }
}
```

主循环的每一轮都做了这些事情：

#### 1. 使用 `epoll_wait` 等待事件发生

监听所有已注册 fd 的可读、可写等事件，一旦 socket 状态准备就绪（如有数据可读），马上返回处理。

#### 2. 处理监听 socket 上的新连接

当 `listen socket` 有连接到达，会调用连接处理器 `acceptTcpHandler`，接收客户端连接并注册到 epoll。

#### 3. 处理客户端命令

如果某个客户端 socket 上有数据可读，Redis 会读取数据、解析命令并执行，然后把响应结果写入该 client 的输出缓冲区。

#### 4. 加入写任务队列

命令结果暂时不会立刻写回客户端，而是加入写任务队列 `server.clients_pending_write`，待本轮末尾统一处理。

#### 5. `beforeSleep`：写任务最终发送

每轮事件处理前，会调用 `beforeSleep`。它会遍历写任务队列，调用 `writeToClient` 将数据真正通过 `write()` 系统调用写入 socket。

---

### 重点细节：**写不完怎么办？**

因为每个 socket 的内核发送缓冲区是有限的，**一条命令的响应如果太大，可能一次写不完**。这种情况，Redis 的处理非常巧妙：

1. `writeToClient` 如果发现还有数据未发送完（例如大对象或网络阻塞），就不会死等；
2. 而是调用：

   ```c
   aeCreateFileEvent(server.el, c->fd, AE_WRITABLE, sendReplyToClient, c);
   ```

   **注册 `AE_WRITABLE` 写事件，并绑定处理函数 `sendReplyToClient`**；
3. 下一轮 `epoll_wait` 发现该 socket 可写时，就调用 `sendReplyToClient` 补发剩余数据；
4. 数据发完后再取消写事件监听，避免 epoll 不断唤醒浪费资源。

---

###  单线程也能高并发的核心逻辑

| 机制              | 描述                                      |
| --------------- | --------------------------------------- |
| **epoll + 单线程** | 所有 fd 用 epoll 统一管理，无需多线程上下文切换           |
| **统一事件循环**      | `aeMain` 实现所有网络、命令处理、延迟任务的调度            |
| **读写分离处理**      | 命令处理后不立即写出，而是加入队列，在 `beforeSleep` 中统一处理 |
| **写事件自动注册**     | 如果响应数据太大写不完，自动注册写事件，后续继续发送              |

---

通过这种模型，Redis 实现了**单线程驱动、事件驱动、I/O 非阻塞、网络极致优化**的高性能架构。







  


