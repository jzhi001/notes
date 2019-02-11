# Dict size adjustment

Include resizing and rehashing.  

* resizing just create an empty Dict
* rehashing move elements to the new Dict

## Resize calling chain

### Server side resize chain

|File     | Function  | Description |
| ------- | --------- | ----------- |
|server.c | serverCron | time event  |
|server.c | databaseCron | time event |
|server.c | [tryResizeHashTables](resize.md#tryresizehashtables) | resize Dict if needed |
|server.c | [htNeedsResize](resize.md#htneedsresize) | whether a Dict needs to resize |
|dict.c   | [dictResize](resize.md#dictresize) | use element size to create Dict |
|dict.c | [dictExpand](resize.md#dictexpand) | create the new Dict and mark Dict to rehashing |

### User side resize chain

| Function  | Description |
| --------- | ----------- |
| [dictAddRaw](add.md#dictaddraw) | add an entry and return it |
| [_dictKeyIndex](find.md#_dictkeyindex) | get slot index to add key or existing entry |
| [_dictExpandIfNeeded](resize.md#_dictexpandifneeded) | if need expand, use element size * 2 to expand |
| [dictExpand](resize.md#dictexpand) | create the new Dict and mark Dict to rehashing |


## Rehash calling chain

### Server side rehash chain

|File     | Function  | Description |
| ------- | --------- | ----------- |
|redis.conf | activerehashing yes | start rehashing Dicts in `serverCron()` |
|server.c | serverCron | time event  |
|server.c | [databaseCron (partial)](resize.md#databasecron-partial) | time event |
|server.c | [incrementallyRehash](resize.md#incrementallyrehash) | rehash db's Dicts in 1 ms |
|dict.c | [dictRehashMilliseconds](resize.md#dictrehashmilliseconds)| migrate elements in given ms |
|dict.c | [dictRehash](resize.md#dictrehash) | migrate n elements |

### User side rehash chain

|File     | Function  | Description |
| ------- | --------- | ----------- |
|dict.c | [_dictRehashStep](resize.md#_dictrehashstep) | migrate one element to new Dict |
|dict.c | [dictRehash](resize.md#dictrehash) | migrate n elements |

Note: rehash step only performs when there's **NO** [safe iterator](iterator.md#safe-vs-non-safe).

This function is called by:

* dictAddRaw
* dictGenericDelete
* dictFind
* dictGetRandomKey
* dictGetSomeKeys

### _dictExpandIfNeeded

```C
static int _dictExpandIfNeeded(dict *ht)

#define DICT_HT_INITIAL_SIZE 4
static unsigned int dict_force_resize_ratio = 5
```

Called in `_dictKeyIndex`, which means expand dict if needed 
before find key index.  

A dict will be expanded:

* if resizable
    * size-slots ratio >= 1
* if NOT resizable
    * size-slots ratio >= 5

```Python3
def dictUsedRatio(dict):
    size = dict.table.size()
    slot = dict.table.slot.size()
    return size / slot

def _dictExpandIfNeeded(dict):
    if dict.isRehashing:
        return True
    elif dict.size == 0:
        return dictExpand(dict, DICT_HT_INITIAL_SIZE)
    else:
        if dict.can_resize and dictUsedRatio(dict) >= 1:
            return dictExpand(dict, dict.table.size * 2)
        elif not dict.can_resize and dictUsedRatio(dict) >= 5:
             return dictExpand(dict, dict.table.size * 2)
        else:
            return True
```

### databaseCron (partial)

This is part of `databaseCron()` function, which only runs 
if `activerehashing` is set to `yes`.  

Try to rehash one db in each cron.

```Python3
def activeRehash(flag):
    if flag:
        dbNum = len(server.dbs)
        for i in range(dbNum):
            if incrementallyRehash(i):
                return
```

### incrementallyRehash

```C
int incrementallyRehash(int dbid)
```

Rehash one of db's Dict within 1 ms.

```Python3
def incrementallyRehash(dbid):
    db = server.db[dbid]
    dict = db.dict
    expires = db.expires
    if dict.isRehashing:
        dictRehashMs(dict, 1)
        return True
    if expires.isRehashing:
        dictRehashMs(expires, 1)
        return True
    return False
```

### tryResizeHashTables

```C
void tryResizeHashTables(int dbid)
```

Database has two `Dict`: `db->dict` and `db->expires`. 
Resize them if needed.

```Python3
def tryResizeHashTables(dbid):
    db = server.dbid
    if db.dict.needsResize():
        db.dict.resize()
    if db.expires.needsResize():
        db.expires.resize()
```

### htNeedsResize

```C
int htNeedsResize(dict *dict)

#define DICT_HT_INITIAL_SIZE     4
```

A Dict needs to be resize if 

1. total slots > initial size (4)
2. elements count / slots <= 10%

`DICT_HT_INITIAL_SIZE` is initial size of slots.

### dictResize

```C
dictResize(dict *d)

#define DICT_HT_INITIAL_SIZE     4
```

Use element size `n` to create new table.  
Minimum size is 4.  

```Python3
def resize(dict):
    if dict.isRehashing:
        raise Exception()
    n = dict.table.size()
    if n < DICT_HT_INITIAL_SIZE:
        n = DICT_HT_INITIAL_SIZE
    return dict.expand(n)
```

### dictExpand

```C
int dictExpand(dict *d, unsigned long size)
```

New slot length is next power of `n`.     
Only creates a new **EMPTY** hash table, and set `isRehashing` flag
to true. Elements migrating is done step by step in other operations.

```Python3
def Dictexpand(dict, n):
    if dict.isRehashing:
        return False
    else:
        slots = nextPower(n)
        if slots == len(dict.table.slots):
            return False # no need to expand
        dict.rehash_ht = Dict(slots)
        dict.isRehashing = True
        return True
```

### dictRehashMilliseconds

```C
int dictRehashMilliseconds(dict *d, int ms)
```

The return value is never used. I guess it's because 
the number is not always equal to actual migrated number.

```Python3
def dictRehashMs(dict, ms):
    start = timeInMs()
    undone = dictRehash(dict, 100)
    while undone and timeInMs() - start < ms:
        undone = dictRehash(dict, 100)
```

Migrate in `ms`.

### _dictRehashStep

```C
static void _dictRehashStep(dict *d)
```

Migrate elements **ONLY** if there's no [safe iterators](iterator.md#safe-vs-non-safe).

```Python3
def dictRehashStep(dict):
    if dic.iterators == 0:
        rehash(dict, 1)
```

### dictRehash

```C
int dictRehash(dict *d, int n)
```

Try to migrate `n` elements.   
It's still time wasting since you may have to skip a lot of 
some empty slots to find an element. Instead, only **N * 10** 
empty visits are allowed. So number of actual migrated is **NOT** 
always equal to `n`.

```Python3
def dictRehash(dict, n):
    empty_chances = n * 10
    slots = dict.table.slots
    while n > 0 and empty_chances > 0 and dict.table.size > 0 and rehash_idx < len(slots):
        if slots[rehash_idx]:
            entryList = slots[rehash_idx]
            for entry in entryList:
                dict.rehash_table.add(entry.key, entry.val)
                entryList.remove(entry)
                dict.table.size -= 1
                n -= 1
        else:
            empty_chances -= 1
        rehash_idx += 1
    migrate_undone =  dict.table.size > 0
    return migrate_undone
```