# Dict find

## dictFind

```C
dictEntry *dictFind(dict *d, const void *key)
```

Find entry

## dictFetchValue

```C
void *dictFetchValue(dict *d, const void *key)
```

Get value

## _dictKeyIndex

```C
static long _dictKeyIndex(dict *d, const void *key, uint64_t hash, dictEntry **existing)
`````

Get slot index of the given key.  
If key already exists, -1 will be returned and `existing` is pointed 
to the existing entry.   
Index belongs to rehash table if dict is rehashing, else main table.  


```Python3
def dictIndexOfEntry(ht, key):
    idx, entry = ht.get(key)
    retIdx = -1 if entry else idx
    return retIdx, entry

def _dictKeyIdx(dict, key, hash):
    if not _dictExpandIfNeeded(dict):
        return -1, NULL
        
    idx, entry = dictIndexOfEntry(dict.table, key)
    if idx == -1:
        idx, entry = dictIndexOfEntry(dict.hash_table, key)
    return idx, entry
```

## dictFindEntryRefByPtrAndHash

Use provided hash and Object to find entry, the returned 
object must have the same address (like `==` operator in Java)
 with the one in argument. 

```C
dictEntry **dictFindEntryRefByPtrAndHash(dict *d, const void *oldptr, uint64_t hash)
```