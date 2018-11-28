# 配置文件(redis.conf)

## 加载配置文件

```c
/* 实现 in redis.c */
int main(int argc, char **argv);

逻辑伪代码：

initServerConfig()

if argc == 2:
    loadServerConfig(argv[1])
```

所有的配置**默认值**会在initServerConfig()中设置  

如果是以"./redis-server /path/to/redis.conf"方式启动，服务器才会调用loadServerConfig()加载配置文件

## timeout N

```redis
timeout 300
```

在客户端闲置N秒后与它断开连接

### 实现

```c
/* 默认值 in redis.c*/
#define REDIS_MAXIDLETIME (60*5)

struct redisServer{
    //省略其他字段
    int maxidletime;
};

typedef struct redisClient{
    //省略其他字段
    time_t lastinteraction;
} redisClient;
```

```c
/* 实现 in redis.c */
void closeTimedoutClients(void);

核心逻辑伪代码：

now = time()

for c in server.clients:
    if now - c.lastinteraction > server.maxidletime:
        close(c)
```

## save seconds changes

```redis
save 900 1  
save 300 10  
save 60 10000  
```

*seconds*秒后如果至少有*changes*个修改就将数据库保存到磁盘

```c
/* 表示save参数的数据结构 in redis.c */
struct saveparam{
    time_t seconds;
    int changes;
};

struct redisServer{
    //省略其他字段
    struct saveparam *saveparams;  /* 所有的struct saveparam */
    int saveparamslen;  /* svaeparams的长度 */
    long long dirty;  /* 未保存的修改数量 */
    time_t lastsave;  /* 上次保存时间 */
};
```

```c
/* 实现 in redis.c*/
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData);

逻辑伪代码：

if not server.bgsaveinprogress:
    now = time()
    for sp in server.saveparams:
        if (server.dirty >= sp.changes) and (now - server.lastsave > sp.seconds):
            bgsave()
```

## dir path

```c
dir ./
```

存放数据库的路径，只能是文件夹  

注：如果dir配置为默认值"./"，那么实际文件夹会受父进程影响。如果我们通过shell运行redis-server，那么shell的当前工作目录(pwd)就是dump.rdb的存放位置

```c
/* 实现 in redis.c*/
static void loadServerConfig(char *filename);

逻辑伪代码：

if chdir(path) == -1:
    exit(1)
```

相关函数：[chdir(2)](http://man7.org/linux/man-pages/man2/chdir.2.html)

## loglevel level

设置日志级别(verbosity)

* debug：用于开发和调试，日志以字符'.'开头

* notice：生产级别，日志以'-'开头

* warning：只显示重要信息，日志以'*'开头

```c
#define REDIS_DEBUG 0
#define REDIS_NOTECE 1
#define REDIS_WARNING 2

struct redisServer{
    //省略其他字段
    int verbosity;   /* 日志等级 */
};
```

```c
/* 实现 in redis.c */
void redisLog(int level, const char *fmt, ...);

逻辑伪代码：

file = server.logfile if server.logfile else stdout

if level >= server.verbosity:
    head_chars = ".-*"
    write(file, head_chars[level] + logs) #level就是上面三个宏中的一个
```

## logfile file

```redis
logfile stdout
```

日志的文件名，如果设为stdout，日志将被打印到标准输出

```c
struct redisServer{
    //省略其他字段
    char *logfile;
};
```

```c
/* 实现 in redis.c */

void redisLog(int level, const char *fmt, ...);

逻辑伪代码：

file = server.logfile if server.logfile else stdout
```

## databases num

```redis
databases 16
```

数据库的数量

```c
/* 默认值 in redis.c */
#define REDIS_DEFAULT_DBNUM 16

struct redisServer{
    //省略其他字段
    int dbnum;
};
```