# Redis协议(beta1)

## Networking layer

* 客户端使用TCP协议与服务器建立连接

* 服务器的端口号为6379

* 服务器与客户端交换的数据必须以"\r\n"(CRLF)结尾

## INLINE commands

inline commands是最简单的命令。这里给出一个例子：

```redis
C: PING
S: +PONG
```

inline command是客户端发送的以CRLF结尾的字符串。服务器一般以一个数字或一个返回码回复。  

如果服务器回复的是返回码，可以从返回码的第一个字符判断命令是否运行成功：

* "+"  成功

* "-"  失败

下面是服务器返回数字的例子：

```redis
C: EXISTS somekey
S: 0
```

由于'somekey'并不存在，服务器返回'0'

## BULK commands

bulk command和inline command相似，但是最后一个参数必须是一个**字节流**。"SET"命令是一个bulk command，看一下它的格式：

```redis
C: SET mykey 6
C: foobar
S: +OK
```

SET命令的最后一个参数是'6'，表示后面的字节流长度。注意字节流也是以CRLF结尾。SET命令的字符串版本如下：

```c
"SET mykey 6\r\nfoobar\r\n"
```

## BULK replies

服务器可能用bulk reply回复incline command和bulk command：

```redis
C: GET mykey
S: 6
S: foobar
```

bulk reply和bulk command相似。第一行的数字表示内容长度，第二行是实际内容。两行回复同样是以CRLF结尾的。字符串版本如下：

```c
"6\r\nfoobar\r\n"
```

如果数据库中并没有"mykey"，服务器会回复一个特殊值"nil":

```redis
C: GET mykey
S: nil
```

不同语言的Redis API不能回复空字符串，而是nil对象。如Ruby的API应该回复nil，C的API应该回复NULL。

## Bulk reply error reporting

bulk reply可能会是错误信息，如使用"GET"命令获取list对象是不合法的。这种情况下bulk reply的第一行会是一个负数表示错误信息长度，第二行是错误信息：

```redis
S: GET alistkey
S: -38
S: -ERR Requested element is not a string
```

## Multi-Bulk replies

如果服务器需要返回多个值，它会先返回bulk reply的数量，然后返回每一个bulk reply:

```redis
C: LRANGE mylist 0 3
S: 4
S: 3
S: foo
S: 3
S: bar
S: 5
S: Hello
S: 5
S: World
```

"4\r\n"表示一共有4个bulk reply。  

如果数据库中并没有"mylist"，服务器会回复一个特殊值"nil":

```redis
C: LRANGE mylist 0 1
S: nil
```

不同语言的Redis API不能回复空字符串，而是nil对象。如Ruby的API应该回复nil，C的API应该回复NULL。

## Multi-Bulk replies errors

同[Bulk reply error reporting](#Bulk-reply-error-reporting)

## Multiple commands and pipelining

客户端可以通过pipelining一次发送多个命令，所有的命令结果会被一起回复。  

注意： **当前版本没有实现pipelining**