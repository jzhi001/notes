# ziplist

## 结构

默认是小端

(zl-bytes)  (zl-tail)  (zl-len)  (entry)  (entry)  ...  (entry)  (zl-end)

uint32_t zl-bytes: ziplist的总长度(包括zl-bytes自己)，有了总长度无需遍历ziplist就可以调整它的内存

uint32_t zl-tail: 最后一个entry的偏移量，用于O(1)时间的尾部pop

uint16_t zl-len: entry数量。如果数量多于 2^16-2 该值会被设为 2^16-1，并且只能通过遍历获取entry数量

uint8_t zl-end: 常量 255，表示ziplist结束。任何entry的第一个字节的值一定不是255

## entries

entry以两个元数据字段开头

1. prevlen: 上一个entry的长度

2. encoding: 表示存放的数据类型，包括integer和多种类型的string

完整的entry结构如下：

```c
<prevlen> <encoding> <entry-data>
```

有时encoding就是entry-data，如较小的integer。这样结构就变成了：

```c
<prevlen><encoding>
```

### 关于prevlen

1. 如果上一个entry的长度小于254，prevlen会使用uint_8保存

2. 否则prevlen使用**5个字节**保存。第一个字节的值固定位254(0xFE)，表示prevlen是5个字节，剩下四个字节为实际长度

### 关于encoding

encoding的第一个字节表示entry类型和entry长度

#### entry表示string

encoding的前两个bit表示entry的长度类型，然后是实际长度  

##### 1 byte (00pppppp)

1个byte的encoding有6位表示长度，最大长度为63

##### 2 bytes (01pppppp)(qqqqqqqq) big endian

2个byte的encoding有14位表示长度，最大长度为16383  

注意：14 bit的长度是按照**大端**储存的

##### 5 bytes (10000000)(qqqqqqqq)(rrrrrrrr)(ssssssss)(tttttttt) big endian

对于长度大于16383的string entry，后四个字节表示实际长度，最大长度为2^32-1。
第一个字节的后六位全部为0  

注意：14 bit的长度是按照**大端**储存的

#### entry表示integer

encoding的前两个bit固定为11，后面两位表示integer的类型  

注意：所有integer entry都是按照**小端**储存的，在大端系统上也一样

##### 3 bytes (11000000)(2 bytes)

int16_t

##### 5 bytes (11010000)(4 bytes)

int32_t

##### 9 bytes (11100000)(8 bytes)

int64_t

##### 4 bytes (11110000)(3 bytes)

24 bit signed

##### 2 bytes (11111110)(1 byte)

8 bit signed

##### 4 bits (1111xxxx)

xxxx在0001和1101之间，表示[0, 12]。因为0000和1111不能用(表示其他类型)，所以实际表示的长度为4 bits的值 - 1

#### zlend (11111111)

ziplist的结束符号