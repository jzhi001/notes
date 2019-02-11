# Dict remove

## dictGenericDelete

```C
static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree)
```

Generic remove function, is not called outside.  

Find and remove entry of a given key. It's up to caller whether 
to free key and value.  

If caller choose not freeing the entry, it is returned for 
caller to free it somewhere else.

```Python3
#skip details
def findEntry(ht, key):
    return ht.find(key)

def dictGenericDelete(dict, key, freeEntry):
    if dict.isRehashing:
        _dictRehashStep(dict)
    entry = findEntry(dict.table, key)
    if entry:
        dict.table.remove(entry) #skip details
    else:
        if dict.isRehashing:
            entry = findEntry(dict.rehash_table, key)
    if entry:
        if freeEntry:
            free(entry.key)
            free(entry.val)
            free(entry)
        return True
    else:
        return False
```

## dictDelete

```C
int dictDelete(dict *ht, const void *key) 
```

Remove and free.

```Python3
def dictDelete(dict, key):
    entry = dictGenericDelete(dict, key, freeEntry = True)
    return False if entry is NULL else True
```

## dictUnlink

```C
dictEntry *dictUnlink(dict *ht, const void *key)
```

Remove but not free. Support Async removes.

```Python3
def dictDelete(dict, key):
    return dictGenericDelete(dict, key, freeEntry = False)
```

## 

```C
void dictFreeUnlinkedEntry(dict *d, dictEntry *he)
```

Helper function for freeing the unlinked entry.

```Python3
def dictFreeUnlinkedEntry(dict, entry):
    if entry:
        free(entry.key)
        free(entry.val)
        free(entry)
```