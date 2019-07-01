# Redis 2.2 config notes

## how to set config

1. redis.conf
2. redis-server - (stdin)
3. config set

## loadServerConfig()

semantic: load config from stdin or config file

## test backtrace()

```c
#if defined(__APPLE__) || defined(__linux__)
#define HAVE_BACKTRACE 1
#endif
```

Use [predefined macros](./clib.md#predefined-OS-macros) to determine whether current OS has proc file system.

Only [proc file system](https://en.wikipedia.org/wiki/Procfs) provides `backtrace(2)`.

Proc file system treats processes as files saved in /proc/pid.