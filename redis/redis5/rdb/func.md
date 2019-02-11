# Note of Functions in rdb.c

## Function notes


### General Write

`[len] [data]`

Calls `rioWrite` to write.

```C
static int rdbWriteRaw(rio *rdb, void *p, size_t len)
```

* rdbSaveType -> write one `char`

### Helper: String to Int

Write `value` into buffer `enc` and return bytes written.

```C
int rdbEncodeInteger(long long value, unsigned char *enc)
```

### Try Saving String as Int

Note: cannot save `long long`.

Store as int if string value can parsed to int.

```C
int rdbTryIntegerEncoding(char *s, size_t len, unsigned char *enc)
```

### Save Compressed

Save in sequence:

* bitmask
* length after compression
* original length
* compressed data

```C
ssize_t rdbSaveLzfBlob(rio *rdb, void *data, size_t compress_len,
                       size_t original_len)
```

Write a compressed object (original length > 4).

```C
ssize_t rdbSaveLzfStringObject(rio *rdb, unsigned char *s, size_t len)
```

### Save String

Do in sequence:

* try saving as int
* save compressed if length > 20
* save raw string

Empty string is allowed, which is `[0] []`.

```C
ssize_t rdbSaveRawString(rio *rdb, unsigned char *s, size_t len)
```

### Save String Object

If `obj` is int encoded(type of `robj->ptr` is `long long`), 
[save as long](rdb.md#save-long). 
Else, [save as string](rdb.md#save-string).

```C
ssize_t rdbSaveStringObject(rio *rdb, robj *obj)
```

### Save Long

First [try saving as int](rdb.md#try-saving-string-as-int), 
if overflow save as string.

```C
ssize_t rdbSaveLongLongAsStringObject(rio *rdb, long long value)
```

### Save Robj Type

Use switch case to save types. Calls `rdbSaveType`.

```C
int rdbSaveObjectType(rio *rdb, robj *o)
```