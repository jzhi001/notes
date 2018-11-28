# redis.c

## 参数

### 状态码

```c
#define REDIS_OK 0
#define REDIS_ERR -1
```

### 静态配置参数

```C
#define REDIS_SERVERPORT 6379
#define REDIS_QUERYBUF_LEN 1024 /* 用户单行命令的最大长度 */
#define REDIS_LOADBUF_LEN 1024 /* ?? */
#define REDIS_MAX_ARGS          16 /* ?? */
#define REDIS_DEFAULT_DBNUM     16 /* 对应redis.conf中的dbnum */
#define REDIS_CONFIGLINE_MAX    1024 /* 配置文件每行字符的最大值 */
```

### hash table 参数

```c
#define REDIS_HT_MINFILL        10      /* 哈希表最少使用 10% */
#define REDIS_HT_MINSLOTS       16384   /* 哈希表slot的最小值 */
```

### 命令类型

```c
#define REDIS_CMD_BULK          1
#define REDIS_CMD_INLINE        0
```

### List相关参数

```c
#define REDIS_HEAD 0
#define REDIS_TAIL 1
```

## 数据结构

### 客户端

```c
typedef struct redisClient {
    int fd;
    dict *dict;  /* ?? 客户端有数据库 key => Redis Object */
    sds querybuf;  /* 存放命令的buffer */
    sds argv[REDIS_MAX_ARGS]; /* 存放命令(如set, foo, bar) */
    int argc;
    int bulklen;    /* bulk read len. -1 if not in bulk read mode */
    list *reply;  /* 服务器返回的robj，Redis Object 链表 */
    int sentlen;  /* 因为是异步写入，所以要记录已写入长度 */
    time_t lastinteraction; /* 上次互动时间(server向这个client读写数据)，用于timeout */
} redisClient;
```

### 服务器

```c
struct redisServer {
    int port;
    int fd;
    dict **dict;    /* dbnums个数据库 */
    long long dirty;            /* 上次save后新的修改数量 */
    list *clients;
    char neterr[ANET_ERR_LEN];  /* ?? */
    aeEventLoop *el;
    int verbosity;  /* 对应redis.conf中的loglevel */
    int cronloops;  /* Redis eventLoop 循环次数 */
    int maxidletime;  /* 最大闲置时间 */
    int dbnum;
    list *objfreelist;  /* robj链表，里面都是用过的robj，通过复用避免内存分配 */
    int bgsaveinprogress;  /* bgsave 进度 */
    time_t lastsave;  /* 上一次save的时间 */
    struct saveparam *saveparams;
    int saveparamslen;
    char *logfile;  /* 对应redis.conf中的logfile */
};
```

### Redis 命令

```c
typedef void redisCommandProc(redisClient *c);
struct redisCommand {
    char *name;
    redisCommandProc *proc;
    int arity;  /* 参数数量(包括命令) */
    int type;  /* bulk 或者 inline */
};
```

### 共享对象

```c
struct sharedObjectsStruct {
    robj *crlf, *ok, *err, *zerobulk, *nil, *zero, *one, *pong;
} shared;
```

## 全局变量

```c
static struct redisServer server; /* 服务器 */
```

```c
/* 命令 */
static struct redisCommand cmdTable[] = {
    {"get",getCommand,2,REDIS_CMD_INLINE},
    {"set",setCommand,3,REDIS_CMD_BULK},
    {"setnx",setnxCommand,3,REDIS_CMD_BULK},
    {"del",delCommand,2,REDIS_CMD_INLINE},
    {"exists",existsCommand,2,REDIS_CMD_INLINE},
    {"incr",incrCommand,2,REDIS_CMD_INLINE},
    {"decr",decrCommand,2,REDIS_CMD_INLINE},
    {"rpush",rpushCommand,3,REDIS_CMD_BULK},
    {"lpush",lpushCommand,3,REDIS_CMD_BULK},
    {"rpop",rpopCommand,2,REDIS_CMD_INLINE},
    {"lpop",lpopCommand,2,REDIS_CMD_INLINE},
    {"llen",llenCommand,2,REDIS_CMD_INLINE},
    {"lindex",lindexCommand,3,REDIS_CMD_INLINE},
    {"lrange",lrangeCommand,4,REDIS_CMD_INLINE},
    {"ltrim",ltrimCommand,4,REDIS_CMD_INLINE},
    {"randomkey",randomkeyCommand,1,REDIS_CMD_INLINE},
    {"select",selectCommand,2,REDIS_CMD_INLINE},
    {"move",moveCommand,3,REDIS_CMD_INLINE},
    {"rename",renameCommand,3,REDIS_CMD_INLINE},
    {"renamenx",renamenxCommand,3,REDIS_CMD_INLINE},
    {"keys",keysCommand,2,REDIS_CMD_INLINE},
    {"dbsize",dbsizeCommand,1,REDIS_CMD_INLINE},
    {"ping",pingCommand,1,REDIS_CMD_INLINE},
    {"echo",echoCommand,2,REDIS_CMD_BULK},
    {"save",saveCommand,1,REDIS_CMD_INLINE},
    {"bgsave",bgsaveCommand,1,REDIS_CMD_INLINE},
    {"shutdown",shutdownCommand,1,REDIS_CMD_INLINE},
    {"lastsave",lastsaveCommand,1,REDIS_CMD_INLINE},
    /* lpop, rpop, lindex, llen */
    /* dirty, lastsave, info */
    {"",NULL,0,0}
};
```

