# data structures in Redis 2.2

## config files

vm_swap_file: /tmp/redis-%p.vm
pid file: /var/run/redis.pid
db file: dump.rdb
appendfilename: appendonly.aof

## redisClient

status: flags
args: querybuf, argc, argv, reqtype, bulklen, multibulklen

### args input

because of event loop, client reads part of commands at a time, querybuf is used to store partial command

when command reading is complete, client will turn string args to robjs, which is argv and argc

reqtype is used to differentiate bulk request or inline request

### reqtype

```c
/* Client request types */
#define REDIS_REQ_INLINE 1
#define REDIS_REQ_MULTIBULK 
```

### flags

```c
/* Client flags */
#define REDIS_SLAVE 1       /* This client is a slave server */
#define REDIS_MASTER 2      /* This client is a master server */
#define REDIS_MONITOR 4     /* This client is a slave monitor, see MONITOR */
#define REDIS_MULTI 8       /* This client is in a MULTI context */
#define REDIS_BLOCKED 16    /* The client is waiting in a blocking operation */
#define REDIS_IO_WAIT 32    /* The client is waiting for Virtual Memory I/O */
#define REDIS_DIRTY_CAS 64  /* Watched keys modified. EXEC will fail. */
#define REDIS_CLOSE_AFTER_REPLY 128 /* Close after writing entire reply. */
#define REDIS_UNBLOCKED 256 /* This client was unblocked and is stored in
      
```

```c
int fd;
    redisDb *db;
    int dictid;
    sds querybuf;
    int argc;
    robj **argv;
    struct redisCommand *cmd;
    int reqtype;
    int multibulklen;       /* number of multi bulk arguments left to read */
    long bulklen;           /* length of bulk argument in multi bulk request */
    list *reply;
    int sentlen;
    time_t lastinteraction; /* time of the last interaction, used for timeout */
    int flags;              /* REDIS_SLAVE | REDIS_MONITOR | REDIS_MULTI ... */
    int slaveseldb;         /* slave selected db, if this client is a slave */
    int authenticated;      /* when requirepass is non-NULL */
    int replstate;          /* replication state if this is a slave */
    int repldbfd;           /* replication DB file descriptor */
    long repldboff;         /* replication DB file offset */
    off_t repldbsize;       /* replication DB file size */
    multiState mstate;      /* MULTI/EXEC state */
    blockingState bpop;   /* blocking state */
    list *io_keys;          /* Keys this client is waiting to be loaded from the
                             * swap file in order to continue. */
    list *watched_keys;     /* Keys WATCHED for MULTI/EXEC CAS */
    dict *pubsub_channels;  /* channels a client is interested in (SUBSCRIBE) */
    list *pubsub_patterns;  /* patterns a client is interested in (SUBSCRIBE) */

    /* Response buffer */
    int bufpos;
    char buf[REDIS_REPLY_CHUNK_BYTES];
```

## redisServer

| Module | Fields |
| :---   | :---   |
| net | port, bindaddr, unixsocket, sofd, neterr |
| daemonize | daemonize |
| save | [saveparams](#saveparams), [saveparamslen](#saveparamslen) |
| loading | loading, loading_start_time, loading_total_bytes |

## net

bind: bindaddr is ip

unixsocket command: sofd if fd of the socket file

neterr is char[] for error message

### saveparams

array of `struct saveparam`, tell Redis when to save db.
Save when [seconds] time passed *AND* [changes] changes were made

```c
struct saveparam{
	time_t seconds;
	int changes;
}
```

### saveparamslen

len of saveparams

### lruclock

least significant 21 bits is time() devided by resolution. 
default resolution is 10.
If using default resolution, time will overflow after 240 days.

### shutdown_asap

mentioned in [sigtermHandler](./redis.md#sigtermHandler)

### port

default: REDIS_SERVERPORT (6379)

### bindaddr

default: NULL

### unixsocket

default: NULL

### ipfd

default: -1

### sofd

default: -1

### dbnum

default: REDIS_DEFAULT_DBNUM (16)

### verbosity

default: REDIS_VERBOSE

REDIS_DEBUG
REDIS_NOTICE
REDIS_WARNING

### maxidletime

default: REDIS_MAXIDLETIME (60 * 5)

### loading

default: 0

### logfile

default: NULL (stdout)

### syslog_enabled

default: 0

### syslog_ident

default: "redis"

### syslog_facility

default: LOG_LOCAL0

### daemonize

default: 0

### appendonly

default: 0

### appendfsync

default: APPENDFSYNC_EVERYSEC

```c
/* Append only defines */
#define APPENDFSYNC_NO 0
#define APPENDFSYNC_ALWAYS 1
#define APPENDFSYNC_EVERYSEC 
```

### no_appendfsync_on_rewrite

default: 0