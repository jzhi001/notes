# Redis commands

## Redis命令数据结构

```c
typedef void redisCommandProc(redisClient *c);
struct redisCommand {
    char *name;    /* 命令名，如GET、SET */
    redisCommandProc *proc;
    int arity;   /* 命令需要的参数数量(包括命令名) */
    int type;   /* inline/bulk */
};
```

如果不理解type字段请看Redis协议中的[inline commands](./protocol.md#INLINE-commands)和[bulk commands](./protocol.md#BULK-commands)

所有的命令都存在全局变量cmdTable中

```c
static struct redisCommand cmdTable[] = {
    //这里省略了其他命令，全部列出来太长了
    {"get",getCommand,2,REDIS_CMD_INLINE},
    {"set",setCommand,3,REDIS_CMD_BULK},
    {"",NULL,0,0}
};
```

## Redis server如何处理命令

TODO

```c
static int processCommand(redisClient *c)
```

```c
static void addReply(redisClient *c, robj *obj);
```

## 所有命令

TODO