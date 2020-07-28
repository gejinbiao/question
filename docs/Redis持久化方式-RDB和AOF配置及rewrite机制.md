# Redis 持久化方式 - RDB 和 AOF 配置及 rewrite 机制
redis 的核心配置配置是 redis.conf ,本文是基于 redis-4.0.6 版本讲解。

# RDB 数据持久化配置
默认情况下，redis 中的 RDB 数据持久化是开启的。 在 redis.conf 有如下一段默认配置：

	save 900 1
	save 300 10
	save 60 10000
	可自行定义（不推荐更改），格式如下：
	save <seconds<changes>

## 配置说明：

比如 save 60 10000 表示如果 60s 内，有 10000 个 key 发生了改变，就保存一次快照，并且每次生成一个新的快照，都会覆盖之前的老快照。

**补充：**

1. 通过 redis-cli SHUTDOWN 命令去停掉redis，redis 在退出的时候会将内存中的数据立即生成一份完整的 rdb 快照
2. 可以手动调用 save 或者 bgsave 命令，同步或异步执行 rdb 快照生成
3. 把上面三行配置注释或者删除掉，就可以关闭 RBD 持久化.

# AOF 数据持久化配置
AOF持久化，默认是关闭的。 打开 redis.conf, 找到如下配置
	appendonly no # 改为 yes 就开启了 AOF
	
	appendfilename "appendonly.aof" # aof 文件名
	
	# appendfsync always
	appendfsync everysec # 默认
	# appendfsync no

开启 AOF 持久化之后，redis 每次接收到一条写命令，会先写入操作系统 cache 中，然后每隔一定时间再 fsync 一下，这就对应了上面的 appendfsync 配置

> fsync 三种策略

	appendfsync always ：每次写入一条数据就执行一次fsync; 
	appendfsync everysec：每隔一秒执行一次fsync; 
	appendfsync no ：不主动执行fsync， 有操作系统自己决定

# 再谈 AOF 的 rewrite 机制

redis 中有缓存淘汰策略，因此会出现数据自动过期，另外也可能会被用户主动删除，导致 redis 中数据变少了。

但是，虽然数据被删除了，其对应的写日志还在 AOF 日志文件中，并且因为 AOF 日志文件就一个，因此这个文件会不断变大～～

所以才有了 rewrite 来解决这个问题，AOF 会自动在后台每隔一定时间做 rewrite 操作。

比如日志里存放了 100w 条数据的写日志，而 redis 内存中的数据只有 10 w。

AOF 会基于当前内存中的 10w 条数据构建一个新的日志文件，然后覆盖之前的旧的日志文件。
**补充：**

1. redis 2.4 之后，会自动进行 rewrite 操作
2. 在redis.conf中，可以配置 rewrite 策略


    	 auto-aof-rewrite-percentage 100 
    	 auto-aof-rewrite-min-size 64mb
    	  # 举例说明上述配置的意义
    	 比如说上一次 rewrite 之后，日志文件大小是 128 MB
    	 如果发现日志文件增长的比例超过了100%（对应第一条配置），比如日志文件变为 300MB
    	 然后就去跟 64 MB（对应第二条配置）做比较，如果大于 64 MB，才会去触发rewrite

3. 如果 redis 在写入 AOF 文件时，机器宕机可能会导致 AOF 文件破损
，这时可以用 redis-check-aof --fix 命令来修复破损的 AOF 文件

> rewrite 流程

1. redis fork一个子进程
1. 子进程基于当前内存中的数据，开始往一个新的 临时的AOF文件 中写入日志
1. redis 主进程接收到客户端新的写操作之后，会将新的日志写入内存，同时写入到旧的AOF文件
1. 子进程写完新的日志文件之后，redis 主进程将内存中的新日志再次追加到新的 AOF 文件中
1. 用新的日志文件替换掉旧的日志文件

**其他**

1. AOF 和 RDB 都开启的时候，redis重启的时候，优先通过 AOF 进行数据恢复，因为数据比较完整
1. RDB 在生成快照时，redis 不会执行 AOF rewrite

**总结**
实践是检验真理的唯一标准，只有动手实践，才能真正体会 redis 是如何做数据恢复的，另外也可以用记事本打开 rdb 快照文件和 aof 日志文件，去看看里面存的是什么东西。
