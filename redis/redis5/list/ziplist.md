# Ziplist

## Gist

Elements are embedded in a ziplist 
(no need for `e->next` pointer). 
For each entry, instead of storing `e->prev`, 
**SIZE** of previous entry is stored (get previous 
by pointer arithmetic).  

So memory is saved by:

* not using `e->next` pointer
* use less bytes to hold previous entry's bytes 
than pointers (often but not always)   

## Structure

```Redis
ziplist: [zlbytes] [entryCount] [zltail] [zlentry] [zlentry] ... [zlend]
entry: [prelen] [encoding (len)] [data]
```

## Design ziplist step by step

### Simplest

```Redis
ziplist: [zllen] [zlentry] [zlentry] ...
entry: [len] [data]
```  

This is like an Array Reply: first size of entries, then 
data size and data.  
A ziplist `['foo', 'bar', '123']` looks like this:  

```Redis
[3] [3 f o o] [3 b a r] [3 1 2 3]
```

### Next Element

```C
entry *next(Entry *e){
    size_t data_len = size(e);
    return e + data_len + ENTRY_SIZE_BYTES;
}
```

This function is good for all elements except the last one. 
To prevent this, we add `End of List` field, it's value is 
`0xFF` and **NO OTHER** bytes has the same value. 

```C
#define LIST_END 255
entry *next(Entry *e){
    size_t data_len = dataSize(e);
    e = e + data_len + ENTRY_SIZE_BYTES;
    return ( *((unsigned char*)e) == LIST_END) ? NULL : e;
}
```

### Push and Pop

To push or pop, we need current bytes to call `realloc` and 
O(1) time function to get the last element. 

```Redis
[zlbytes] [zltail] [zllen] [zlentry] ... [zlend]
```

### Double linked entry

`[prevLen] [len] [data]`  

To be able to move backward, we just add how many bytes 
previous elements used to current element.

### Previous Size Encoding

Goal: Use minimum bytes to store previous element's bytes. 

If `prelen in [0, 253]`, just use one byte.  
Else, use `0xFE` followed with 4 bytes which stores `prelen`. 

### Encoding

Normally data is binary-safe string, but if data is a 
number, we convert it to integer for saving memory.  

So let's add a bit to specify type, 0 for string and 1 for int.

```Redis
[type] [len] [data]
```  

### String

Redis uses 1 byte, 2 bytes and 4 bytes to store length and 
string encoding.

* 00pppppp 6 bits length
* 01pppppp|qqqqqqqq 14 bits length
* 10000000|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt 4 bits length

### Integer

We need to category all signed integer types:  

* byte 00
* short 01
* int  10
* long 11

Combine integer types with integer encoding: 

```Rdis
byte 0100_0000 [1 byte]
short 0101_0000 [2 bytes]
int  0110_0000 [4 bytes]
long 0111_0000 [8 bytes]
``` 

Redis use `11` as integer encoding to align with string encoding. 
It uses different mask and add 2 integer types(24-bit int and tiny int).

### End of List

Note: `255` is also an encoding.


  



