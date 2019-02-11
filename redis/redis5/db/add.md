# How to add key-value to Redis database

## Add only if key is not exist

This function makes sure key is new to database.
Since database doesn't hold the key, the function does not need to deal with expiration.

Calls [dictAdd](../dict/add.md#dictadd).

```C
void dbAdd(redisDb *db, robj *key, robj *val)
```

```Python3
def dbAdd(db, key, val):
    if key in db:
        raise Exception('key already exists')
    else:
        db[key] = val
        # notify blocked clients which are waiting for this key
        if (key is Set) or (key is List):
            signalKeyAsReady(db, key)
```

## Overwrite operation

Implementation of [overwrite operation](./writeOps.md#overwrite).   
Note: the key must exists in database.

```C
void dbOverwrite(redisDb *db, robj *key, robj *val)
``` 

```Python3
def dbOverwrite(db, key, val):
    old = db[key]
    if old:
        old.ptr = val.ptr
        val.cache_stat = old.cache_stat
    else:
        raise Exception('key is not exist')
```

## General set function

Implementation of [replace operation](./writeOps.md#replace).  
   
Although it is documented as "High level Set operation" and 
"All the new keys in the database should be created via this interface", 
it still cannot replace `dbAdd()` and `dbOverwrite()` because this function 
force database to remove the expire.  

```C
void setKey(redisDb *db, robj *key, robj *val)
```

```Python3
def setKey(db, key, val):
    val = lookupKeyWrite(db, key)
    if val:
        dbOverwrite(db, key, val)
    else:
        dbAdd(db, key, val)
    val.refCount += 1
    removeExpire(db,key)
    signalModifiedKey(db,key)
```
