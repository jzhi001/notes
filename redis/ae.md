# ae

## unsolved

为什么fd不能大于setsize  

AE_DONT_WAIT 是处理时间事件的方式，指不等到指定时间直接处理

AE_NONE表示fileEvent的null  

beforeSleep？？ afterSleep？？

void aeEventFinalizerProc(strcut aeEventLoop *eventLoop, void *clientData) 干嘛用的？？ clientData的后处理？

它用在哪？？

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

事件循环是一个数据结构，它保存了用户注册的事件  

事件循环是循环处理的，每次循环做两个操作

1. 查找并处理到时间的time event

2. 查找并处理可以进行I/O操作的file event

## Event Loop 实现

**以下代码都不是源码，它们的作用是帮助你从简单到复杂地理解源码**

上面说过事件循环是一个数据结构，我们先声明它。后面我们会逐渐完成它的定义

```c
struct eventLoop;
```

### time event

要描述一个时间事件，我们需要多个字段

* 唯一标识

* 何时执行

* 如何执行

* 存放数据的内存 clientData??

* 处理事件后如何处理数据 finalizerProc ??

eventLoop要怎么保存多个时间事件呢？

事件的数量是未知的，所以用链表好一些

在某个事件完成后我们要删除它，此时我们有该事件的指针。为了简化删除操作我们让链表变为双向的(否则要从头遍历找到该节点的前一个节点)

```c
struct eventLoop;

strcut timeEvent{
    long id;
    long when_sec;
    long when_ms;
    void *data;
    void (*timeProc)(void *data);
    void (*finalizer)(void *data);
    struct timeEvent *next, *prev;
} timeEvent;
```
