## Redis事务

### 1.相关命令

| 命令        | 格式                | 作用                                                         | 返回结果                                                     |
| ----------- | ------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **WATCH**   | WATCH key [key ...] | 将给出的Keys标记为监测态，作为事务执行的条件                 | always OK                                                    |
| **UNWATCH** | UNWATCH             | 清除事务中Keys的 监测态，如果调用了EXEC or DISCARD，则没有必要再手动调用UNWATCH | always OK                                                    |
| **MULTI**   | MULTI               | 显式开启redis事务，后续commands将排队，等候使用EXEC进行原子执行 | always OK                                                    |
| **EXEC**    | EXEC                | 执行事务中的commands队列，恢复连接状态。如果WATCH在之前被调用，只有监测中的Keys没有被修改，命令才会被执行，否则停止执行（详见下文，CAS机制） | 成功： 返回数组 —— 每个元素对应着原子事务中一个command的返回结果;失败： 返回NULL |
| **DISCARD** | DISCARD             | 清除事务中的commands队列，恢复连接状态。如果WATCH在之前被调用，释放监测中的Keys | always OK                                                    |

### 2.Redis事务

MULTI, EXEC, DISCARD and WATCH是Redis事务的基础。用来显式开启并控制一个事务，它们允许在一个步骤中执行一组命令。并保证：

- **事务中的所有命令都会被序列化并按顺序执行。**

- 在执行Redis事务的过程中，不会出现由另一个客户端发出的请求。这保证**命令队列**作为一个单独的原子操作被执行。

- 队列中的命令要么全部被处理，要么全部被忽略。

同时，Redis使用AOF(append-only file)，使用一个额外的write操作将事务写入磁盘。如果发生宕机，进程奔溃等情况，可以使用redis-check-aof tool 修复append-only file，使服务正常启动，并恢复部分操作。

#### 用法

使用MULTI命令显式开启Redis事务。该命令总是以OK回应。此时用户可以发出多个命令，Redis不会执行这些命令，而是将它们排队。EXEC被调用后，所有的命令都会被执行。而调用DISCARD可以清除事务中的commands队列并退出事务。

以下示例以原子方式，递增键foo和bar。

```redis-cli
>MULTI
OK

>INCR foo
QUEUED

>INCR bar
QUEUED

>EXEC
1）（整数）1
2）（整数）1
```

从上面的命令执行中可以看出，EXEC返回一个数组，其中每个元素都是事务中单个命令的返回结果，而且顺序与命令的发出顺序相同。

当Redis连接处于MULTI请求的上下文中时，所有命令将以字符串QUEUED（从Redis协议的角度作为状态回复发送）作为回复，并在命令队列中排队。只有EXEC被调用时，排队的命令才会被执行，此时才会有真正的返回结果。

### 3. 事务中的错误

事务期间，可能会遇到两种命令错误：

#### 在调用EXEC命令之前出现错误（COMMAND排队失败）

例如，命令可能存在语法错误（参数数量错误，错误的命令名称...）；

或者可能存在某些关键条件，如内存不足的情况（如果服务器使用maxmemory指令做了内存限制）。

客户端会在EXEC调用之前检测第一种错误。 通过检查排队命令的状态回复（***注意：这里是指排队的状态回复，而不是执行结果***），如果命令使用QUEUED进行响应，则它已正确排队，否则Redis将返回错误。如果排队命令时发生错误，大多数客户端将中止该事务并清除命令队列。然而：

在Redis 2.6.5之前，这种情况下，在EXEC命令调用后，客户端会执行命令的子集（成功排队的命令）而忽略之前的错误。

从Redis 2.6.5开始，服务端会记住在累积命令期间发生的错误，当EXEC命令调用时，将拒绝执行事务，并返回这些错误，同时自动清除命令队列。

示例如下：

```redis-cli
>MULTI
OK

>INCR a b c
ERR wrong number of arguments for 'incr' command
```

这是由于INCR命令的语法错误，将在调用EXEC之前被检测出来，并终止事务（version2.6.5+）。

#### 在调用EXEC命令之后出现错误

例如，使用错误的值对某个key执行操作（如针对String值调用List操作）

EXEC命令执行之后发生的错误并不会被特殊对待：即使事务中的某些命令执行失败，其他命令仍会被正常执行。

示例如下：

```
>MULTI
OK

>SET a 3
QUEUED

>LPOP a
QUEUED

>EXEC
OK
ERR Operation against a key holding the wrong kind of value
```

EXEC返回一个包含两个元素的字符串数组，一个元素是OK，另一个是-ERR……。

能否将错误合理的反馈给用户这取决于客户端library(如：Spring-data-redis.redisTemplate)的自身实现。

**需要注意的是，即使命令失败，队列中的所有其他命令也会被处理----Redis不会停止命令的处理**

### 4.Redis事务不支持Rollback

事实上Redis命令在事务执行时可能会失败，但仍会继续执行剩余命令而不是Rollback（事务回滚）。

Redis命令可能会执行失败，仅仅是由于错误的语法被调用（命令排队时检测不出来的错误），或者使用错误的数据类型操作某个Key： 这意味着，实际上失败的命令都是编程错误造成的，都是开发中能够被检测出来的，生产环境中不应该存在。由于不必支持Rollback,Redis内部简洁并且更加高效。

### 5.清除命令队列

DISCARD被用来中止事务。事务中的所有命令将不会被执行,连接将恢复正常状态。

```redis-cli
>SET foo 1
OK

>MULTI
OK

>INCR foo
QUEUED

>DISCARD
OK

> GET foo
"1"
```

