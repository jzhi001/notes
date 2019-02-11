# How to get key's value in Redis database
## db.c::lookupKey() family

`lookupKey()` family is API for callers to get key's value in database.

NOTE: all master-slave parts are ignored because, well.. I haven't figure it out yet :(  

### General lookup function

`lookupKey()`, it does one more thing than just get value from dictionary: update cache LFU if caller wants:

* if flag is LOOKUP_NONE, cache LFU is updated (don't get confused with "NONE", treat it as TOUCH!)
* if flag is LOOKUP_NOTOUCH, do nothing. (support TTL, TYPE...)

```C
robj *lookupKey(redisDb *db, robj *key, int flags)

#define LOOKUP_NONE 0 // LOOKUP_TOUCH 
#define LOOKUP_NOTOUCH (1<<0) // do NOT update cache
```

```Python3
def lookupKey(db, key, touch):
    val = db.get(key) # returns db.dict[key]
    if touch:
        val.updateCache()
    return val
```

### lookupKey() for read

#### General lookupKey() for read

`lookupKeyReadWithFlags()` calls `lookupKey()` and does two things more:

1. expire the key if it reaches TTL
2. call `lookupKey()`
3. update server cache states (for INFO command)

```C
robj *lookupKeyReadWithFlags(redisDb *db, robj *key, int flags)
```

```Python3
def lookupKeyReadWithFlags(db, key, flag):
    if expireIfNeeded(db, key):
        return NULL
    else:
        val = lookupKey(db, key, flag)
        if val is NULL:
            server.keyspace_miss += 1
        else:
            server.keyspace_hit += 1
        return val
```

#### Most used lookupKey() for read

The most commonly used function for read operation is `lookupKeyRead()`, which gets key from dict and update cache LFU.

```C
robj *lookupKeyRead(redisDb *db, robj *key)
```

```Python3
def lookupKeyRead(db, key):
    # note LOOKUP_NONE means touch, don't be confused!
    return lookupKeyReadWithFlags(db, key, LOOKUP_NONE)
```

#### lookupKey() for write

`lookupKey()` for **ALL** write operations (both overwrite and replace):

1. expire key if it reaches TTL
2. get key
3. update cache LFU

```C
robj *lookupKeyWrite(redisDb *db, robj *key)
```

```Python3
def lookupKeyWrite(db, key):
    expireIfNeeded(db, key)
    val = lookupKey(db, key, LOOKUP_NONE)
    return val
```

#### Helper functions of lookupKey()

`lookupKey()` for read or reply if key not exist. 

```C
robj *lookupKeyReadOrReply(client *c, robj *key, robj *reply)
```

```Python3
def lookupKeyReadOrReply(client, key, reply):
    val = lookupKeyRead(client.db, key)
    if val is NULL:
        client.addReply(reply)
    return val
```

Check out `STRLEN` command implementation as example:

```Python3
def strlen(client):
    key = client.args[1]
    # reply 0 if key not exist
    val = lookupKeyReadOrReply(client, key, '0')
    if val:
        checkType(client, val, OBJ_STRING)
        addI64toReply(client, val.length)
```

There's also `lookupKeyWriteOrReply()`, it works same as `lookupKeyReadOrReply()` except calls `lookupKeyWrite()`.

```C
robj *lookupKeyWriteOrReply(client *c, robj *key, robj *reply)
```

### Conclusion

1. If you lookup key for write operation, key's cache LFU is **ALWAYS** updated
2. Most key lookups in read operation calls `lookupKeyRead()`, which **UPDATES** cache LFU
3. Some read commands just need to get key and leave its cache LFU untouched, that's why `lookupKeyReadWithFlags()` is open to callers
4. `lookupKey[Read|Write]OrReply()` is like helper function, it replies when key not exist
5. when you call helper function, cache LFU of the key is **ALWAYS** updated 