# Redis Beta1

[源码地址](https://code.google.com/archive/p/redis/downloads?page=7)

## [Redis协议](./protocol.md)

## [Redis配置文件(redis.conf)](./redis-conf.md)

* [server如何加载配置文件](./redis-conf.md#加载配置文件)

* [timeout](./redis-conf.md#timeout-N)

* [save](./redis-conf.md#save-seconds-changes)

* [dir](./redis-conf.md#dir-path)

* [loglevel](./redis-conf.md#loglevel-level)

* [logfile](./redis-conf.md#logfile-file)

* [databases](./redis-conf.md#databases-num)

## Redis数据类型(Redis Object)

* String

* List

## Redis数据结构

* event loop

* [client](./client.md#数据结构)

* [server](./server.md#server-数据结构)

* Redis Object 实现

  * [robj](./robj.md)

  * [String type robj](./robj.md#String-type-robj)

  * [List type robj](./robj.md#List-type-robj)

* [Redis command](./redis-command.md)

## Redis服务器运行流程

* [初始化配置相关字段](#initServerConfig)

* 初始化其他字段

* 注册主循环事件serverCron

* 加载配置文件

* 加载dump.rdb

* 注册文件事件acceptHandler(接受并创建client)

  * 接受客户端连接

  * 创建redisClient

    * 初始化变量

    * 注册文件事件readQueryFromClient(从客户端读取命令并在命令完整时执行)

    * 命令执行函数中注册文件事件sendReplyToClient(将结果写入客户端)

* 开始事件循环

## [dump.rdb](./rdb.md#格式)

## [save](./rdb.md#save) & [bgsave](./rdb.md#bgsave)

## Redis命令及实现

TODO

## Redis modules

* [sds](../sds.md)

* [adlist](../adlist.md)

* dict

* [ae(eventLoop)](../ae.md)

* anet