# Write Operations in Redis

Adding key-value to a Redis database is also called a write operation.  
Write operations perform differently on dealing with key's expiration.

## Replace

First, you set a key with 100 seconds expiration.
```Redis
you$ SET foo bar EX 100
=> OK
```
Then, you take a phone call and get back one minute later.  
Now the key `foo` has 40 seconds to live, and you assign a new value to it:
```Redis
you$ SET foo oof
=> OK

you$ TTL foo
=> (int)-1
```
As you noticed, expiration of `foo` is removed after a new value is assigned.  
This kind of write operation is called **Replace**, which refreshes expiration.  

## Overwrite

Then, you want to see if other commands are also replace operations:
```Redis
you$ SET counter 0 EX 100
=> OK

you$ INCR foo
=> (int)1

you$ TTL foo
=> 98
```
Surprisingly, `INCR` command does not reset expiration, also does `LPUSH`, `LPOP`, `SADD`, `SETRANGE`...  
These commands perform **Overwrite** operations, which leave expiration untouched.

## Replace and Overwrite

Imagine you have a plate, and you have some rice on it.
You cooked the rice a hour ago so it will be aged in two days.
If you take a spoon of the rice, or add some sugar on it, does your moves change the aging time of the rice? No! That's what a overwrite operation means.  

Actually you hate rice and clear the plate. Then you plate it with just-bought pasta.
Should we say that the food in plate foo will be aged in two days (aging time of rice)? Of cause not, it should be aging time of the pasta! And a replace operation works the same way.

```Redis
you$ SET plate rice EX 172800
=> OK

you$ APPEND plate sugar
=> (int)9

you$ TTL plate
=> (int) 172000

you$ SET plate pasta
=> OK

you$ TTL plate
=> (int)-1
```  

If you want to see how author of Redis explains this, let me quote from description of [EXPIRE](https://redis.io/commands/expire) command:

> The timeout will only be cleared by commands that delete or overwrite the contents of the key, including DEL, SET, GETSET and all the *STORE commands. This means that all the operations that conceptually alter the value stored at the key without replacing it with a new one will leave the timeout untouched. For instance, incrementing the value of a key with INCR, pushing a new value into a list with LPUSH, or altering the field value of a hash with HSET are all operations that will leave the timeout untouched.  
