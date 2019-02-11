# How Dict add key-value

## dictAddRaw

```C
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
```

Low level add function.  

If key already exists, just return the existing entry. 
Else create entry, add to dict and return the entry.  

Why add only if key not exist?  
Robj contains not only data, it also has "common fields" 
such as cacheLFU and encoding. Simply replace old entry will 
wipe all "common fields", so let caller to handle it.     

```Python3
#return (added entry, exsisting entry)
def dictAddRaw(dict, key):
   if dict.isRehashing:
       _dictRehashStep(dict)
   idx, entry = _dictKeyIndex(dict, key)
   if idx == -1:
       return NULL, entry
   else:
       entry = DictEntry(key)
       ht = dict.rehash_table if dict.isRehashing else dit.table
       entryList = ht.slots[idx]
       entryList.addHead(entry)
       ht.size += 1
       return entry, NULL   
```

### dictAddRaw calling chain

* [dictAddRaw](add.md#dictaddraw)
* [_dictKeyIndex](find.md#_dictkeyindex)
* [_dictExpandIfNeeded](resize.md#_dictexpandifneeded)
* [dicExpand](resize.md#dictexpand)

## dictAdd

```C
int dictAdd(dict *d, void *key, void *val)
```

Return error if key exists, else add entry by calling 
[dictAddRaw](add.md#dictaddraw).

```Python3
def dictAdd(dict, key, val):
    entry, existing = dictAddRaw(dict, key)
    if entry:
        entry.val = val
        return True
    else:
        return False
```

## dictReplace

```C
int dictReplace(dict *d, void *key, void *val)
```

Add entry if key not exists, else replace old value. 

```Python3
def dictReplace(dict, key, val):
    entry, existing = dictAddRaw(dict, key)
    if entry:
        entry.val = val
        return True
    else:
        old = existing.val
        existing.val = val
        free(old)
        return False
```

## dictAddOrFind

```C
dictEntry *dictAddOrFind(dict *d, void *key)
```

Add entry to dict if key not exist, else find entry. 
Then return the entry.  

```Python3
def dictAddOrFind(dict, key):
    entry, existing = dictAddRaw(dict, key)
    return entry if entry else existing
```