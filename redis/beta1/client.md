# Client

## 数据结构

```c
typedef struct redisClient {
    int fd;
    dict *dict;
    sds querybuf;  /* sds, 存放客户端的输入 */
    sds argv[REDIS_MAX_ARGS];  /* querybuf.split() */
    int argc;
    int bulklen;    /* bulk长度，-1表示inline */
    list *reply;  /* robj list */
    int sentlen;  /* reply中当前robj的已写入长度 */
    time_t lastinteraction; /* timeout in redis.conf */
} redisClient;
```

## 创建客户端

```c
//in redis.c
static int createClient(int fd) {
    redisClient *c = malloc(sizeof(*c));

    anetNonBlock(NULL,fd);
    anetTcpNoDelay(NULL,fd);
    if (!c) return REDIS_ERR;
    selectDb(c,0);
    c->fd = fd;
    c->querybuf = sdsempty();
    c->argc = 0;
    c->bulklen = -1;
    c->sentlen = 0;
    c->lastinteraction = time(NULL);
    if ((c->reply = listCreate()) == NULL) oom("listCreate");
    listSetFreeMethod(c->reply,decrRefCount);
    if (aeCreateFileEvent(server.el, c->fd, AE_READABLE,
        readQueryFromClient, c, NULL) == AE_ERR) {
        freeClient(c);
        return REDIS_ERR;
    }
    if (!listAddNodeTail(server.clients,c)) oom("listAddNodeTail");
    return REDIS_OK;
}
```

## 读取客户端输入(file event)

创建client时注册了文件事件readQueryFromClient:

```c
//in redis.c
static void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask);
```

异步读入客户端输入，输入完整时就执行命令

```python3
逻辑伪代码：

#第一部分：获取用户输入并存到querybuf中
buf = []
nread = read(fd, buf)

if nread == -1:
    if errno == EAGAIN:
        #non-block I/O正常但是此时没有输入
        nread = 0
    else:
        #read异常
        return
elif nread == 0:
    #client closed
    return

#如果有新的输入就进入第二部分
if not nread:
    return

c.querybuf += buf[:nread]
c.lastinteraction = time()

#第二部分：尝试执行命令
def handleCmd():
    if c.bulklen == -1:
        #inline mode
        try:
            cmd_tail = c.querybuf.index('\n')
            cmd = c.querybuf[:cmd_tail+1]
            c.argv = cmd.split('')
            c.querybuf = c.querybuf[cmd_tail+1:]
            if processCommand(c) and len(c.querybuf):
                handleCmd()
        except ValueError:
            #命令不完整
            return
    else:
        #bulk mode
        if c.bulklen > len(c.querybuf):
            #不完整
            return
        #忽略"\r\n"
        c.argv.append(c.querybuf[:c.bulklen-1])
        c.argc += 1
        c.querybuf = c.querybuf[c.bulklen:]
        processCommand(c)
        return

    handleCmd()
```

## 执行命令

```c
//in redis.c
static int processCommand(redisClient *c);
```

所有命令的参数存放在`redisClient::argv`中

返回值：

0 -> 客户端断开连接

1 -> 客户端保持连接

```python3
逻辑伪代码:

cmd = lower(c.argv[0])

if cmd == 'quit':
    freeClient(c)
    return 0

cmd = lookupCommand(cmd);

#此时cmd的类型是struct redisCommand
#省略各种检查

#如果是bulk类型
if cmd.type == REDIS_CMD_BULK and c.bulklen == -1:
    n = c.argv[c.argc-1]
    c.argv[c.argc-1] = NULL
    c.argc -= 1
    c.bulklen += (n + 2)  #加上CRLF

    if len(c.querybuf) >= c.bulklen:
        #bulkbuf中已经有完整的bulk
        c.argv[c.argc] = c.querybuf[:c.bulklen-1]
        c.argc += 1
        c.querybuf = c.querybuf[c.bulklen:]
    else:
        #不完整
        return 1
cmd.proc(c)
resetClient(c)
return 1
```

## 将结果写入客户端(file event)

所有的`redisCommand->proc`中都调用`addReply`将结果写入客户端

```c
static void addReply(redisClient *c, robj *obj);

逻辑伪代码：

createFileEvent(sendReplyToClient)
c.reply.append(robj)
robj.addReference()
```

其中注册了文件事件，用于将回复写入客户端

```c
static void sendReplyToClient(aeEventLoop *el, int fd, void *privdata, int mask);
```

异步写入`c->reply`中所有`robj`的值。使用`c->sentlen`记录当前`robj`的已写入长度，当前`robj`写完后从`c->reply`删除。全部完成后从事件循环中删除该事件