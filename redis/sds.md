sds.h

### 未解决

#define SDS_MAX_PREALLOC (1024*1024)

sdshdr5 的操作

long 和 long long 的区别

-----------------------------------------------------------------------------------------------------------------------------------------

sds -> simple dynamic string
sdshdr -> sds header

T: sdshdr type，也就是sdshdr中len的长度

sds是主角，sds所在的sdshdr中其他的字段是sds的header

```c
struct sdshdrT{
    uintT_t len;  /* 已使用长度 */
    uintT_t alloc; /* sds总长（不算header和字符串结尾的NULL） */
    unsigned char flags; /* 最低三位表示T，其余五位没有用 */
    char buf[]; /* sds */
};

struct sdshdr5 {
    unsigned char flags; /* 最低三位表示flag，其余五位表示长度 */
    char buf[];
};
```

const char *SDS_NOINIT 
表示不初始化buf，定义在sds.c中

``` c
typedef char *sds;
```

### sds mask

``` c
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
#define SDS_TYPE_MASK 7
#define SDS_TYPE_BITS 3
```

sds5   000  
sds8   001  
sds16  010  
sds32  011  
sds64  100  
sds  mask  111  


[Swallowing the Semicolon in C macros](https://gcc.gnu.org/onlinedocs/cpp/Swallowing-the-Semicolon.html#Swallowing-the-Semicolon)

[Concatenation in C marcos](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html)

[Stringizing in C marcos](https://gcc.gnu.org/onlinedocs/cpp/Stringizing.html#Stringizing)

#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)))
使用sdshdr type和sds声明sdshdr指针变量sh

T: sds type, s: sds

sds s;
SDS_HDR_VAR(64, s)
struct sdshdr64 *sh = (void*)(s - sizeof(struct sdshdr64))

指针运算，通过sds找到sdshdr

#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))
使用sdshdr type和sds将sds转换为shshdr指针

#define SDS_TYPE_5_LEN(f) ((f)>>SDS_TYPE_BITS)
不知道

``` c
static inline size_t sdslen(const sds s) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return SDS_TYPE_5_LEN(flags);
        case SDS_TYPE_8:
            return SDS_HDR(8,s)->len;
        case SDS_TYPE_16:
            return SDS_HDR(16,s)->len;
        case SDS_TYPE_32:
            return SDS_HDR(32,s)->len;
        case SDS_TYPE_64:
            return SDS_HDR(64,s)->len;
    }
    return 0;
}
```

计算sds的长度。思路是通过sds拿到sdshdr指针，然后返回其中的len字段  
sds[-1]是指针运算，可以得到sdshdr中sds前面是char flags，然后通过flags & SDS_TYPE_MASK获得sdshdr type，最后调用SDS_HDR(T,s)拿到sdshdr指针就能获取成员变量len
如果是sdshdr5，flags的最低三位表示type，其余五位表示长度。所以flags>>3可以得到长度

```c
static inline size_t sdsavail(const sds s) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5: {
            return 0;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            return sh->alloc - sh->len;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            return sh->alloc - sh->len;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            return sh->alloc - sh->len;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            return sh->alloc - sh->len;
        }
    }
    return 0;
}
```

获得sds的空闲长度。思路是获得sds所在的sdshdr指针，然后返回sh->alloc - sh->len
sdshdr5为什么返回0？？

```c
static inline void sdssetlen(sds s, size_t newlen) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            {
                unsigned char *fp = ((unsigned char*)s)-1;
                *fp = SDS_TYPE_5 | (newlen << SDS_TYPE_BITS);
            }
            break;
        case SDS_TYPE_8:
            SDS_HDR(8,s)->len = newlen;
            break;
        case SDS_TYPE_16:
            SDS_HDR(16,s)->len = newlen;
            break;
        case SDS_TYPE_32:
            SDS_HDR(32,s)->len = newlen;
            break;
        case SDS_TYPE_64:
            SDS_HDR(64,s)->len = newlen;
            break;
    }
}
```

设置sds的长度。思路是通过sds拿到它所在的sdshdr *sh，然后修改sh->len。
如果是sdshdr5，则修改flags的前五位

```c
static inline void sdsinclen(sds s, size_t inc) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            {
                unsigned char *fp = ((unsigned char*)s)-1;
                unsigned char newlen = SDS_TYPE_5_LEN(flags)+inc;
                *fp = SDS_TYPE_5 | (newlen << SDS_TYPE_BITS);
            }
            break;
        case SDS_TYPE_8:
            SDS_HDR(8,s)->len += inc;
            break;
        case SDS_TYPE_16:
            SDS_HDR(16,s)->len += inc;
            break;
        case SDS_TYPE_32:
            SDS_HDR(32,s)->len += inc;
            break;
        case SDS_TYPE_64:
            SDS_HDR(64,s)->len += inc;
            break;
    }
}
```

增加sds的长度，思路和sdssetlen相同

```c
static inline size_t sdsalloc(const sds s) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return SDS_TYPE_5_LEN(flags);
        case SDS_TYPE_8:
            return SDS_HDR(8,s)->alloc;
        case SDS_TYPE_16:
            return SDS_HDR(16,s)->alloc;
        case SDS_TYPE_32:
            return SDS_HDR(32,s)->alloc;
        case SDS_TYPE_64:
            return SDS_HDR(64,s)->alloc;
    }
    return 0;
}
```

获取sds的总长度

```c
static inline void sdssetalloc(sds s, size_t newlen) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            /* Nothing to do, this type has no total allocation info. */
            break;
        case SDS_TYPE_8:
            SDS_HDR(8,s)->alloc = newlen;
            break;
        case SDS_TYPE_16:
            SDS_HDR(16,s)->alloc = newlen;
            break;
        case SDS_TYPE_32:
            SDS_HDR(32,s)->alloc = newlen;
            break;
        case SDS_TYPE_64:
            SDS_HDR(64,s)->alloc = newlen;
            break;
    }
}
```

设置sds的总长度

------------------------------------------------------------

sds.c

```c
static inline int sdsHdrSize(char type) {
    switch(type&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return sizeof(struct sdshdr5);
        case SDS_TYPE_8:
            return sizeof(struct sdshdr8);
        case SDS_TYPE_16:
            return sizeof(struct sdshdr16);
        case SDS_TYPE_32:
            return sizeof(struct sdshdr32);
        case SDS_TYPE_64:
            return sizeof(struct sdshdr64);
    }
    return 0;
}
```

根据type返回对应sdshdr类型的长度（字节）

```c
static inline char sdsReqType(size_t string_size) {
    if (string_size < 1<<5)
        return SDS_TYPE_5;
    if (string_size < 1<<8)
        return SDS_TYPE_8;
    if (string_size < 1<<16)
        return SDS_TYPE_16;
#if (LONG_MAX == LLONG_MAX)
    if (string_size < 1ll<<32)
        return SDS_TYPE_32;
    return SDS_TYPE_64;
#else
    return SDS_TYPE_32;
#endif
}
```

Req应该是require
根据字符串长度返回对应的sdshdr type
LONG_MAX VS LLONG_MAX ??

#### sds sdsnewlen(const void *init, size_t initlen) 

sds的构造函数

参数

init: 字符串
initlen: 字符串长度

要求

init = NULL，buf会被初始化为全0
init = SDS_NOINIT，buf不会初始化
len = 0（空字符串） 用sdshdr8，Redis中的空字符串总是用来扩展的
字符串后面自动加NULL，为了利用printf等库函数
字符串中含有NULL也没问题（兼容二进制数组），因为sdshdr中包含长度

思路

根据字符串长度获取对应的容器类型type
根据type分配内存（sdshdr的长度 + initlen + 1（NULL terminator））
根据type初始化字段 len = alloc = initlen, flag
复制字符串到buf字段
buf结尾加上'\0'
返回sds(sh->buf)

#### sds sdsempty(void)

sds可选构造参数，构造长度为0的空字符串sds

#### sds sdsnew(const char *init)

sds可选构造参数，使用C-string(以NULL结尾的字符串)进行构造

参数

init: C-string

#### sds sdsdup(const sds s)

深拷贝sds

#### void sdsfree(sds s)

sds的析构函数

参数

s: 被析构的sds，可能是NULL

要求

如果s = NULL，直接返回

#### void sdsupdatelen(sds s)

将sds的长度更新为strlen(s)，也就是从头到第一个NULL的长度
这个函数用于手动操作sds，如：

```C
sds s = sdsnew("foobar");
s[2] = '\0';
sdsupdatelen(s);
printf("%d\n", sdslen(s));
```

会输出 "2"
如果不调用sdsupdatelen(),会输出 "6"

#### void sdsclear(sds s)

将sds长度设为0，以清空字符串
不用释放旧字符串占用的内存，它们将用于以后的扩展操作

#### sds sdsMakeRoomFor(sds s, size_t addlen)

确保sds有至少*addlen*字节的可用空间，和一个额外的字节给NULL
在进行扩展操作时应该先调用该函数

扩展要求

* 如果新长度 < SDS_MAX_PREALLOC，最终长度为新长度 * 2
* 否则最终长度为新长度 + SDS_MAX_PREALLOC 

注意

* 该函数并不修改sds的**长度**，也就是sdslen()的返回值
* 不要用sdshdr5作为新类型，因为sdshdr5没有记录可用空间的字段

思路

先拿到sds的可用空间avail  
如果avail >= addlen，就直接返回  
否则根据扩展要求算出新的长度和对应的sdshdr类型 type  
根据type扩展sdshdr的内存  
如果扩展前后type一样可以用realloc  
否则只能用malloc（header中字段的长度有变化）重新分配内存  
修改header字段  

#### sdsRemoveFreeSpace(sds s)

通过重新分配sds的内存消除sds后面的可用空间  
sds保持不变，但是下一次扩展操作会重新分配内存  
调用该函数后，参数中的sds及所有指向该sds的引用不再合法，需要指向该函数的返回值

思路

根据sds长度得到新的sdshdr类型type  
如果old type = type || type > SDS_TYPE_8 就保留old type  
否则就重新分配内存  
修改字段  

#### size_t sdsAllocSize(sds s)

返回sdshdr占用的内存大小，包括：

1. sds header

2. string

3. free buffer

4. NULL

#### void *sdsAllocPtr(sds s)

返回sds所在的sdshdr指针

#### void sdsIncrLen(sds s, ssize_t incr)

根据*incr*改变sds的长度和剩余空间，在末尾加上NULL  

注意： *incr*可能是负数

用例： 将sds当做buffer用，从kernel中读取字节到sds末尾  
如果是一般的字符串(char*)，则要单独声明缓存，读取完成后再strcat()

```C
size_t oldlen = sdslen(s);
sds s = sdsMakeRoomFor(s, BUFFER_SIZE);
size_t nread = read(fd, s+oldlen, BUFFER_SIZE);
if(nread) sdsIncrLen(s, read);
```

思路

在修改长度前要验证*incr*是否合法  
如果*incr* > 0，新长度是否不超过sdshdr的最大长度  
如果小于0，当前长度是否大于-*incr*

#### sds sdsgrouzero(sds s, size_t len)

扩展字符串，扩展的部分全部置0  
如果*len* < 当前长度，直接返回  

#### sds sdscatlen(sds s, const void *t, size_t len)

在*s*后面添加*t*指向的*len*长度内容  
函数调用后，参数中的sds不再合法，所有指向它的引用要指向该函数的返回值

#### sds sdscat(sds s, const char *t)

在sds后面添加C-string

#### sdscatsds(sds s, const sds t)

在*s*后面添加*t*
函数调用后，参数中的sds不再合法，所有指向它的引用要指向该函数的返回值

#### sds sdscpylen(sds s, const char *t, size_t len)

将*s*修改为*t*指向的*len*长度内容（可能为二字节流）

#### sds sdscpy(sds s, const char *t)

同sdscpylen()，但是t必须指向C-string

#### int sdsll2str(char *s, long long value)

将数字转换为字符串存在*s*中，返回*s*的长度
*s*必须指向一个至少长为SDS_LLSTR_SIZE的字符串  
该函数是sdscatlonglong()的helper function

```c
#define SDS_LLSTR_SIZE 21 /*在我的机器上long long最多占20个字符（包括负号）*/
```

思路:

* 将abs(value)从低位到高位存在s中

* 如果value是负数就加上'-'，然后加上'\0'(此时数字是倒序的)

* 计算出长度

* 翻转s，得到正序

#### int sdsull2str(char *s, unsigned long long v)

同sdsll2str

#### sds sdsfromlonglong(long long value)

返回值为*value*的sds

#### sds sdscatvprintf(sds s, const char *fmt, va_list ap)

可变长度参数版本的sdscatprintf()


