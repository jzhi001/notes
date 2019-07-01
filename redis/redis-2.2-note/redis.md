# Redis 2.2 redis.c notes

main() ??
initServerConfig() ??
commandTableDictType ??
populateCommandTable() ??
lookupCommandByCString() ??
genRedisInfoString() ??
initServer() ??
loadAppendOnlyFile() ??
loadRdbFile() ??
beforeSleep() ??
redisPanic() ??

## main()

entrance of redis-server

redis-server [/path/to/redis.conf]
redis-server - (read config from stdin)
redis-server [-v] [--version] [--help]

semantic: load config, aof, rdb files if provided;
register beforesleep hook (handle client query before doing file IO)
start main event loop

## createPidFile()

semantic: write pid of current process to `server.pidfile`

## daemonize()

semantic: run Redis as daemon

fork()

exit() if parent

if child: create new session and redirect input source and output to /dev/null

## initServerConfig()

semantic:

set config related fields to default value
create command table (server.commands)

[updateLRUClock()](#updateLRUClock())
[resetServerSaveParams()](#resetServerSaveParams())
[appendServerSaveParams()](#appendServerSaveParmas())

## usage()

semantic: show help message and exit

## updateLRUClock()

set time part (least sig 21 bits) of [server.lruclock](./DS.md#lruclock) to now

## resetServerSaveParams()

set `server.saveparams` to NULL and `server.saveparamlen` to 0

## appendServerSaveParams

add saveparam to server.saveparams

## initServer()

unfinished

```c
signal(SIGHUP, SIG_IGN);
signal(SIGPIPE, SIG_IGN);
```

semantic: ignore SIGHUP and SIGPIPE signals

[setupSignalHandlers()](#setupSignalHandlers())

## setupSignalHandlers()

semantic: 

Add `SA_NODEFER`, `SA_ONSTACK`, `SA_RESETHAND` to sa_flags.

If not on [proc file system](./clib.md#OS-predefined-macros) use [sigtermHandler](#sigtermHandler()) as handler for `SIGTERM`.  

Else use [sigsegvHandler](./sigsegvHandler()) for `SIGTERM`, `SIGBUS`, `SIGSEGV`, `SIGFPE`, `SIGILL`.

## sigtermHandler()

log "received SIGTERM"
server.shutdown_asap = 1

## sigsegvHandler()

semantic:

log signal value
log [redis info](#genRedisInfoString())
log [IP register](#getMcontextEip())
log backtrace messages
remove pid file

use default handler to handle the signal

## genRedisInfoString()

unfinished

## getMcontextEip()

semantic: get value of IP register

structure of `ucontext_t` varies between platforms

```c
return (void*) uc->uc_mcontext.gregs[16];
```

mcontext stands for machine context, gregs stands for general registers. Index 16 is IP register.

It is clear from Redis 5.0.4 debug.c

```c
#elif defined(__X86_64__) || defined(__x86_64__)
    /* Linux AMD64 */
    serverLog(LL_WARNING,
    "\n"
    "RAX:%016lx RBX:%016lx\nRCX:%016lx RDX:%016lx\n"
    "RDI:%016lx RSI:%016lx\nRBP:%016lx RSP:%016lx\n"
    "R8 :%016lx R9 :%016lx\nR10:%016lx R11:%016lx\n"
    "R12:%016lx R13:%016lx\nR14:%016lx R15:%016lx\n"
    "RIP:%016lx EFL:%016lx\nCSGSFS:%016lx",
        (unsigned long) uc->uc_mcontext.gregs[13],
        (unsigned long) uc->uc_mcontext.gregs[11],
        (unsigned long) uc->uc_mcontext.gregs[14],
        (unsigned long) uc->uc_mcontext.gregs[12],
        (unsigned long) uc->uc_mcontext.gregs[8],
        (unsigned long) uc->uc_mcontext.gregs[9],
        (unsigned long) uc->uc_mcontext.gregs[10],
        (unsigned long) uc->uc_mcontext.gregs[15],
        (unsigned long) uc->uc_mcontext.gregs[0],
        (unsigned long) uc->uc_mcontext.gregs[1],
        (unsigned long) uc->uc_mcontext.gregs[2],
        (unsigned long) uc->uc_mcontext.gregs[3],
        (unsigned long) uc->uc_mcontext.gregs[4],
        (unsigned long) uc->uc_mcontext.gregs[5],
        (unsigned long) uc->uc_mcontext.gregs[6],
        (unsigned long) uc->uc_mcontext.gregs[7],
        (unsigned long) uc->uc_mcontext.gregs[16],
        (unsigned long) uc->uc_mcontext.gregs[17],
        (unsigned long) uc->uc_mcontext.gregs[18]
    );
```