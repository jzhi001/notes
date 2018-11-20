# SDS

## unsolved

long 和 long long 的区别

### kept for future

[Swallowing the Semicolon in C macros](https://gcc.gnu.org/onlinedocs/cpp/Swallowing-the-Semicolon.html#Swallowing-the-Semicolon)

[Stringizing in C marcos](https://gcc.gnu.org/onlinedocs/cpp/Stringizing.html#Stringizing)

---

## sds.h

### sds -> simple dynamic string

#### 为什么要用sds？

* 一般计算字符串长度，也就是调用strlen，需要O(n)时间。而Redis需要O(1)

* Redis希望减少分配/释放内存的次数，一般cat/trim操作一定会进行内存操作

* Redis希望sds可以复用库函数，如printf

#### sds是什么？

``` c
typedef char *sds;
```

sds就是char*的别名，也就是说所有对sds的操作就是对普通字符串的操作。虽然复用库函数这个需求解决了，但是说好的数据结构呢？

#### 简单版的sds

要实现上面的需求，sds只需要额外的两个字段：当前字符串长度和给字符串分配的内存长度。
有了字符串长度字段计算长度就变成了O(1)时间；记录了内存长度后字符串就可以多留一些内存用于以后的操作，而不是每次变更字符串长度时都要按新长度分配内存。

```c
struct sdshdr{ /* sds header */
    unsigned int len; /* 目前字符串的长度 */
    unsigned int alloc; /* 给字符串分配的内存长度，不包括最后的NULL */
    char[] buf; /* 字符串 */
};
```

数据结构告诉我们sds其实是被包含在sdshdr中的。但是前面说过sds才是这个库的主角，我们如何通过sds找到它所在的数据结构呢？  

答案就是C语言中的指针运算了：

```c
sds getHeader(sds *s){
    return (void*)(s - sizeof(struct sdshdr));
}
```

多说一句，sdshdr的定义方式用了[动态数组](https://stackoverflow.com/questions/2060974/how-to-include-a-dynamic-array-inside-a-struct-in-c)，也就是把空数组变量放在结构体字段的最后一位。这样这个数组占用的内存是算在结构体中，而且它没有固定长度限制。  

在sdshdr中**必须**使用动态数组而不是指针。如果使用指针：

```c
struct sdshdr{
    unsigned int len;
    unsigned int alloc;
    char *buf;
};
```

buf指向堆中的字符串地址，我们就无法通过sds找到sdshdr了。

#### 现实中的sds

简单的sds已经满足所有需求了，但是它还**不够效率**。看下面的sds header：

```c
struct sdshdr s = {
    3,
    3,
    "foo\x00"
};

assert(sizeof(s) == 12);
```

s是一个非常段的字符串(长度4字节)，但是它却需要额外2个uint记录长度！作为较底层的C程序，Redis希望len和alloc字段类型尽可能的小，也就是说s用两个char就可以了！  

Redis按照C语言的整数类型定义了不同的sdshdr类型，并增加了type字段保存类型,type的最低三位为不同类型的[mask](https://en.wikipedia.org/wiki/Mask_(computing))：

* uint8_t   ->  sdshdr8  (0x001)

* uint16_t  ->  sdshdr16 (0x010)

* uint32_t  ->  sdshdr32 (0x011)

* uint64_t  ->  sdshdr64 (0x100)

* SDS_TYPE_MASK (0x111)

* SDS_TYPE_BITS (3)

```c
struct sdshdr8{
    uint8_t len;
    uint8_t alloc;
    unsigned char flags;
    char[] buf;
};
```

所以我们应该用sdshdr8来保存"foo"，它比上面的节省了6个字节，并且在"foo"扩展到256字节之前都OK。  

可是如果我们从不扩展"foo"呢？如果我们确定不对字符串进行操作，那么我们就不需要alloc字段了。Redis为这种长度短、几乎不变的字符串定义了更紧凑的类型：

```c
struct sdshdr5{
    unsigned char flags; /* 最低三位表示type，其余五位表示长度 */
    char buf[];
}
```

这种类型可以存最长为32的字符串。但是由于节省了两个变量，获取长度就稍微麻烦一点：

```c
#define SDS_TYPE_5_LEN(f) ((f) >> SDS_TYPE_BITS)
```

源码中使用flags作为宏参数，那么怎么通过sds获取flags呢？ 答案是s[-1]。

#### sds中的"多态"

使用sds获取flags是简单的，因为在所有类型的sds header中flags字段都紧挨着字符串。我们利用这个相对距离进行指针运算获取了flags，那么其他的字段呢？比如sdshdr？是不是没那么简单了，因为不同header类型有不同长度的字段，所以我们要根据类型减去对应的header长度。Redis作者利用[宏的拼接](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html)避免了大量重复:

```c
#define SDS_HDR(T, s) (struct sdshdr##T*)((s)-sizeof(struct sdshdr##T))
#define SDS_HDR_VAR(T, s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));
```

但是其他操作就不能这样写了，比如获取len字段的sdslen：

```c
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

因为sds的库函数使用sds作为参数，所以我们要先获得sds的header类型，根据类型做处理。sds.h中大量的switch-case就是在写sds的"多态"  

#### sds.h 漏网之鱼

到这里sds.h中的内容就基本上看完了，我们还落下了一点东西  

```c
const char *SDS_NOINIT 
```

用于sds的构造函数[sdsnewlen](#sds-sdsnewlen(const-void-*init,-size_t-initlen))，将SDS_NOINIT作为参数表示不给字符串分配内存

```c
#define SDS_MAX_PREALLOC (1024*1024)
```

用于调整sds内存的函数[sdsMakeRoomFor](#sds-sdsMakeRoomFor(sds-s,-size_t-addlen))，表示可分配给sds的内存最大值

```c
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
```

sdshdr定义中用到了[C的类型参数](https://gcc.gnu.org/onlinedocs/gcc/Common-Type-Attributes.html#Common-Type-Attributes)

---

## sds.c

这里的代码都**不是源码**，而是我读过源码一段时间后自己实现的(防止默写代码)  
该模块中有现成的测试用例  
动手写一遍可以加深对模块抽象层面和底层细节(有时还有linux)的理解，也能提高C语言的熟练度  
并且很多函数都是算法题/笔试题的变种，非常值得实现  
相比源码，我自己的实现有更高的可读性(自认为)，但是效率更低(因为使用子函数重构)  
如果你看不懂源码，可以先参考我的代码

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

* SDS_MAX_PREALLOC表示可分配的内存的最大值
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

思路

* 最初缓存的大小为fmt的长度 * 2

* 用栈上分配的**静态**缓存比用堆上分配的缓存快（没有syscall）

* 如果缓存长度大于静态缓存长度，就只能用堆上分配的缓存

* vsnprintf的返回值不能告诉你缓存是否够用

* buf[-2] = NULL， 如果写后buf[-2] != NULL，就表示缓存不够，将缓存长度翻倍，再写一遍

* 循环后buf指向完整的字符串了，调用sdscat即可

注意

1. 静态缓存指的是char[1024]这种使用编译时就能确定的值初始化数组

2. char[x]会在栈上分配，但是可能因为x过大导致栈溢出

3. 上面的方法是[VLA](https://en.wikipedia.org/wiki/Variable-length_array#C99)，并不推荐使用

4. 因为va_list只能用一次，所以在循环中要用va_copy复制va_list

#### sds sdscatprintf(sds s, const char *fmt, ...)

调用后所有对*s*的引用都不再合法，需要修改为返回值

注意

* 使用va_list前后必须有va_start和va_end

* 调用va_arg会修改va_list，而va_copy不会

#### sds sdscatfmt(sds s, char const *fmt, ...)

和sdscatprintf相似，但是比它快，因为该函数没有调用libc中的printf  
该函数只实现了几个format：

* %s - C string

* %S - sds string

* %i - signed int

* %I - 64 bit signed integer

* %u - unsigned int

* %U - 64 bit unsigned integer

* %% - verbatim '%'

#### sds sdstrim(sds s, const char *cset)

删除*s*中所有包含在C-string *cset*中的字符  
调用该函数后，所有对*s*的引用不再合法，应修改为该函数的返回值

例子：

```c
sds s = sdsnew("AA...AA.a.aa.aHelloWorld     :::");
sds s = sdstrim(s,"Aa. :");
printf("%s\n", s);
```

会输出 "Hello World"

思路： 使用双指针，判断cset中是否含有字符

#### sds sdsrange(sds s, ssize_t start, ssize_t end)

返回s[start, end]，*start*和*end*可能为负数

注意：start和end可能会越界，如果越界应该调整至左右边界

#### void sdstolower(sds s)

#### void sdstoupper(sds s)

#### int sdscmp(const sds s1, const sds s2)

先按照较短sds的长度使用memcmp()比较两个sds  
如果相同，再比较长度

#### sds *sdssplitlen(const char *s, ssize_t len, const char *sep, int seplen, int *count)

使用*sep*分割*s*并返回sds数组头，返回的数组长度放在*count*中  
sep可能是字符串

以下情况返回NULL

* 内存不足

* *s*或*sep*长度为0

思路

* 使用偏移量start表示子字符串开头，偏移量p表示当前位置  
* 遍历p，如果p处匹配sep，就把s[start:p-1]加入结果  
* 遍历结束后单独加上最后一个sep后面的子字符串  
* 因为不知道结果数量，所以需要动态调整堆上分配的内存
* 源码中使用了goto，用于错误后的处理（相当于finally）

#### void sdsfreesplitres(sds *tokens, int count)

释放sdssplitlen返回的数组内存  
注意：要先释放所有sds，再释放数组

#### sds sdscatrepr(sds s, const char *p, size_t len)

在s后面添加字符串p，并将p中所有不可打印字符(用isprint检测)变成字符串形式('\n' -> "\\n")  
被添加的字符串需用""包裹
该函数用于模拟用户输入    
函数返回后所有对s的引用不再合法，应该修改为函数的返回值

需要转化的字符有:

* '\\'

* '"'

* '\t'

* '\n'

* '\r'

* 'a'

* 'b'

* 其它不可打印的十六进制数 -> "\\xAB"

例子：

```c
sds s = sdsnew("line: ");
char *p = {'a', '\n', 'b', '\t', '\xFF'};
s = sdscatrepr(s, p, 5);
printf("%s\n", s);
```

会输出 "a\nb\t\xFF"

#### sds *sdssplitargs(const char *line, int *argc)

按空格分割*line*中的参数并返回结果数组，数组长度存在*argc*中  
参数中的空格会被忽略  
该函数用于处理用户输入  
调用者要手动调用sdsfreespliters()释放返回数组的内存

以下情况返回NULL

* 引号不匹配，如: 'ab"

* 右引号后不是空格，如： 'foo'bar

注意去掉最外层的引号并处理转义字符

注：非常适合当做算法题

#### sdsmapchars(sds s, const char *from, const char *to, size_t setlen)

将from中的字符替换成to，调用后原来的引用不再合法

如：

s = "hello", from = "ho", to = "01"  
则返回结果为 "0ell1"

#### sds sdsjoin(char **argv, int argc, char *sep)

C-string版本的join，返回使用*sep*将argv连接的sds

#### sds sdsjoinsds(sds *argv, int argc, const char *sep, size_t seplen)

sds版本的join