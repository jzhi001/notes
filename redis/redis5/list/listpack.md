# Listpack

## Introduction

See [Antirez's gist](https://gist.github.com/antirez/66ffab20190ece8a7485bd9accfbc175), 
it's clear enough.

## Difference with Ziplist

Entries in a ziplist hold length of previous entry. When 
inserting an entry `e`, `e->next` should modify `prelen`, 
and so does `e->next->next`... This is called cascade updating.  

entries in Listpack stores length of itself so the cascade 
problem is avoided.  

## Entry Structure

```Redis
<encoding-type><element-data><element-tot-len>
|                                            |
+--------------------------------------------+
            (This is an element)
```

## Functions

```C
uint32_t lpCurrentEncodedSize(unsigned char *p)
```

Given `listpack*` p, return `len(encoding) + len(data)`.  

```C
unsigned long lpEncodeBacklen(unsigned char *buf, uint64_t l)
```

Given encoding length + data length `l`, generate 
`<element-tot-len>` and return its length. If `buf` is not 
`NULL`, put `<element-tot-len>` in `buf`.

```C
unsigned char *lpSkip(unsigned char *p)
```  

Given `listpack*` p, return pointer to next element.  

* use encoding to calc <element-tot-len> (encoding length + data length)
* calc bytes needed to store <element-tot-len>
* now length of an element is known, skip 