# Dict random

## dictGetRandomKey

```C
dictEntry *dictGetRandomKey(dict *d)
```

1. random a slot until it's not null
2. random a element from that slot

## dictGetSomeKeys

```C
unsigned int dictGetSomeKeys(dict *d, dictEntry **des, unsigned int count)
```

Sample some keys from dict.  

### Overview

* keys are more like *shuffled* than randomised , so this 
function is faster then randomising every key.
* re-random if continuously encounter empty slot

### How to iterate?

Now only deal with one table

1. only move `count * 10` steps  
2. random a slot index `i` as starting point, then iterates from 
`i` to end, then from head to `i`

```Python3
steps = count * 10
i = randint(slots.length)

while steps:
    precess(slots[i])
    i = (i + 1) % len(slots) # same with (i + 1) & sizemask
    steps -= 1
```

### what to do with a slot?

Now only deal with one table

* If slot is empty, move to next slot. If passed five continuous empty 
slots, re-random a slot.  
* If slot is not empty, add entries until reach requirement.  

```Python3
result = []
steps = count * 10
i = randint(slots.length)
n_empty = 0

while steps:
    slot = slots[i]
    if slot:
        for entry in slot:
            if len(result) == count:
                return result
            else:
                result.append(entry)
    else:
        n_empty += 1
        if n_empty == 5:
            i = randint(slots.length)
            
    i = (i + 1) % len(slots) # same with (i + 1) & sizemask
    steps -= 1
```

### deal with rehashing

1. process one slot of two tables in each loop (zigzag)
2. When dict is being rehashed, skip `main_table[0: rehash_idx - 1]`
3. If you use one randomized index to iterate two tables, 
handle when out of range

### clear logic

It is way better to use two index for each table.  
It is also way better to separate two tables' iteration. Index 
handling part is confusing to me.