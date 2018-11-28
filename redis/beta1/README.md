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

* client

* server

* Redis Object 实现

  * [robj](./robj.md)

  * [String type robj](./robj.md#String-type-robj)

  * [List type robj](./robj.md#List-type-robj)

* Redis command

## Redis运行流程

TODO

## save & bgsave

TODO

## dump.rdb

TODO

## Redis命令及实现

TODO

## Redis modules

* [sds](../sds.md)

* adlist

* dict

* [ae(eventLoop)](../ae.md)

* anet