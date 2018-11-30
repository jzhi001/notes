# dump.rdb

## 格式

* 文件头 "REDIS0000"

* 数据库(REDIS_SELECTDB db)

* 数据(key => robj)

  * robj类型(如REDIS_STRING)

  * key长度

  * key内容

  * robj

    * string type robj

      * 长度

      * 值

    * list type robj

      * 元素个数

      * 所有元素的(长度和值)

* REDIS_EOF(表示rdb文件结束)

## save

```c
static int saveDb(char *filename);
```

每次save都会创建临时文件，按照格式将所有数据写入临时文件。最后用临时文件替换原本的rdb文件。  

### bgsave

```c
static int saveDbBackground(char *filename);
```

开启子进程执行saveDb()，并将server.bgsaveinprogress设为1。server在serverCron中检查子进程是否完成。