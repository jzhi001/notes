# Redis Server

## server 数据结构

```c
struct redisServer {
    int port;
    int fd;

    //数据库
    dict **dict;
    int dbnum;  /* databases in redis.conf */

    list *clients;
    char neterr[ANET_ERR_LEN]; /* 存放anet相关的错误信息 */

    //事件循环
    aeEventLoop *el;
    int cronloops;  /* 主事件循环次数 */
    int maxidletime;  /* timeout in redis.conf */
    list *objfreelist;          /* robj垃圾堆 */

    //本地存储(save/bgsave)
    int bgsaveinprogress;
    long long dirty;  /* 上次save后有多少未保存的修改 */
    time_t lastsave;
    struct saveparam *saveparams;
    int saveparamslen;

    //日志
    char *logfile;
    int verbosity;  /* loglevel in redis.conf */
};
```

其中标记`xxx in redis.conf`的字段是用于[实现redis配置](./redis-conf.md)的  

服务器实例是redis.c中的全局变量`server`

```c
static struct redisServer server;
```

## server运行流程

* [初始化配置相关字段](#initServerConfig)

* 初始化其他字段

* 注册主循环事件serverCron

* 加载配置文件

* 加载dump.rdb

* 注册文件事件acceptHandler(接受并创建client)

* 开始事件循环

## initServerConfig

在[redis.conf](./redis-conf.md)中都见过了

```c
static void initServerConfig() {
    server.dbnum = REDIS_DEFAULT_DBNUM;
    server.port = REDIS_SERVERPORT;
    server.verbosity = REDIS_DEBUG;
    server.maxidletime = REDIS_MAXIDLETIME;
    server.saveparams = NULL;
    server.logfile = NULL; /* NULL = log on standard output */
    ResetServerSaveParams();

    appendServerSaveParams(60*60,1);  /* save after 1 hour and 1 change */
    appendServerSaveParams(300,100);  /* save after 5 minutes and 100 changes */
    appendServerSaveParams(60,10000); /* save after 1 minute and 10000 changes */
}
```

## initServer

TODO

### serverCron

主事件在`initServer`中注册：

```c
aeCreateTimeEvent(server.el, 1000, serverCron, NULL, NULL);
```

* 调整所有db大小

* 每10次循环关闭闲置客户端`closeTimedoutClients`

* bgsave

  * 如果正在bgsave：等待子进程完成bgsave

  * 如果没在bgsave：[检查是否需要save](./redis-conf.md#save-seconds-changes)，如果需要就调用`saveDbBackground`

* 返回1000 -> 1秒后继续处理该事件

## acceptHandler

注意：只要不删除file event它就一直会被处理

* `anetAccept`接受客户端连接

* `createClient(int fd)`给客户端创建`redisClient`数据结构