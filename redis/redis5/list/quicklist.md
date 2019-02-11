# QuickList

## Good Articles

[post by 张铁蕾](http://zhangtielei.com/posts/blog-redis-quicklist.html)

## Introduction


## Config

* list-max-ziplist-size -2

    * positive: maximum elements of each node
    * negative: maximum bytes used
    
    `quickList->fill`
    
* list-compress-depth 0
    
    Compress nodes after certain depth. Depth means how many nodes 
    you have to traverse from head or tail to get to a certain node.  
    For example `head` and `tail` is at depth 0, `head->next` and 
    `tail->prev` is at depth 1. Head and tail is always 
    **uncompressed** for fast push/pop. So depth 0 means do not 
    config any node.  
    `quickList->compress` stores the depth. It is a 16-bit unsigned 
    int so valid depth is [0, 2**16 - 1], invalid depth will be 
    saturated.
    
## Confusing Parts

### Where is Ziplist

Raw ziplist is pointed by `quicklistNode->zl`; Compressed ziplist 
is pointed by `quicklistNode->zl->compressed`, here `zl` points to 
`quicklistLzf`. Compressed length is pointed by 
`quicklistNode->zl->sz` and `quicklistNode->sz` remains for 
decompressing.

### What is Recompress

`quicklistNode->recompress` is a 1-bit int. It tells whether 
the node is decompressed for use and needed to be compressed again.
 
It matters when compressing list. 
Node will be compressed if `recompress` is true, 
else the node stays raw encoding and we try compress the next/prev node.

### quicklist->compress

It means uncompressed node count on **one** side of list.
    
## Compress Node

Requirements for compression:

* node's ziplist raw encoded
* bytes used of ziplist is greater than 48

Use lzf to compress and then [store](quicklist.md#where-is-ziplist).

## Decompress Node

Reverse processing of compress.  
`quicklistNode->recompress` ??

## Compress List with Node

After node is inserted, list need to re-compressed according to depth: 
De-compress nodes outside depth and compress nodes inside depth.

## Need Optimization

* return false if `fill >= 0`
* return true if `fill` is valid and `sz < opt level`

```C
static const size_t optimization_level[] = {4096, 8192, 16384, 32768, 65536};
```

Note: You cannot de-reference an array variable, the compiler 
converts `*arr` to `*(&arr[0])`.

## Node Allow Insert

Check whether a ziplist is allowed to add new elemnt at head.  
First calculate how many bytes will be added: 

* bytes of new data, which is `sz`
* bytes of new data's encoding
* bytes of head elements's `prevlen` field (1 or 5 bytes in soure code)

New byte count is estimated as maximum result.

Then check whether bytes used of new ziplist is valid:

* if fill < 0 (bytes limited), bytes must under limit
* else (element size limited)
    * bytes must under `SAFE_LIMIT`, which is 8192
    * element size must under limit

```C
REDIS_STATIC int _quicklistNodeAllowInsert(const quicklistNode *node,
                                           const int fill, const size_t sz)
```


    