---
title: redis-eval
date: 2018-08-27 11:17:50
tags: redis,lua
categories: redis
---

# EVAL

**EVAL script numkeys key [key ...] arg [arg ...]**

从 Redis 2.6.0 版本开始，通过内置的 Lua 解释器，可以使用 [EVAL](http://doc.redisfans.com/script/eval.html#eval) 命令对 Lua 脚本进行求值。

- `script` 参数是一段 Lua 5.1 脚本程序，它会被运行在 Redis 服务器上下文中，这段脚本不必(也不应该)定义为一个 Lua 函数。
- `numkeys` 参数用于指定键名参数的个数。
- 键名参数 `key [key ...]` 从 [EVAL](http://doc.redisfans.com/script/eval.html#eval) 的第三个参数开始算起，表示在脚本中所用到的那些 Redis 键(key)，这些键名参数可以在 Lua 中通过全局变量 `KEYS` 数组，用 `1` 为基址的形式访问( `KEYS[1]` ， `KEYS[2]` ，以此类推)。
- 在命令的最后，那些不是键名参数的附加参数 `arg [arg ...]` ，可以在 Lua 中通过全局变量 `ARGV` 数组访问，访问的形式和 `KEYS` 变量类似( `ARGV[1]` 、 `ARGV[2]` ，诸如此类)。

```lua
> eval "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second
1) "key1"
2) "key2"
3) "first"
4) "second"
```

通过docker 安装reids

```dockerfile
docker run --name redis-lua -p 6379:6379 -d redis 
```

> ~ docker ps
>
> CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
> 939cd668f9d0        redis               "docker-entrypoint.s…"   6 seconds ago       Up 2 seconds        0.0.0.0:6379->6379/tcp   redis-lua

执行以下命令进入docker容器内部：
```shell
docker exec -it 939cd668f9d0 /bin/bash
```

## 执行eval命令

在 Lua 脚本中，可以使用两个不同函数来执行 Redis 命令，它们分别是：

- `redis.call()`
- `redis.pcall()`

这两个函数的唯一区别在于它们使用不同的方式处理执行命令所产生的错误。

> 当 `redis.call()` 在执行命令的过程中发生错误时，脚本会停止执行，并返回一个脚本错误，错误的输出信息会说明错误造成的原因：
>
> ```
> redis> lpush foo a
> (integer) 1
> 
> redis> eval "return redis.call('get', 'foo')" 0
> (error) ERR Error running script (call to f_282297a0228f48cd3fc6a55de6316f31422f5d17): ERR Operation against a key holding the wrong kind of value
> ```
>
> 和 `redis.call()` 不同， `redis.pcall()` 出错时并不引发(raise)错误，而是返回一个带 `err` 域的 Lua 表(table)，用于表示错误：
>
> ```
> redis 127.0.0.1:6379> EVAL "return redis.pcall('get', 'foo')" 0
> (error) ERR Operation against a key holding the wrong kind of value
> ```

`redis.call()` 和 `redis.pcall()` 两个函数的参数可以是任何格式良好(well formed)的 Redis 命令：

```
> eval "return redis.call('set','foo','bar')" 0
OK
```

需要注意的是，上面这段脚本的确实现了将键 `foo` 的值设为 `bar` 的目的，但是，它违反了 [EVAL](http://doc.redisfans.com/script/eval.html#eval) 命令的语义，因为脚本里使用的所有键都应该由 `KEYS` 数组来传递，就像这样：

```
> eval "return redis.call('set',KEYS[1],'bar')" 1 foo
OK
```

## golang中使用Lua脚本操作Redis

为了在我的一个基本库中降低与Redis的通讯成本，我将一系列操作封装到LUA脚本中，借助Redis提供的EVAL命令来简化操作。
 EVAL能够提供的特性：

1. 可以在LUA脚本中封装若干操作，如果有多条Redis指令，封装好之后只需向Redis一次性发送所有参数即可获得结果
2. Redis可以保证Lua脚本运行期间不会有其他命令插入执行，提供像数据库事务一样的原子性
3. Redis会根据脚本的SHA值缓存脚本，已经缓存过的脚本不需要再次传输Lua代码，减少了通信成本，此外在自己代码中改变Lua脚本，执行时Redis必定也会使用最新的代码。

导入常见的Go库如   "github.com/go-redis/redis"，就可以实现以下代码。

**生成一段Lua脚本**

```go
// KEYS: key for record
// ARGV: fieldName, currentUnixTimestamp, recordTTL
// Update expire field of record key to current timestamp, and renew key expiration
var updateRecordExpireScript = redis.NewScript(`
redis.call("EXPIRE", KEYS[1], ARGV[3])
redis.call("HSET", KEYS[1], ARGV[1], ARGV[2])
return 1
`)
```

 该变量创建时，Lua代码不会被执行，也不需要有已存的Redis连接。 Redis提供的Lua脚本支持，默认有KEYS、ARGV两个数组，KEYS代表脚本运行时传入的若干键值，ARGV代表传入的若干参数。由于Lua代码需要保持简洁，难免难以读懂，最好为这些参数写一些注释

注意：上面一段代码使用``跨行，`所在的行虽然空白回车，也会被认为是一行，报错时不要看错代码行号。

**运行一段Lua脚本**

````go
updateRecordExpireScript.Run(c.Client, []string{recordKey(key)}, 
                                    expireField,
                                    time.Now().UTC().UnixNano(), int64(c.opt.RecordTTL/time.Second)).Err()
````

 运行时，Run将会先通过EVALSHA尝试通过缓存运行脚本。如果没有缓存，则使用EVAL运行，这时Lua脚本才会被整个传入Redis。

## Lua脚本的限制

1. Redis不提供引入额外的包，例如os等，只有redis这一个包可用。
2. Lua脚本将会在一个函数中运行，所有变量必须使用local声明
3. return返回多个值时，Redis将会只给你第一个

## 脚本中的类型限制

1. 脚本返回nil时，Go中得到的是err = redis.Nil（与Get找不到值相同）
2. 脚本返回false时，Go中得到的是nil，脚本返回true时，Go中得到的是int64类型的1
3. 脚本返回{"ok": ...}时，Go中得到的是redis的status类型（true/false)
4. 脚本返回{"err": ...}时，Go中得到的是err值，也可以通过return redis.error_reply("My Error")达成
5. 脚本返回number类型时，Go中得到的是int64类型
6. 传入脚本的KEYS/ARGV中的值一律为string类型，要转换为数字类型应当使用to_number

 **如果脚本运行了很久会发生什么？**

Lua脚本运行期间，为了避免被其他操作污染数据，这期间将不能执行其它命令，一直等到执行完毕才可以继续执行其它请求。当Lua脚本执行时间超过了lua-time-limit时，其他请求将会收到Busy错误，除非这些请求是SCRIPT KILL（杀掉脚本）或者SHUTDOWN NOSAVE（不保存结果直接关闭Redis）

 

# 参考：

 [Go中通过Lua脚本操作Redis](https://www.jianshu.com/p/fec18f59ff8f)

[redis命令参考](http://doc.redisfans.com/script/eval.html)

 

 

 

 

 

  

 

 

 

 

 

 

 