# Dict iterator

## Structure

```C
typedef struct dictIterator {
    dict *d;
    long index; //current slot
    int table //main table or rehash table
    int safe;
    dictEntry *entry, *nextEntry;
    /* unsafe iterator fingerprint for misuse detection. */
    long long fingerprint;
} dictIterator;
```

## Safe vs Non-safe

Here safe means it's OK to **modify** dict when having iterators.  

* Non-safe iterator: only supports `next()`
* Safe iterator: supports all function

In other words, dict must be un-modified when any non-safe iterator exists.

## dictFingerprint

```C
long long dictFingerprint(dict *d)
```

Supports non-safe iterators.   
Finger print is calculated when creating and 
releasing non-safe iterators. If dict is modified by 
non-safe iterators, two finger prints are different.  

```psudo
i1 = main_table.address
i2 = main_table.slots
i3 = main_table.size
i4 = main_table.address
i5 = main_table.slots
i6 = main_table.size

def fingerPrint():
    arr = [i1, i2, i3, i4, i5, i6]
    h = 0
    for i in range(6):
        h += arr[i]
        h = hash(h)
    return h
```

## dictNext

```C
dictEntry *dictNext(dictIterator *iter)
```

* If non-safe, generate finger print.  
* If rehashing, iterate rehash table after finished main table.

## dictScan

### Overview
  
* To perform iteration, start with `scan 0`, use the returned 
value to continue until Redis returned 0.  

* If elements is modified in a bucket that is already iterated by 
scan, that change will be missed

* But returned keys may be duplicated.

### Good articles

 * [How the Redis Hash Table Scan Function Works](https://medium.freecodecamp.org/redis-hash-table-scan-explained-537cc8bb9f52)
 * [Redis 深度历险：核心原理与应用实践](https://book.douban.com/subject/30386804/) Chapter 1.11

### Guessing slot index during rehashing

Suppose dict `d` has 8 slots, A key `k` is in `slot[2]`. Now 
`d` is resized to a 16-slot table, can you tell which slot `k` 
is in without having `hash(k)`? And what about `d` shrinking 
to 4 slots?

The answer is, index is 2 (0b010) or 6 (0b110) for 16-slot, 0 or 
1 for 4-slot. Why is that? 

In Redis, length of slots is always power of 2, so the bitwise and method  
is used: `hash(key) & (len(slots) - 1 )`, which means only get `n` least 
significant bits and the result is in `[0, len(slot) - 1]`.  

Back to the question, let's assume `hash(k) = 0b1101_1010)`and 
`len(slots) = 8`, so slot index is `0b1101_1010 & 0b0111 = 0b10`. 
If dict expands to 16-slot, actual slot index is `0b1101_1010 & 0b1111 = 0b1010`. 
But you can also guess it without the hash because we **trimmed** hash value 
to get 4-slot-index, which means the hash is `0b?010` and the answer is in 
one of two possibilities, `0b0010` and `0b1010`.

### Cursor value

Cursor value is slot index. So starting with value 0 means 
starting from the first slot.  

When a dict is expanded, you have to iterate table's slot and 
all the possible slots. If a 4-slot dict expands to 16-slot 
dict, and cursor value is `0b10`, the slots you have to go 
through are: 

* old table: 0b0010
* new table:
    * 0b0010 
    * 0b0110
    * 0b1010
    * 0b1110

But when dict is contracted, multiple slots in old table 
are mapped to a single slot in table. In this case, the 
cursor value represents slot index of new table, and you do 
the same as above. 

The pattern is, value of a cursor is always the slot 
index of the **SMALLER** table.  

### Cursor algorithm

A simple implementation (assume dict is rehashing):

```Python3
def guess(cursor, sizeDiff):
    result = []
    x, cursorLen = cursor, 0
    while(x > 0):
        cursorLen += 1
        x >>= 1
    for i in range(2 ** sizeDiff):
        x = cursor + i << cursorLen
        result.append(x)
    return result

# here t0 is smaller table
def scan(t0, t1, cursor):
    process(t0.slots[cursor])
    process(t1.slots[cursor])
    possibilities = guess(cursor, t1.sizemask - t0.sizemask)
    for i in possibilities:
        process(t1.slots[i])
    return cursor + 1
```

What Redis uses is higher order bits iteration, which is 
easier for generating all guesses. 

```C
v |= ~m1;
v = rev(v);
v++;
v = rev(v);
```

`m0 ^ m1` sets only different bits of two `sizemask` to 1. 
Continue guessing if `v & (m0 ^ m1)`.  

  