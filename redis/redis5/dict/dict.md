# Dict

## Why not just using Hash Table?

Answer is simple, for resizing.  

Normally, we resize a hash table by creating a new table and 
rehashing all elements to it.  

The problem is other operations are blocked if the hash table is huge. 
This is not acceptable for a "single-threaded" application like Redis.  

Redis resizing just go one step further than classic's, it keeps 
the new table and rehashing **dynamically** in **other API functions** 
(add, get, remove...). Server also helps rehashing in its time event.   

Dynamic rehashing adds complexity to other API functions since 
you have to operate two hash tables. That's why a new data 
structure is needed.

## Structure

|Field |Type | Purpose |
| --- | --- | --- |
| type | dictType* | provide clone() and free() for key and val, and key.equals|
| privdata | void* | length of slots |
| ht | dictht | main table and rehash table |
| iterators | unsigned long | number of iterators |

## API

* [add](add.md)
* [find value or entry](find.md)
* [remove and clear](remove.md)
* [resize and rehash](resize.md)
* [Iterator](iterator.md#dict-iterator)
* [Scan](iterator.md#dictscan)

 