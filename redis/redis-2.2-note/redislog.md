# Redis 2.2 redislog notes

## related config

loglevel debug/verbose/notice/warning
logfile stdout
syslog-enabled no/yes
syslog-ident redis
syslog-facility local0 - local7

## related fields

server.logfile
server.syslog_enabled
server.syslog_ident
server.syslog_facility

## redisLog()

semantic: log into log file and syslog

log format: [pid] day Month hour:min:sec [char] message

```c
void redisLog(int level, const char *fmt, ...)
```

There are 4 log levels

```c
#define REDIS_DEBUG 0
#define REDIS_VERBOSE 1
#define REDIS_NOTICE 2
#define REDIS_WARNING 3
```

implementation details

```c
// in redisLog()
const int syslogLevelMap[] = { LOG_DEBUG, LOG_INFO, LOG_NOTICE, LOG_WARNING };
const char *c = ".-*#";

if (server.syslog_enabled) syslog(syslogLevelMap[level], "%s", msg);
```

## syslog

`openlog()` is called in `initServer()`



