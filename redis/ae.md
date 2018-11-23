# ae

## unsolved

为什么fd不能大于setsize  

AE_DONT_WAIT 是处理时间事件的方式，指不等到指定时间直接处理

AE_NONE表示fileEvent的null  

beforeSleep？？ afterSleep？？

void aeEventFinalizerProc(strcut aeEventLoop *eventLoop, void *clientData) 干嘛用的？？ clientData的后处理？

它用在哪？？

timeProc例子：

```c
//in module.c
aeCreateTimeEvent(server.el,period,moduleTimerHandler,NULL,NULL)

int moduleTimerHandler(struct aeEventLoop *eventLoop, long long id, void *clientData)

//in server.c
aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL)

int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData)

//in redis-benchmark.c
aeCreateTimeEvent(config.el,1,showThroughput,NULL,NULL)

int showThroughput(struct aeEventLoop *eventLoop, long long id, void *clientData)
```

所有的timeProc函数中都没有用到函数中的三个参数!

所有的aeCreatTimeEvent调用中clientData和finalizerProc都是NULL!!

如：

注册时间事件a，它会在10分钟后被处理。如果此时使用AE_DONT_WAIT的flag调用aeEventProcess，事件a会被立刻处理

eventLoop中每一个文件对应一个fileEvent，一个fired，一个epoll_event

## Event Loop

并行的一种实现方式，在说事件循环前我们先说说并行  

### 为什么需要并行？

有两种操作(下面会讲)会阻塞当前线程，我们把这种操作称为event(事件)  

当线程被阻塞的时候，我们就不能相应用户的操作

#### 1. I/O操作会阻塞当前线程

#### 2. 定时操作会阻塞当前线程

### 多线程

创建一个新的线程并将事件交给它处理

新线程和当前线程是独立的，有自己的栈

OS通过分配时间片的方式在两个线程间来回切换，并执行一小段代码

因为线程切换，一个线程的阻塞不会影响另一个线程的运行

### 事件循环

事件循环使用一个数据结构保存用户注册的事件  

它的运行方式是死循环(除非有停止命令)，每次循环做两个操作

1. 查找并处理到时间的time event

2. 查找并处理可以进行I/O操作的file event

## Event Loop 实现

**以下代码都不是源码，它们的作用是帮助你从简单到复杂地理解源码**

上面说过事件循环是一个数据结构。这里我们先声明它，后面我们会逐渐完成它的定义

```c
struct eventLoop;
```

### time event

### time event 类型

要定义timeEvent，我们先想想它需要那些字段：

* 什么时候处理该事件 -> 时间戳

* 如何处理该事件 -> 函数指针

* struct eventLoop中保存了多个timeEvent，我们要区分每个事件 -> id

struct eventLoop应该用哪种容器保存多个时间事件呢？

* timeEvent的数量是不固定的

* 注册事件或删除事件需要进行增删操作

根据上面的特点，Redis使用了链表。不过我个人认为优先级队列好一些  
使用链表不需要频繁调整容器的内存长度，而且链表的增删都需要O(1)时间  
Redis中新的time event被加在链表的头部  
为了简化删除操作，Redis使用了双向链表。这样就避免了从头部遍历找前一个节点

目前为止我们的设计：

```c
typedef void timeProc(void);

struct timeEvent{
    long id;
    long when_sec;             /* 何时执行 */
    long when_ms;
    timeProc *proc;    /* 处理函数 */
    struct timeEvent *prev;
    struct timeEvent *next;
} timeEvent;
```

```c
struct eventLoop{
    long timeEventNextId;              /* 用于给timeEvent分配id */
    struct timeEvent *timeEventHead;  /* timeEvent链表 */
} eventLoop;
```

#### 创建&注册time event

timeEvent和struct eventLoop都定义好了，现在把它们连起来：完成创建timeEvent并注册到struct eventLoop的函数

```c
/* 将当前时间 + milliseconds毫秒的结果存在sec和ms指向的内存中 */
static void addMsToNow(long long milliseconds, long *sec, long *ms);

/* 创建timeEvent并将它放在struct eventLoop链表头部 */
long createTimeEvent(struct eventLoop *el, long long ms, timeProc *proc){
    id = el->timeEventNextId++;
    timeEvent *te;

    te = malloc(sizeof(*te));
    if(te == NULL) return -1;
    te->id = id;
    te->proc = proc;
    addMsToNow(ms, te->when_sec, te->when_ms);
    te->prev = NULL;
    te->next = el->timeEventHead;
    if(te->next)
        te->next->prev = te;
    el->timeEventHead = te;
    return id;
}
```

#### 处理time event

处理函数的简单的实现

```c
/* 将当前时间(秒和微秒)分别存在seconds和milliseconds指向的内存中 */
static void getTime(long *seconds, long *milliseconds);
/* 将当前节点从链表中删除 */
static void delNode(timeEvent *te);
/* 删除time event */
static void delTimeEvent(timeEvent *te);

void processTimeEvents(struct eventLoop *el){
    struct timeEvent *te = el->timeEventHead;
    while(te){
        long now_sec, now_ms;
        getTime(&now_sec, &now_ms);
        if(now_sec > te_when_sec ||
            (now_sec == te->when_sec && now_ms >= te->when_ms))
        {
            if(te->proc) te->proc();
        }
        delNode(te);
        delTimeEvent(te);
        te = te->next;
    }
}
```

这个实现非常简单：遍历time event链表，如果当前事件到时间了就处理，处理完成后将它从链表中删除。这种实现有一下几个问题 

第一个问题是处理定期事件时不太效率：如果用户想要注册定期执行的time event，它就只能在timeProc函数中将当前事件再注册一遍。这样就浪费了很多时间在分配和释放内存上

这个问题的解决方法当然是复用当前的time event。Redis根据timeProc函数的返回值判断该事件是否为定期事件：

* 返回值 == AE_NOMORE -> 停止定期事件并删除

* 其他返回值 -> [返回值]毫秒后再次执行该事件

Redis中并不直接删除time event，而是将它的id修改为AE_DELETED_EVENT_ID，在下一次循环中删除（为什么？？）

根据问题修改代码

```c
#define AE_NOMORE -1;  //不再定期执行
#define AE_DELETED_EVENT_ID -1;  //删除该事件
typedef int timeProc(void); //将返回值变成int
```

```c
void timeEventProc(eventLoop *el){
    struct timeEvent *te = el->timeEventHead;
    while(te){
        if(te == AE_DELETED_EVENT_ID){
            struct timeEvent *next = te->next;
            delNode(te);
            delTimeEvent(te);
            te = next;
            continue;
        }

        long now_sec, now_ms;
        getTime(&now_sec, &now_ms);
        if(now_sec > te->when_sec || (now_sec == te->when_sec && now_ms > te->when_ms) ){
            int retval = te->proc();
            if(retval = AE_NOMORE){
                addMsToNow(retval, &te->when_sec, &te->when_ms);
            }else{
                te->id = AE_DELETED_EVENT_ID;
            }
        }
        te = te->next;
    }
}
```

为什么不处理time event中注册的time event??

clock skew??

### file event
