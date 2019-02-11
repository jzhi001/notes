# Dict clear

## _dictClear

```C
int _dictClear(dict *d, dictht *ht, void(callback)(void *))
```

Clear one of the dict's table. 

* free all entries and key values
* free slots (`ht->table`)

```Python3
def _dictClear(dict, mainTable, callback):
    ht = dict.table if mainTable else dict.rehash_table
    for entryList in ht.slots:
        for entry in entryList:
            entryList.remove(entry)
    if mainTable:
        dict.table = NULL
    else:
        dict.rehash_table = NULL
    dictReset(dict) # reset fields
```

I haven't figure out this:

```C
if (callback && (i & 65535) == 0) callback(d->privdata);
```

## dictRelease

```C
void dictRelease(dict *d)
```

Clear both `dict->table` and `dict->hash_table` by calling 
[_dictClear](clear.md#_dictclear).  
No callback used.   

```Python3
def dictClear(dict):
    _dictClear(dcit, True, NULL)
    _dictClear(dcit, False, NULL)
```

## dictEmpty

```C
void dictEmpty(dict *d, void(callback)(void*))
```

```Python3
def dictEmpty(dict, callback):
    _dictClear(dcit, True, callback)
    _dictClear(dcit, False, callback)
    dict.rehshIdx = -1
    dict.iterators = 0
```

## dictReset

```C
static void _dictReset(dictht *ht)
```

Reset all fields in [dictht](../ht/ht.md#structure)

```Python3
def dictReset(ht):
    ht.slots = NULL
    ht.size = 0
```