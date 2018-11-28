# robj(Redis Object)

Redis中的数据库是sds => robj的哈希表，所有的数据在Redis中都是以robj表示的

## Redis Object 容器

```c
typedef struct redisObject {
    int type;    /* 类型 */
    void *ptr;   /* 实际的对象，如sds和adlist */
    int refcount;  /* 被引用次数 */
} robj;
```

### robj::type

```c
/* redisObject::type */
#define REDIS_STRING 0
#define REDIS_LIST 1
#define REDIS_SET 2  /* 该版本未实现set */
#define REDIS_SELECTDB 254  /* 特殊的type，用于dump.rdb */
#define REDIS_EOF 255   /* 特殊的type，用于dump.rdb */
```

### robj::refcount

Redis server通过refcount监视robj的被引用情况。如果一个robj没有被任何对象引用，robj指向的数据结构(`robj::ptr`)会被释放内存，robj自己并不会被释放内存，而是被放到垃圾堆(`redisServer::objfreelist`)。当Redis server需要创建新的robj时，它会先尝试复用垃圾堆中的robj，如果垃圾堆是空的才去分配新的内存。复用robj可以避免频繁的内存操作(malloc/free)，提高效率。

```c
struct redisServer{
    //省略其他字段
    list *objfreelist;  /* robj垃圾堆 */
};
```

#### robj的构造函数

```c
static robj *createObject(int type, void *ptr) {
    robj *o;

    //先翻垃圾堆找现成的
    if (listLength(server.objfreelist)) {
        listNode *head = listFirst(server.objfreelist);
        o = listNodeValue(head);
        listDelNode(server.objfreelist,head);
    } else {
        //没有垃圾才分配内存
        o = malloc(sizeof(*o));
    }
    if (!o) oom("createObject");
    o->type = type;
    o->ptr = ptr;
    o->refcount = 1;
    return o;
}
```

#### robj的"析构函数"

```c
static void decrRefCount(void *obj) {
    robj *o = obj;
    //如果robj没有被任何对象引用
    if (--(o->refcount) == 0) {
        //释放o->ptr指向的数据结构
        switch(o->type) {
        case REDIS_STRING: freeStringObject(o); break;
        case REDIS_LIST: freeListObject(o); break;
        case REDIS_SET: freeSetObject(o); break;
        default: assert(0 != 0); break;
        }
        //放到垃圾堆
        if (!listAddNodeHead(server.objfreelist,o))
            free(o);
    }
}
```

```c
static void incrRefCount(robj *o) {
    o->refcount++;
}
```

---

## String type robj

### robj::ptr

使用[sds](../sds.md)保存字符串

### 构造函数

因为比较简单所以没有构造函数

```c
createObject(REDIS_STRING, sds);
```

### 析构函数

```c
static void freeStringObject(robj *o) {
    sdsfree(o->ptr);
}
```

---

## List type robj

### robj::ptr

使用[adlist](../adlist.md)保存序列

### 构造函数

```c
static robj *createListObject(void) {
    list *l = listCreate();

    if (!l) oom("createListObject");
    listSetFreeMethod(l,decrRefCount);
    return createObject(REDIS_LIST,l);
}
```

### 析构函数

```c
static void freeListObject(robj *o) {
    listRelease((list*) o->ptr);
}
```