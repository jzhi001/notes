# Redis 2.2 AOF notes

loadAppendOnlyFile() ??
createFakeClient() 
    fakeclient: why these fields ??
    aeProcessEvents() ??
    why redisAssert() ??
    why free fakeClient's argv

## alias in config.h

redis_fstat is alias of fstat

redis_stat is alias of stat

## loadAppendOnlyFile()

semantic: create a fake client for executing aof commands

fields involved:

```c
selectDb(c,0);
c->fd = -1;
c->querybuf = sdsempty();
c->argc = 0;
c->argv = NULL;
c->bufpos = 0;
c->flags = 0;
/* We set the fake client as a slave waiting for the synchronization
    * so that Redis will not try to send replies to this client. */
c->replstate = REDIS_REPL_WAIT_BGSAVE_START;
c->reply = listCreate();
c->watched_keys = listCreate();
listSetFreeMethod(c->reply,decrRefCount);
listSetDupMethod(c->reply,dupClientReplyValue);
initClientMultiState(c);
```

use infinite loop to read and exec commands

time allocated in event loop: 1000 loops