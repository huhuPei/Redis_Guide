# 事件驱动库

AE 是一个简易的事件驱动模型，使用单线程处理所有的事件，避免了多线程上下文切换的成本与潜在资源竞争，简单又高效；使用 IO 多路复用技术提高了网络服务的并发处理能力。

### 1. 接口设计
由于不同 OS 内核的 polling API 是不同的，为了兼容不同的内核，redis 将不同平台的 polling API 封装成相同的调用接口，通过预编译的方式将其隔离。通过这种特殊的方式，实现了面向对象统一接口、隔离实现的功能。
```
// ae.c
#ifdef HAVE_EPOLL
#include "ae_epoll.c"
#else
    #ifdef HAVE_KQUEUE
    #include "ae_kqueue.c"
    #else
    #include "ae_select.c"
    #endif
#endif
```
当平台是 linux 时，将会使用 ae_epoll.c 的接口实现，平台的定义见于 config.h 文件。   

各个接口的作用：
aeApiCreate()：创建与事件 polling 相关的资源，linux 是创建epoll 实例；   
aeApiFree()：释放之前分配的资源；   
aeApiAddEvent()：增加对指定 fd 读/写事件的监听；  
aeApiDelEvent()：删除对指定 fd 读/写事件的监听；  
aeApiPoll()：获取所有触发的事件，标记相关 fd 为就绪状态。  

### 2. 事件类型
AE 包含两种事件：文件事件、时间事件    

#### 2.1 文件事件   
本质是网络 IO 事件，Unix-Like 系统中 socket 也是用 fd 表示，所以 socket 读写也可以用文件读写表示。
```
// ae.h
typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE|EXCEPTION) */
    aeFileProc *rfileProc;
    aeFileProc *wfileProc;
    aeFileProc *efileProc;
    void *clientData;
} aeFileEvent;
```
从结构体可以看出，事件包含事件类型和事件的处理函数，事件类型可以是读/写/读写，读写可以分别指定处理函数。

#### 2.2 时间事件  
本质是定时任务，指定一个未来触发时间，如果当前时间超过这个时间就会进行处理。
```
// ae.h
typedef struct aeTimeEvent {
    long long id; /* time event identifier. */
    long when_sec; /* seconds */
    long when_ms; /* milliseconds */
    aeTimeProc *timeProc;
    aeEventFinalizerProc *finalizerProc;
    void *clientData;
    struct aeTimeEvent *next;
} aeTimeEvent;
```
触发时间保存在 when_sec 和 when_ms 中。   

**循环运行**    
事件处理函数 timeProc 的返回值可以决定时间事件的运行次数。    
当返回值是 AE_NOMORE 时，表示该事件不再触发；当返回值是大于 0 的值时，表示下次触发的时间间隔。因此，想要循环运行一个事件，处理函数只需返回一个固定的值即可。

返回值能否等于 0 值？   
如果时间下次触发的间隔被设置为 0，那么每次事件循环都会处理这个事件，代码上是可行的，但逻辑上讲不能等于 0，因为没有什么作用。

### 3. 事件管理器
负责注册事件和删除事件。
```
typedef struct aeEventLoop {
    int maxfd;
    long long timeEventNextId;
    aeFileEvent events[AE_SETSIZE]; /* Registered events */
    aeFiredEvent fired[AE_SETSIZE]; /* Fired events */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* This is used for polling API specific data */
} aeEventLoop;
```

#### 3.1 文件事件注册
文件事件使用数组存储，通过 fd 进行定位，使用位运算注册事件。
```
// 注册 fd 读事件
aeFileEvent *fe = &eventLoop->events[fd];
fe->mask |= AE_READABLE;
// 注册 fd 写事件
fe->mask |= AE_WRITABLE;

// 删除 fd 读事件
fe->mask = fe->mask & (~AE_READABLE);

// 当 fd 没有任何事件时，mask 等于 AE_NONE
fe->mask == AE_NONE is true
```

#### 3.2 时间事件注册
时间事件使用链表存储，注册事件会添加到链表头部。
```
// 注册时间事件
aeTimeEvent *te = zmalloc(sizeof(*te));
te->next = eventLoop->timeEventHead;
eventLoop->timeEventHead = te;
```

### 4. 事件处理
由 aeProcessEvents() 进行处理，通过 flags 设置不同运行方式，默认是处理所有事件，包括文件和时间。   

**运行逻辑：**   
1、先处理文件事件，再处理时间事件；    
2、若设置了 AE_DONT_WAIT，当没有可以处理的文件事件时，poll 不会等待文件事件触发，将直接返回，然后去处理时间事件；    
3、若没有设置 AE_DONT_WAIT（默认），如果有可处理的时间事件，poll 将等待一定时间，否则将阻塞等待直到有文件事件触发。    

默认情况下，redis 会有循环运行的时间事件，所以 poll 都是等待一定时间，没有 IO 事件便会返回，这也算是非阻塞的一种方式，有等待时间的非阻塞。网络上流传的 redis 非阻塞 IO 是不是指的这个呢？    

事件处理代码感觉比较绕。
```
if (eventLoop->maxfd != -1 ||
    ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
    int j;
    aeTimeEvent *shortest = NULL;
    struct timeval tv, *tvp;

    if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
        // poll 需要等待，找到最近的时间事件
        shortest = aeSearchNearestTimer(eventLoop);
    if (shortest) {
        // 有时间事件，计算 poll 的等待时间
        ...
    } else {
        // poll 不需要等待，等待时间设置为 0
        if (flags & AE_DONT_WAIT) {
            tv.tv_sec = tv.tv_usec = 0;
            tvp = &tv;
        } else {
            // poll 需要等待，但是没有时间事件，将阻塞等待
            tvp = NULL; /* wait forever */
        }
    }
    // 传入 poll 等待时间
    numevents = aeApiPoll(eventLoop, tvp);
    // 处理文件事件
    for (j = 0; j < numevents; j++) {
        ...
    }
}
// 处理时间时间
if (flags & AE_TIME_EVENTS)
    processed += processTimeEvents(eventLoop);

```

### 性能问题
问题 1：时间事件采用链式存储，每次查找超时事件都需要从头遍历整个链表，效率较低。
方案：可以使用具有顺序特性的数据结构，如最小堆，查找效率是 O(1)。

问题 2：当网络 IO 数量过多时，用单线程来监听所有的 IO 事件，无法处理及时所有请求，会造成较大的响应延迟。   
方案：可以使用多个多路复用线程一起监听 IO 事件，利用多核CPU的处理能力，提高请求响应的处理速度。    

注：当然，以上的问题都是基于 1.2.0 版本，在最新的版本中应该已经优化处理了。

### 参考文件
```
// version：1.2.0
ae.h
ae.c
ae_epoll.c
config.h
```