### 新的哈希表类型

sds => Redis Object  
用于Redis数据库

### serverCron 时间事件处理函数(1s)

Redis server的事件循环处理函数

* 如果db的使用率小于REDIS_HT_MINFILL就缩小db

* 每循环5次记录连接的客户端数量

* 每循环10次关闭闲置客户端

* 检查bgsave进度(文件名固定为dump.rdb)
  * 如果正在bgsave，等待其完成后修改server.dirty = 0, server.lastsave = now, server.gbsaveinprogress = 0 (wait阻塞？？)
  * 如果没在bgsave，遍历server.saveparams判断是否需要bgsave (server.dirty >= sp->changes && now-server.lastsave > sp->seconds) server.saveparametes表示redis.conf中的所有save配置

## save配置

添加save配置

```c
static void appendServerSaveParams(time_t seconds, int changes);
```

重置save配置

```c
static void ResetServerSaveParams();
```

## 服务器初始化

初始化配置

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

服务器初始化

?? signal()

给事件循环注册了时间事件

```c
aeCreateTimeEvent(server.el, 1000, serverCron, NULL, NULL);
```

## 加载配置文件

```c
static void loadServerConfig(char *filename);
```

REDIS_CONFIGLINE_MAX指配置文件每行字符的最大值

使用chdir调到dir配置的文件夹

## 服务器回应 文件事件处理函数

```c
static void sendReplyToClient(aeEventLoop *el, int fd, void *privData, int mask);
```

这里privData是redisClient *

c->reply可能包含多个robj，每个robj都是异步写入

## 查找命令

```c
static strcut redisCommand *lookupCommand(char *name);
```

遍历cmdTable

## 重置客户端 ？？

让客户端准备处理下一个命令

```c
static void resetClient(redisClient *c) {
    freeClientArgv(c);
    c->bulklen = -1;
}
```

## 处理命令

如果该函数被调用，说明client已经读取了完整的命令并存在argv/argc中  
该函数可能做两件事情

* 执行命令

* 让server留出bulk去读取client

返回值：

* 0：client已经被关闭

* 1：还可以和client互动

如命令： set foo bar

c->argc = 3

c->argv = {"set", "foo", "bar"}

## 读取客户端的命令  文件事件处理函数

```c
static void readQueryFromClient(aeEventLoop *el, int fd, void *privData, int mask);
```

这里privData是RedisClient *

命令读到c->querybuf中(因为是异步读，需要多次read，多以需要buffer)

## 选择数据库

```c
static in selectDb(redisClient *c, int id);
```

## 创建客户端

```c
static int createClient(int fd);
```

## 添加回复  注册文件事件

```c
static void addReply(redisClient *c, robj *obj);
```

其中创建了文件事件，处理函数是sendReplyToClient

```c
aeCreateFileEvent(server.el, c->fd, AE_WRITABLE,
        sendReplyToClient, c, NULL)
```

## 连接客户端  文件处理函数

```c
cfd = anetAccept

createClient(cfd);
```

## 数据库存储

```c
static int saveDb(char *filename)
```

?? network byte order  htonl()

创建临时文件，文件名是 temp-时间.随机数.rdb

写rdb文件头： "REDIS0000"

遍历所有数据库，如果db[i]被使用过：

* 写select db的操作码REDIS_SELECTDB

* 写db的索引i

对于db[i]中每一个 key => robj:

写key的 长度 和 值

对于robj：

* 如果是string类型

  * 写长度

  * 写robj->ptr的值(字符串格式)

* 如果是list类型

  * 写list的元素个数

  * 遍历list，写每个元素(sds)的 长度 和 值

写REDIS_EOF

关闭文件

使用rename(2)将临时文件移动(可能重命名)到最终目的地(可能会替换旧文件)

server.dirty = 0

server.lastsave = time(NULL)

return REDIS_OK

```c
static int saveDbBackground(char *filename)
```

创建子进程并执行saveDb()

主进程不等子进程执行完，而是通过server.bgsaveinprogress确认是否完成

## 加在rdb

```c
static in loadDb(char *filename)
```

saveDb反过来

## 命令