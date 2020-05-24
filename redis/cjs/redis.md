# NoSql-Redis

## NO-SQL和SQL的对比

> 在大数据时代中,经历了数据库分库,读写分离,之后在老的SQL面临扩展瓶颈的情况下,出现了NOSQL

- NoSQL(NoSQL = Not Only SQL )，意即“不仅仅是SQL”，
    泛指非关系型的数据库。随着互联网web2.0网站的兴起，传统的关系数据库在应付web2.0网站，特别是超大规模和高并发的SNS类型的web2.0纯动态网站已经显得力不从心，暴露了很多难以克服的问题，而非关系型的数据库则由于其本身的特点得到了非常迅速的发展。NoSQL数据库的产生就是为了解决大规模数据集合多重数据种类带来的挑战，尤其是大数据应用难题，包括超大规模数据的存储。（例如谷歌或Facebook每天为他们的用户收集万亿比特的数据）。这些类型的数据存储不需要固定的模式，无需多余操作就可以横向扩展。

- NoSQL数据库种类繁多，但是一个共同的特点都是去掉关系数据库的关系型特性。
    数据之间无关系，这样就非常容易扩展。也无形之间，在架构的层面上带来了可扩展的能力。

- NoSQL无需事先为要存储的数据建立字段，随时可以存储自定义的数据格式。而在关系数据库里，
    增删字段是一件非常麻烦的事情。如果是非常大数据量的表，增加字段简直就是一个噩梦

- RDBMS vs NoSQL

    - RDBMS
        - 高度组织化结构化数据
        - 结构化查询语言（SQL）
        - 数据和关系都存储在单独的表中。
        - 数据操纵语言，数据定义语言
        - 严格的一致性
        - 基础事务
    - NoSQL
        - 代表着不仅仅是SQL
        - 没有声明性查询语言
        - 没有预定义的模式
          -键 - 值对存储，列存储，文档存储，图形数据库
        - 最终一致性，而非ACID属性
        - 非结构化和不可预知的数据
        - CAP定理
        - 高性能，高可用性和可伸缩性

    **ACID和CAP**

    - ACID: 

        - **事务的原子性(Atomicity)：**是指一个事务要么全部执行，要么不执行，也就是说一个事务不可能只执行了一半就停止了。比如你从取款机取钱，这个事务可以分成两个步骤：1划卡，2出钱。不可能划了卡，而钱却没出来。这两步必须同时完成，要么就不完成。
        - [**事务**](https://baike.baidu.com/item/事务)**的[一致性](https://baike.baidu.com/item/一致性)(Consistency)：**是指事务的运行并不改变数据库中数据的一致性。例如，完整性约束了a+b=10，一个事务改变了a，那么b也应该随之改变。

        - **独立性(Isolation）：**事务的独立性也有称作隔离性，是指两个以上的事务不会出现交错执行的状态。因为这样可能会导致数据不一致。

        - **持久性(Durability）：**[事务](https://baike.baidu.com/item/事务)的[持久性](https://baike.baidu.com/item/持久性)是指事务执行成功以后，该事务对数据库所作的更改便是持久的保存在数据库之中，不会无缘无故的[回滚](https://baike.baidu.com/item/回滚)。

    - CAP

        - 一致性（C）：在[分布式系统](https://baike.baidu.com/item/分布式系统/4905336)中的所有数据备份，在同一时刻是否同样的值。（等同于所有节点访问同一份最新的数据副本）
        - 可用性（A）：在集群中一部分节点故障后，[集群](https://baike.baidu.com/item/集群/5486962)整体是否还能响应[客户端](https://baike.baidu.com/item/客户端/101081)的读写请求。（对数据更新具备高可用性）
        - 分区容忍性（P）：以实际效果而言，分区相当于对通信的时限要求。系统如果不能在时限内达成数据一致性，就意味着发生了分区的情况，必须就当前操作在C和A之间做出选择。
        - CAP原则的精髓就是要么AP，要么CP，要么AC，但是不存在CAP。如果在某个分布式系统中数据无副本， 那么系统必然满足强一致性条件， 因为只有独一数据，不会出现数据不一致的情况，此时C和P两要素具备，但是如果系统发生了网络分区状况或者宕机，必然导致某些数据不可以访问，此时可用性条件就不能被满足，即在此情况下获得了CP系统，但是CAP不可同时满足 [1] 。

## no_sql设计

> 使用Bson设计NoSql

### 聚合模型


![](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20191018150107.png)
- KV模型
- BSON
- 列族
- 图形

### CAP的3进2

#### P:分布式中必须要实现

> CAP理论就是说在分布式存储系统中，最多只能实现上面的两点。
> 而由于当前的网络硬件肯定会出现延迟丢包等问题，所以
>
>
> 分区容忍性是我们必须需要实现的。
>
>
> 所以我们只能在一致性和可用性之间进行权衡，没有NoSQL系统能同时保证这三点。
> =======================================================================================================================
> C:强一致性 A：高可用性 P：分布式容忍性
>  CA 传统Oracle数据库
>
>
>  AP 大多数网站架构的选择
>
>
>  CP Redis、Mongodb
>
>注意：分布式架构的时候必须做出取舍。
>  一致性和可用性之间取一个平衡。多余大多数web应用，其实并不需要强一致性。
> 
> 因此牺牲C换取P，这是目前分布式数据库产品的方向
> =======================================================================================================================
> 一致性与可用性的决择
>
>
> 对于web2.0网站来说，关系数据库的很多主要特性却往往无用武之地
>
>数据库事务一致性需求 
> 　　很多web实时系统并不要求严格的数据库事务，对读一致性的要求很低， 有些场合对写一致性要求并不高。允许实现最终一致性。
> 
>
>数据库的写实时性和读实时性需求
> 　　对关系数据库来说，插入一条数据之后立刻查询，是肯定可以读出来这条数据的，但是对于很多web应用来说，并不要求这么高的实时性，比方说发一条消息之 后，过几秒乃至十几秒之后，我的订阅者才看到这条动态是完全可以接受的。
> 
>对复杂的SQL查询，特别是多表关联查询的需求 
> 　　任何大数据量的web系统，都非常忌讳多个大表的关联查询，以及复杂的数据分析类型的报表查询，特别是SNS类型的网站，从需求以及产品设计角 度，就避免了这种情况的产生。往往更多的只是单表的主键查询，以及单表的简单条件分页查询，SQL的功能被极大的弱化了。

### Base原则

> BASE就是为了解决关系数据库强一致性引起的问题而引起的可用性降低而提出的解决方案。
>
> BASE其实是下面三个术语的缩写：
>     基本可用（Basically Available）
>     软状态（Soft state）
>     最终一致（Eventually consistent）
>
>
> 它的思想是通过让系统放松对某一时刻数据一致性的要求来换取系统整体伸缩性和性能上改观。为什么这么说呢，缘由就在于大型系统往往由于地域分布和极高性能的要求，不可能采用分布式事务来完成这些指标，要想获得这些指标，我们必须采用另外一种方式来完成，这里BASE就是解决这个问题的办法

## [Redis](http://redisdoc.com/)

> - 单进程
>     - 单进程模型来处理客户端的请求。对读写等事件的响应是通过对epoll函数的包装来做到的。Redis的实际处理速度完全依靠主进程的执行效率
>     - epoll是Linux内核为处理大批量文件描述符而作了改进的epoll，是Linux下多路复用IO接口select/poll的增强版本，它能显著提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率。
> - 数据库
>     - 默认16个数据库(0-15)，类似数组下表从零开始，初始默认使用零号库
>     - select命令切换数据库,select num[1-16]
>     - dbsize:查看当前数据库的key的数量
>     - flushdb:清空当前库
>     - Flushall:清空所有库

### 数据类型

#### string(字符串)

>
> string是redis最基本的类型，你可以理解成与Memcached一模一样的类型，一个key对应一个value。
>
>
> string类型是二进制安全的。意思是redis的string可以包含任何数据。比如jpg图片或者序列化的对象 。
>
>
> string类型是Redis最基本的数据类型，一个redis中字符串value最多可以是512M

#### hash(哈希)

> Redis hash 是一个键值对集合。
> Redis hash是一个string类型的field和value的映射表，hash特别适合用于存储对象。
>
>
> 类似Java里面的Map<String,Object>
>

#### list(列表)

> Redis 列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素导列表的头部（左边）或者尾部（右边）。
> 它的底层实际是个链表

#### set(集合)

> Redis的Set是string类型的无序集合。它是通过HashTable实现实现的，

#### zset(有序集合)

> zset(sorted set：有序集合)
> Redis zset 和 set 一样也是string类型元素的集合,且不允许重复的成员。
> 不同的是每个元素都会关联一个double类型的分数。
> redis正是通过分数来为集合中的成员进行从小到大的排序。zset的成员是唯一的,但分数(score)却可以重复。

### 操作命令

#### key

- keys * ：查看当前库的所有值
- exists [key]：查看这个key是否存在
- move [key] [db]：移动key到其他库中
- expire [key] [second]：设置某个key的生存周期
- ttl [key]：查看某个key还有多久过期
- type [key]：观察某个key的类型

## 配置文件

### units配置

> ![](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20191023091619.png)
>
> 配置单位,k,kb是这种单位是没有大小写区分的

### 包含其他配置

> include /path 包含其他目录的文件

### 网络配置

```bash
bind 0.0.0.0 # redis服务绑定的网卡地址

# Protected mode is a layer of security protection, in order to avoid that
# Redis instances left open on the internet are accessed and exploited.
#
# When protected mode is on and if:
#
# 1) The server is not binding explicitly to a set of addresses using the
#    "bind" directive.
# 2) No password is configured.
#
# The server only accepts connections from clients connecting from the
# IPv4 and IPv6 loopback addresses 127.0.0.1 and ::1, and from Unix domain
# sockets.
#
# By default protected mode is enabled. You should disable it only if
# you are sure you want clients from other hosts to connect to Redis
# even if no authentication is configured, nor a specific set of interfaces
# are explicitly listed using the "bind" directive.
protected-mode yes # 默认为yes,开启的话,只有bind的网卡或者设置密码才可以访问

# Accept connections on the specified port, default is 6379 (IANA #815344).
# If port 0 is specified Redis will not listen on a TCP socket.
port 6379 # 配置监听端口号

# TCP listen() backlog.
#
# In high requests-per-second environments you need an high backlog in order
# to avoid slow clients connections issues. Note that the Linux kernel
# will silently truncate it to the value of /proc/sys/net/core/somaxconn so
# make sure to raise both the value of somaxconn and tcp_max_syn_backlog
# in order to get the desired effect.
tcp-backlog 511	
#设置tcp的backlog，backlog其实是一个连接队列，backlog队列总和=未完成三次握手队列 + 已经完成三次握手队列。
#在高并发环境下你需要一个高backlog值来避免慢客户端连接问题。注意Linux内核会将这个值减小到/proc/sys/net/core/somaxconn的值，所以需要确认增大somaxconn和tcp_max_syn_backlog两个值来达到想要的效果
# Unix socket.
#
# Specify the path for the Unix socket that will be used to listen for
# incoming connections. There is no default, so Redis will not listen
# on a unix socket when not specified.
#
# unixsocket /tmp/redis.sock
# unixsocketperm 700

# Close the connection after a client is idle for N seconds (0 to disable)
timeout 0 # 超时时间
# TCP keepalive.
#
# If non-zero, use SO_KEEPALIVE to send TCP ACKs to clients in absence
# of communication. This is useful for two reasons:
#
# 1) Detect dead peers.
# 2) Take the connection alive from the point of view of network
#    equipment in the middle.
#
# On Linux, the specified value (in seconds) is the period used to send ACKs.
# Note that to close the connection the double of the time is needed.
# On other kernels the period depends on the kernel configuration.
#
# A reasonable value for this option is 300 seconds, which is the new
# Redis default starting with Redis 3.2.1.
tcp-keepalive 300 # 是否检测redis集群中的其他机器是否存活,默认300秒
```

### 日志以及服务器运行时信息

```bash
# By default Redis does not run as a daemon. Use 'yes' if you need it.
# Note that Redis will write a pid file in /var/run/redis.pid when daemonized.
daemonize yes #如果选择yes,则每次运行会将pid写入/var/run/redis.pid, 是yes时 redis会作为守护进程运行,并且将日志输出到logfile

# If you run Redis from upstart or systemd, Redis can interact with your
# supervision tree. Options:
#   supervised no      - no supervision interaction
#   supervised upstart - signal upstart by putting Redis into SIGSTOP mode
#- 通过将Redis置于SIGSTOP模式来启动信号
#   supervised systemd - signal systemd by writing READY=1 to $NOTIFY_SOCKET
# - signal systemd将READY = 1写入$ NOTIFY_SOCKET
#   supervised auto    - detect upstart or systemd method based on
#                        UPSTART_JOB or NOTIFY_SOCKET environment variables
# - 检测upstart或systemd方法基于 UPSTART_JOB或NOTIFY_SOCKET环境变量
# Note: these supervision methods only signal "process is ready."
#       They do not enable continuous liveness pings back to your supervisor.
supervised no #监督互动,默认没有

# If a pid file is specified, Redis writes it where specified at startup
# and removes it at exit.
# - 如果pidfile指定了,会在体统运行的时候写入redis运行的pid,并且在redis退出的时候删除它
#	
# When the server runs non daemonized, no pid file is created if none is
# specified in the configuration. When the server is daemonized, the pid file
# is used even if not specified, defaulting to "/var/run/redis.pid".
# - 如果daemonize选项设置成no,则不会生成这个文件,如果是yes,则是守护进程,即使没有设置pid文件位置,也会默认的创建/var/run/redis.pid

# Creating a pid file is best effort: if Redis is not able to create it
# nothing bad happens, the server will start and run normally.
pidfile /var/run/redis_6379.pid # 指定pid文件的生成位置

# Specify the server verbosity level. #指定服务器的日志级别
# This can be one of:
# debug (a lot of information, useful for development/testing) #debeg级别,建议开发,测试时使用
# verbose (many rarely useful info, but not a mess like the debug level)# 相对于debug,会稍微少点
# notice (moderately verbose, what you want in production probably)# 建议生产环境使用
# warning (only very important / critical messages are logged) # 仅仅非常重要的消息
loglevel notice #默认使用notice

# Specify the log file name. Also the empty string can be used to force
# Redis to log on the standard output. Note that if you use standard
# output for logging but daemonize, logs will be sent to /dev/null
logfile /var/log/redis/redis.log #指定日志的文件以及文件名,如果使用空字符串,则会打印到命令行,如果以守护进程启动,则会打印到/dev/null

# To enable logging to the system logger, just set 'syslog-enabled' to yes,
# and optionally update the other syslog parameters to suit your needs.
# syslog-enabled no # - 是否使用系统日志

# Specify the syslog identity.
# syslog-ident redis #- 使用系统日志的名称

# Specify the syslog facility. Must be USER or between LOCAL0-LOCAL7.
# syslog-facility local0 日志输出设备,Linux中有LOCAL0-7

# Set the number of databases. The default database is DB 0, you can select
# a different one on a per-connection basis using SELECT <dbid> where
# dbid is a number between 0 and 'databases'-1
databases 16 # -默认有多少个库,默认使用0,可以通过select <dbid>选择库,库id在 0- (16-1) 之间

# By default Redis shows an ASCII art logo only when started to log to the
# standard output and if the standard output is a TTY. Basically this means
# that normally a logo is displayed only in interactive sessions.
#
# However it is possible to force the pre-4.0 behavior and always show a
# ASCII art logo in startup logs by setting the following option to yes.
always-show-logo yes #默认情况下,只有在交互式页面才会显示日志输出,开启后则始终输出
```


###  持久化配置
```bash
#RDB配置
################################ SNAPSHOTTING  ################################
#
# Save the DB on disk:
#
#   save <seconds> <changes>
#
#   Will save the DB if both the given number of seconds and the given
#   number of write operations against the DB occurred.
#
#   In the example below the behaviour will be to save:
#   after 900 sec (15 min) if at least 1 key changed
#   after 300 sec (5 min) if at least 10 keys changed
#   after 60 sec if at least 10000 keys changed
#
#   Note: you can disable saving completely by commenting out all "save" lines.
#
#   It is also possible to remove all the previously configured save
#   points by adding a save directive with a single empty string argument
#   like in the following example:
#
#   save ""

save 900 1  # 每900秒 只要操作了一次,就会持久化
save 300 10 # 每300秒 操作10次会持久化
save 60 10000 #每60秒 操作10000次会持久化

# By default Redis will stop accepting writes if RDB snapshots are enabled
# (at least one save point) and the latest background save failed.
# This will make the user aware (in a hard way) that data is not persisting
# on disk properly, otherwise chances are that no one will notice and some
# disaster will happen.
#
# If the background saving process will start working again Redis will
# automatically allow writes again.
#
# However if you have setup your proper monitoring of the Redis server
# and persistence, you may want to disable this feature so that Redis will
# continue to work as usual even if there are problems with disk,
# permissions, and so forth.
stop-writes-on-bgsave-error yes

# Compress string objects using LZF when dump .rdb databases?
# For default that's set to 'yes' as it's almost always a win.
# If you want to save some CPU in the saving child set it to 'no' but
# the dataset will likely be bigger if you have compressible values or keys.
rdbcompression yes

# Since version 5 of RDB a CRC64 checksum is placed at the end of the file.
# This makes the format more resistant to corruption but there is a performance
# hit to pay (around 10%) when saving and loading RDB files, so you can disable it
# for maximum performances.
#
# RDB files created with checksum disabled have a checksum of zero that will
# tell the loading code to skip the check.
rdbchecksum yes 

# The filename where to dump the DB
dbfilename dump.rdb #rdb持久化的名称

# The working directory.
#
# The DB will be written inside this directory, with the filename specified
# above using the 'dbfilename' configuration directive.
#
# The Append Only File will also be created inside this directory.
#
# Note that you must specify a directory here, not a file name.
dir /var/lib/redis #dir目录,以及rdb存放的位置

#AOF配置
############################## APPEND ONLY MODE ###############################

# By default Redis asynchronously dumps the dataset on disk. This mode is
# good enough in many applications, but an issue with the Redis process or
# a power outage may result into a few minutes of writes lost (depending on
# the configured save points).
#
# The Append Only File is an alternative persistence mode that provides
# much better durability. For instance using the default data fsync policy
# (see later in the config file) Redis can lose just one second of writes in a
# dramatic event like a server power outage, or a single write if something
# wrong with the Redis process itself happens, but the operating system is
# still running correctly.
#
# AOF and RDB persistence can be enabled at the same time without problems.
# If the AOF is enabled on startup Redis will load the AOF, that is the file
# with the better durability guarantees.
#
# Please check http://redis.io/topics/persistence for more information.

appendonly no #是否开启AOF

# The name of the append only file (default: "appendonly.aof")

appendfilename "appendonly.aof" #文件名称
# The fsync() call tells the Operating System to actually write data on disk
# instead of waiting for more data in the output buffer. Some OS will really flush
# data on disk, some other OS will just try to do it ASAP.
#
# Redis supports three different modes:
#
# no: don't fsync, just let the OS flush the data when it wants. Faster.
# always: fsync after every write to the append only log. Slow, Safest.
# everysec: fsync only one time every second. Compromise.
#
# The default is "everysec", as that's usually the right compromise between
# speed and data safety. It's up to you to understand if you can relax this to
# "no" that will let the operating system flush the output buffer when
# it wants, for better performances (but if you can live with the idea of
# some data loss consider the default persistence mode that's snapshotting),
# or on the contrary, use "always" that's very slow but a bit safer than
# everysec.
#
# More details please check the following article:
# http://antirez.com/post/redis-persistence-demystified.html
#
# If unsure, use "everysec".

# appendfsync always 
# Always：同步持久化 每次发生数据变更会被立即记录到磁盘  性能较差但数据完整性比较好
appendfsync everysec# 默认使用everysec
# everysec:出厂默认推荐，异步操作，每秒记录   如果一秒内宕机，有数据丢失
# appendfsync no

# When the AOF fsync policy is set to always or everysec, and a background
# saving process (a background save or AOF log background rewriting) is
# performing a lot of I/O against the disk, in some Linux configurations
# Redis may block too long on the fsync() call. Note that there is no fix for
# this currently, as even performing fsync in a different thread will block
# our synchronous write(2) call.
#
# In order to mitigate this problem it's possible to use the following option
# that will prevent fsync() from being called in the main process while a
# BGSAVE or BGREWRITEAOF is in progress.
#
# This means that while another child is saving, the durability of Redis is
# the same as "appendfsync none". In practical terms, this means that it is
# possible to lose up to 30 seconds of log in the worst scenario (with the
# default Linux settings).
#
# If you have latency problems turn this to "yes". Otherwise leave it as
# "no" that is the safest pick from the point of view of durability.

no-appendfsync-on-rewrite no #默认是no,aof每次写入磁盘都会等待磁盘同步,可能会存在竞争,当这个选项设置成yes的时候,aof每次写入只会写入磁盘缓冲区,并不一定写入磁盘,虽然效率很高,但是可能会丢掉30秒左右的数据,直接选no即可
#因此，如果应用系统无法忍受延迟，而可以容忍少量的数据丢失，则设置为yes。如果应用系统无法忍受数据丢失，则设置为no。
# Automatic rewrite of the append only file.
# Redis is able to automatically rewrite the log file implicitly calling
# BGREWRITEAOF when the AOF log size grows by the specified percentage.
#
# This is how it works: Redis remembers the size of the AOF file after the
# latest rewrite (if no rewrite has happened since the restart, the size of
# the AOF at startup is used).
#
# This base size is compared to the current size. If the current size is
# bigger than the specified percentage, the rewrite is triggered. Also
# you need to specify a minimal size for the AOF file to be rewritten, this
# is useful to avoid rewriting the AOF file even if the percentage increase
# is reached but it is still pretty small.
#
# Specify a percentage of zero in order to disable the automatic AOF
# rewrite feature.

auto-aof-rewrite-percentage 100 #自动重写的大小的百分比,百分比基于上次rewrite的百分比且文件大于auto-aof-rewrite-min-size值执行重写
auto-aof-rewrite-min-size 64mb #设置最小的文件重写大小

# An AOF file may be found to be truncated at the end during the Redis
# startup process, when the AOF data gets loaded back into memory.
# This may happen when the system where Redis is running
# crashes, especially when an ext4 filesystem is mounted without the
# data=ordered option (however this can't happen when Redis itself
# crashes or aborts but the operating system still works correctly).
#
# Redis can either exit with an error when this happens, or load as much
# data as possible (the default now) and start if the AOF file is found
# to be truncated at the end. The following option controls this behavior.
#
# If aof-load-truncated is set to yes, a truncated AOF file is loaded and
# the Redis server starts emitting a log to inform the user of the event.
# Otherwise if the option is set to no, the server aborts with an error
# and refuses to start. When the option is set to no, the user requires
# to fix the AOF file using the "redis-check-aof" utility before to restart
# the server.
#
# Note that if the AOF file will be found to be corrupted in the middle
# the server will still exit with an error. This option only applies when
# Redis will try to read more data from the AOF file but not enough bytes
# will be found.
aof-load-truncated yes

# When rewriting the AOF file, Redis is able to use an RDB preamble in the
# AOF file for faster rewrites and recoveries. When this option is turned
# on the rewritten AOF file is composed of two different stanzas:
#
#   [RDB file][AOF tail]
#
# When loading Redis recognizes that the AOF file starts with the "REDIS"
# string and loads the prefixed RDB file, and continues loading the AOF
# tail.
aof-use-rdb-preamble yes
```

### 安全

```bash
################################## SECURITY ###################################

# Require clients to issue AUTH <PASSWORD> before processing any other
# commands.  This might be useful in environments in which you do not trust
# others with access to the host running redis-server.
# 在键入其他命令之前 键入 auth <password>
# This should stay commented out for backward compatibility and because most
# people do not need auth (e.g. they run their own servers).
#
# Warning: since Redis is pretty fast an outside user can try up to
# 150k passwords per second against a good box. This means that you should
# use a very strong password otherwise it will be very easy to break.
#
# requirepass foobared
requirepass P@ssw0rdRedis123 #设置密码
# 也可以通过 config set requirepass <password>来设置密码 如果设置的是""空字符串 则表示没有密码
# Command renaming.
#
# It is possible to change the name of dangerous commands in a shared
# environment. For instance the CONFIG command may be renamed into something
# hard to guess so that it will still be available for internal-use tools
# but not available for general clients.
#
# Example:
#
# rename-command CONFIG b840fc02d524045429941cc15f59e41cb7be6c52
#
# It is also possible to completely kill a command by renaming it into
# an empty string:
#
# rename-command CONFIG ""
#
# Please note that changing the name of commands that are logged into the
# AOF file or transmitted to slaves may cause problems.

```

### 限制

#### 最大连接数

```bash
################################### CLIENTS ####################################

# Set the max number of connected clients at the same time. By default
# this limit is set to 10000 clients, however if the Redis server is not
# able to configure the process file limit to allow for the specified limit
# the max number of allowed clients is set to the current file limit
# minus 32 (as Redis reserves a few file descriptors for internal uses).
#
# Once the limit is reached Redis will close all the new connections sending
# an error 'max number of clients reached'.
#
# maxclients 10000  #用于设置最大链接客户端数量,当用户达到这个数量的时候,新用户则收到max number of clients reached,不设置时默认是10000

```

#### 内存管理

```bash
############################## MEMORY MANAGEMENT ################################

# Set a memory usage limit to the specified amount of bytes.
# When the memory limit is reached Redis will try to remove keys
# according to the eviction policy selected (see maxmemory-policy).
#
# If Redis can't remove keys according to the policy, or if the policy is
# set to 'noeviction', Redis will start to reply with errors to commands
# that would use more memory, like SET, LPUSH, and so on, and will continue
# to reply to read-only commands like GET.
#
# This option is usually useful when using Redis as an LRU or LFU cache, or to
# set a hard memory limit for an instance (using the 'noeviction' policy).
#
# WARNING: If you have slaves attached to an instance with maxmemory on,
# the size of the output buffers needed to feed the slaves are subtracted
# from the used memory count, so that network problems / resyncs will
# not trigger a loop where keys are evicted, and in turn the output
# buffer of slaves is full with DELs of keys evicted triggering the deletion
# of more keys, and so forth until the database is completely emptied.
#
# In short... if you have slaves attached it is suggested that you set a lower
# limit for maxmemory so that there is some free RAM on the system for slave
# output buffers (but this is not needed if the policy is 'noeviction').
#
# maxmemory <bytes> 
#设置redis可以使用的内存量。一旦到达内存使用上限，redis将会试图移除内部数据，移除规则可以通过maxmemory-policy来指定。如果redis无法根据移除规则来移除内存中的数据，或者设置了“不允许移除”，
#那么redis则会针对那些需要申请内存的指令返回错误信息，比如SET、LPUSH等。
#但是对于无内存申请的指令，仍然会正常响应，比如GET等。如果你的redis是主redis（说明你的redis有从redis），那么在设置内存使用上限时，需要在系统中留出一些内存空间给同步队列缓存，只有在你设置的是“不移除”的情况下，才不用考虑这个因素
# MAXMEMORY POLICY: how Redis will select what to remove when maxmemory
# is reached. You can select among five behaviors:
# 用于设置内存管理策略,当redis使用的内存达到maxmemory时有六种策略
#（1）volatile-lru：使用LRU算法移除key，只对设置了过期时间的键
#（2）allkeys-lru：使用LRU算法移除key
#（3）volatile-random：在过期集合中移除随机的key，只对设置了过期时间的键
#（4）allkeys-random：移除随机的key
#（5）volatile-ttl：移除那些TTL值最小的key，即那些最近要过期的key
#（6）noeviction：不进行移除。针对写操作，只是返回错误信息
#
# volatile-lru -> Evict using approximated LRU among the keys with an expire set.
# allkeys-lru -> Evict any key using approximated LRU.
# volatile-lfu -> Evict using approximated LFU among the keys with an expire set.
# allkeys-lfu -> Evict any key using approximated LFU.
# volatile-random -> Remove a random key among the ones with an expire set.
# allkeys-random -> Remove a random key, any key.
# volatile-ttl -> Remove the key with the nearest expire time (minor TTL)
# noeviction -> Don't evict anything, just return an error on write operations.
#
# LRU means Least Recently Used 
# LFU means Least Frequently Used
#
# LRU:最近最少使用的
# LFU:最少使用的
# Both LRU, LFU and volatile-ttl are implemented using approximated
# randomized algorithms.
#
# Note: with any of the above policies, Redis will return an error on write
#       operations, when there are no suitable keys for eviction.
#
#       At the date of writing these commands are: set setnx setex append
#       incr decr rpush lpush rpushx lpushx linsert lset rpoplpush sadd
#       sinter sinterstore sunion sunionstore sdiff sdiffstore zadd zincrby
#       zunionstore zinterstore hset hsetnx hmset hincrby incrby decrby
#       getset mset msetnx exec sort
#
# The default is:
#
# maxmemory-policy noeviction

# LRU, LFU and minimal TTL algorithms are not precise algorithms but approximated
# algorithms (in order to save memory), so you can tune it for speed or
# accuracy. For default Redis will check five keys and pick the one that was
# used less recently, you can change the sample size using the following
# configuration directive.
#
# The default of 5 produces good enough results. 10 Approximates very closely
# true LRU but costs more CPU. 3 is faster but not very accurate.# 5个会比较好,10个比较消耗内存,3个则比较快,但是不准确
#
 maxmemory-samples 5 #设置LRU和TTL算法的算法精度
```

### 常见配置

```bash
参数说明
redis.conf 配置项说明如下：
1. Redis默认不是以守护进程的方式运行，可以通过该配置项修改，使用yes启用守护进程
  daemonize no
2. 当Redis以守护进程方式运行时，Redis默认会把pid写入/var/run/redis.pid文件，可以通过pidfile指定
  pidfile /var/run/redis.pid
3. 指定Redis监听端口，默认端口为6379，作者在自己的一篇博文中解释了为什么选用6379作为默认端口，因为6379在手机按键上MERZ对应的号码，而MERZ取自意大利歌女Alessia Merz的名字
  port 6379
4. 绑定的主机地址
  bind 127.0.0.1
5.当 客户端闲置多长时间后关闭连接，如果指定为0，表示关闭该功能
  timeout 300
6. 指定日志记录级别，Redis总共支持四个级别：debug、verbose、notice、warning，默认为verbose
  loglevel verbose
7. 日志记录方式，默认为标准输出，如果配置Redis为守护进程方式运行，而这里又配置为日志记录方式为标准输出，则日志将会发送给/dev/null
  logfile stdout
8. 设置数据库的数量，默认数据库为0，可以使用SELECT <dbid>命令在连接上指定数据库id
  databases 16
9. 指定在多长时间内，有多少次更新操作，就将数据同步到数据文件，可以多个条件配合
  save <seconds> <changes>
  Redis默认配置文件中提供了三个条件：
  save 900 1
  save 300 10
  save 60 10000
  分别表示900秒（15分钟）内有1个更改，300秒（5分钟）内有10个更改以及60秒内有10000个更改。
 
10. 指定存储至本地数据库时是否压缩数据，默认为yes，Redis采用LZF压缩，如果为了节省CPU时间，可以关闭该选项，但会导致数据库文件变的巨大
  rdbcompression yes
11. 指定本地数据库文件名，默认值为dump.rdb
  dbfilename dump.rdb
12. 指定本地数据库存放目录
  dir ./
13. 设置当本机为slav服务时，设置master服务的IP地址及端口，在Redis启动时，它会自动从master进行数据同步
  slaveof <masterip> <masterport>
14. 当master服务设置了密码保护时，slav服务连接master的密码
  masterauth <master-password>
15. 设置Redis连接密码，如果配置了连接密码，客户端在连接Redis时需要通过AUTH <password>命令提供密码，默认关闭
  requirepass foobared
16. 设置同一时间最大客户端连接数，默认无限制，Redis可以同时打开的客户端连接数为Redis进程可以打开的最大文件描述符数，如果设置 maxclients 0，表示不作限制。当客户端连接数到达限制时，Redis会关闭新的连接并向客户端返回max number of clients reached错误信息
  maxclients 128
17. 指定Redis最大内存限制，Redis在启动时会把数据加载到内存中，达到最大内存后，Redis会先尝试清除已到期或即将到期的Key，当此方法处理 后，仍然到达最大内存设置，将无法再进行写入操作，但仍然可以进行读取操作。Redis新的vm机制，会把Key存放内存，Value会存放在swap区
  maxmemory <bytes>
18. 指定是否在每次更新操作后进行日志记录，Redis在默认情况下是异步的把数据写入磁盘，如果不开启，可能会在断电时导致一段时间内的数据丢失。因为 redis本身同步数据文件是按上面save条件来同步的，所以有的数据会在一段时间内只存在于内存中。默认为no
  appendonly no
19. 指定更新日志文件名，默认为appendonly.aof
   appendfilename appendonly.aof
20. 指定更新日志条件，共有3个可选值： 
  no：表示等操作系统进行数据缓存同步到磁盘（快） 
  always：表示每次更新操作后手动调用fsync()将数据写到磁盘（慢，安全） 
  everysec：表示每秒同步一次（折衷，默认值）
  appendfsync everysec
 
21. 指定是否启用虚拟内存机制，默认值为no，简单的介绍一下，VM机制将数据分页存放，由Redis将访问量较少的页即冷数据swap到磁盘上，访问多的页面由磁盘自动换出到内存中（在后面的文章我会仔细分析Redis的VM机制）
   vm-enabled no
22. 虚拟内存文件路径，默认值为/tmp/redis.swap，不可多个Redis实例共享
   vm-swap-file /tmp/redis.swap
23. 将所有大于vm-max-memory的数据存入虚拟内存,无论vm-max-memory设置多小,所有索引数据都是内存存储的(Redis的索引数据 就是keys),也就是说,当vm-max-memory设置为0的时候,其实是所有value都存在于磁盘。默认值为0
   vm-max-memory 0
24. Redis swap文件分成了很多的page，一个对象可以保存在多个page上面，但一个page上不能被多个对象共享，vm-page-size是要根据存储的 数据大小来设定的，作者建议如果存储很多小对象，page大小最好设置为32或者64bytes；如果存储很大大对象，则可以使用更大的page，如果不 确定，就使用默认值
   vm-page-size 32
25. 设置swap文件中的page数量，由于页表（一种表示页面空闲或使用的bitmap）是在放在内存中的，，在磁盘上每8个pages将消耗1byte的内存。
   vm-pages 134217728
26. 设置访问swap文件的线程数,最好不要超过机器的核数,如果设置为0,那么所有对swap文件的操作都是串行的，可能会造成比较长时间的延迟。默认值为4
   vm-max-threads 4
27. 设置在向客户端应答时，是否把较小的包合并为一个包发送，默认为开启
  glueoutputbuf yes
28. 指定在超过一定的数量或者最大的元素超过某一临界值时，采用一种特殊的哈希算法
  hash-max-zipmap-entries 64
  hash-max-zipmap-value 512
29. 指定是否激活重置哈希，默认为开启（后面在介绍Redis的哈希算法时具体介绍）
  activerehashing yes
30. 指定包含其它的配置文件，可以在同一主机上多个Redis实例之间使用同一份配置文件，而同时各个实例又拥有自己的特定配置文件
  include /path/to/local.conf

```



## 查错

> ```bash
> config get dir #启动项目的路径 配置文件可能会根据启动目录不一样而不一样
> ```

### docker

```bash
# 配置dir
dir ./
#daemonize配置,一般我们会配置成后台启动,但是如果在docker中 一定要配置成前台启动,否则会自动退出
daemonize no
```



## 持久化

### RDB

​	Redis Database

> Redis会单独创建（fork）一个子进程来进行持久化，会先将数据写入到
> 一个临时文件中，待持久化过程都结束了，再用这个临时文件替换上次持久化好的文件。
> 整个过程中，主进程是不进行任何IO操作的，这就确保了极高的性能
> 如果需要进行大规模数据的恢复，且对于数据恢复的完整性不是非常敏感，那RDB方
> 式要比AOF方式更加的高效。RDB的缺点是最后一次持久化后的数据可能丢失。
>
> 实际上,就是有一个子进程专门用来持久化,每隔一段时间或者多少次操作之后持久化一次数据,相对于AOF效率更高,但是很大概率会丢失最后一次持久化的数据

#### Fork

> fork的作用就是复制和当前主进程一样的进程,新进程的所有数据(变量,环境变量,程序计数器等)数值和原进程一致,但是是一个全新的进程,并作为原进程的子进程

#### 配置

> <a href="#持久化配置">跳转</a>

#### 触发快照

```bash
#命令: 
127.0.0.1:6379> save
#会阻塞所有的值,将所有的数据缓存到rdb文件中
127.0.0.1:6379> bgsave
#不会阻塞,会在后台异步的执行保存操作
127.0.0.1:6379> lastsave
(integer) 1572169066
# lastsave会将最后一次保存的时间戳返回出来
# 除了手动触发,还有默认配置种的 save [second] [changes] 达到多少秒操作多少次自动触发缓存的操作
 save 900 1 #900秒操作数据一次就会缓存
```

#### 恢复

```bash
#因为默认配置 dir配置的是 ./ 所以在redis-server运行目录中会生成dump.rdb文件.也可以通过config get dir 查看运行的目录
127.0.0.1:6379> CONFIG GET dir
1) "dir"
2) "/data"
#rdb名称可以通过配置修改,默认是dir目录下的dump.rdb
```

![](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20191027175122.png)

#### 优劣势

> 优势: 适合对数据完整性一致性要求不高的环境,数据恢复速度较快
>
> 劣势: 如果redis意外挂了会丢失最后一次需要持久化的数据,并且fork一个进程,内存会克隆一份,大致有两倍内存的膨胀

#### 关闭方法

> 配置文件中,所有的save选项只留一个 save ""
>
> 也可以动态的修改 config set save "" 不需要重启服务的情况下关闭rdb

![](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20191027180144.png)

### AOF

​	Append Only file

> 以日志的形式来记录每个写操作，将Redis执行过的所有写指令记录下来(读操作不记录)，
> 只许追加文件但不可以改写文件，redis启动之初会读取该文件重新构建数据，换言之，redis
> 重启的话就根据日志文件的内容将写指令从前到后执行一次以完成数据的恢复工作

<a href="#持久化配置">配置</a>

#### AOF使用方法

```bash
#正常恢复
#首先在配置文件中 appendonly yes 开启aof
#接着将appendonly.aof 复制到config get dir目录下
#然后重启redis即可
#当aof文件破损或者异常时,redis启动会失败,
#在redis启动目录下存在redis-check-aof文件通过命令
redis-check-aof --fix <filename>修复文件
```



![](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20191027182650.png)

#### AOF Rewirte

- 在一般情况下 aof因为是通过写日志的方式写持久化文件的,很容易出现持久化文件过大的问题,并且在操作缓存中可能会存在对某个缓存多次操作的情况

- 例如:

  ```bash
  set k1 c1
  set k1 c2
  set k1 c3
  set k1 c4
  set k1 c5
  #或者
  set k2 1
  INCR k2 ....................*9999
  #这个时候日志会记录很多没有意义的操作,造成文件过大并且数据恢复速度过慢
  #rewirte机制可以消除这些隐患
  #AOF文件持续增长而过大时，会fork出一条新进程来将文件重写(也是先写临时文件最后再rename)，遍历新进程的内存中数据，每条记录有一条的Set语句。重写aof文件的操作，并没有读取旧的aof文件，而是将整个内存中的数据库内容用命令的方式重写了一个新的aof文件，这点和快照有点类似
  #Redis会记录上次重写时的AOF大小，默认配置是当AOF文件大小是上次rewrite后大小的一倍且文件大于64M时触发
  ```

#### AOF优劣势

- **优势**：相较于RDB，AOF在数据持久化上会更安全，即使持久化策略设置成异步操作，最多也只会丢失两秒的数据。
- **劣势**：缓存文件对比RDB会大很多，运行效率也会比RDB慢一点，不等待磁盘同步的情况下和rdb相同，但是将失去意义，异步相较于会好点，同步则效率会比RDB慢很多。

![](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20191027185612.png)

### 小总结

- RDB持久化方式会按照次数或者时间对缓存数据持久化。
- AOF会将每次操作类似日志的方式写入到磁盘中，并且会在日志文件过大后，执行重写，参数来自配置文件。
- 如果redis服务只用来做缓存，可以不使用任意的持久化方式。
- AOF和RDB可以共存，在共存的时候，重启恢复数据以AOF为主，因为AOF持久化的数据相对于RDB要更为完整。
- 作者建议同时使用AOF和RDB，RDB更适合用于备份，AOF的文件在不断的变化，RDB不会，AOF可能存在潜在的BUG，RDB文件用于防止万一的存在。

## Redis事物

> 可以一次执行多个命令，本质是一组命令的集合。一个事务中的所有命令都会序列化，**按顺序地串行化执行而不会被其它命令插入，不许加塞。**
>
> 一个队列中，一次性、顺序性、排他性的执行一系列命令。

- ![](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20191028220736.png)

### 常用命令

#### 正常执行

```bash
127.0.0.1:6379> MULTI  #开启事物
OK
127.0.0.1:6379> set k1 k
QUEUED
127.0.0.1:6379> set k2 k
QUEUED
127.0.0.1:6379> get k1
QUEUED
127.0.0.1:6379> set id 1
QUEUED
127.0.0.1:6379> INCR id
QUEUED
127.0.0.1:6379> incr id
QUEUED
127.0.0.1:6379> DECr id
QUEUED
127.0.0.1:6379> get id
QUEUED
127.0.0.1:6379> exec #执行事物
1) OK
2) OK
3) "k"
4) OK
5) (integer) 2
6) (integer) 3
7) (integer) 2
8) "2"
```

#### 放弃事物

```bash
127.0.0.1:6379> MULTI  # 开启事物
OK
127.0.0.1:6379> set k1 k
QUEUED
127.0.0.1:6379> set k2 k1
QUEUED
127.0.0.1:6379> set k1 k3
QUEUED
127.0.0.1:6379> DISCARD  # 放弃事物
OK
127.0.0.1:6379> get k1
"k"
127.0.0.1:6379> get k2
"k"
```

#### 全体连坐

```bash
127.0.0.1:6379> MULTI # 开启事物
OK
127.0.0.1:6379> set k1 k3
QUEUED
127.0.0.1:6379> set k2 k4
QUEUED
127.0.0.1:6379> setget k2
(error) ERR unknown command `setget`, with args beginning with: `k2`, 
127.0.0.1:6379> set k3 k3
QUEUED
127.0.0.1:6379> exec # 执行事物，但是如果出现了运行时错误，就会丢弃所有的改动
(error) EXECABORT Transaction discarded because of previous errors.
127.0.0.1:6379> get k1
"k"
127.0.0.1:6379> get k2
"k"
127.0.0.1:6379> get k3
(nil)
```

#### 冤头债主

```bash
127.0.0.1:6379> MULTI # 开启事物
OK
127.0.0.1:6379> set age 1 
QUEUED
127.0.0.1:6379> INCR age
QUEUED
127.0.0.1:6379> set email cjs@cjs.com
QUEUED
127.0.0.1:6379> get age
QUEUED
127.0.0.1:6379> incr email  #自增用在字符串上会出错
QUEUED
127.0.0.1:6379> exec #执行
1) OK
2) (integer) 2
3) OK
4) "2"
5) (error) ERR value is not an integer or out of range # 会报错,但是不会影响其他操作
127.0.0.1:6379> get email #并不会影响其他正常操作
"cjs@cjs.com"
127.0.0.1:6379> get age
"2"

```

#### watch监控

###### 悲观锁

> **顾名思义**,就是比较悲观,每次拿数据的时候都会认为别人会修改,所以在每次拿数据的时候都会对数据上锁,这样别人获取这个数据,就会block直到它拿到锁,传统的关系型数据库里面就用到了很多这种机制,比如行锁,表锁等,读锁,写锁等,都是在做操作之前先上锁

###### 乐观锁

> 和悲观锁对比,相较于悲观锁,则比较乐观每次获取数据都认为我在更改这个数据之前,不会有人对数据进行修改,但是在更新操作的时候通常会对比一下版本号,
>
> 策略:提交的版本必须大于记录的当前版本号

###### CAS(Check And Set)

```bash
#redis可以使用watch [key] 监听一个或者多个key,在下次提交之前 如果监听的key进行了修改,则放弃本次操作.
127.0.0.1:6379> WATCH xyk #监控xyk
OK
127.0.0.1:6379> MULTI #开启事物
OK
#=========================================
#另外一个客户端加塞了一条命令
127.0.0.1:6379> set xyk 400
OK
#=========================================
127.0.0.1:6379> INCRBY xyk 20 
QUEUED
127.0.0.1:6379> DECRBY cunkuan 20
QUEUED
127.0.0.1:6379> exec #执行报错,因为监控的值在事物期间被修改 所以执行错误
(nil)
127.0.0.1:6379>  
#unwatch
127.0.0.1:6379> watch xyk #监控xyk
OK
127.0.0.1:6379> set xyk 1000
OK
127.0.0.1:6379> get xyk
"1000"
127.0.0.1:6379> unwatch #取消监控
OK
127.0.0.1:6379> MULTI #开启事物
OK
127.0.0.1:6379> INCRBY xyk 20
QUEUED
127.0.0.1:6379> get xyk
QUEUED
127.0.0.1:6379> exec #执行成功
1) (integer) 1020
2) "1020"
127.0.0.1:6379> watch xyk #监控xyk
OK
127.0.0.1:6379> set xyk 2000 #在监控期间 下一次事物外修改了这个数据
OK
127.0.0.1:6379> MULTI #开启事物
OK
127.0.0.1:6379> incrby xyk 20 
QUEUED
127.0.0.1:6379> get xyk
QUEUED
127.0.0.1:6379> exec #执行出错
(nil)
```

###### 小节

- Watch指令，类似乐观锁，事务提交时，如果Key的值已被别的客户端改变，比如某个list已被别的客户端push/pop过了，整个事务队列都不会被执行
- 通过WATCH命令在事务执行之前监控了多个Keys，倘若在WATCH之后有任何Key的值发生了变化，EXEC命令执行的事务都将被放弃，同时返回Nullmulti-bulk应答以通知调用者事务执行失败

## 主从

### 一主N从

- 这种情况下,对配置文件没有特殊要求,正常启动多台服务器,我们自己指定一台服务器作为master,其他服务器作为slaver,在slaver的redis服务上执行命令

    - 主从命令

        - ```bash
            127.0.0.1:6379> SLAVEOF 172.18.0.2 6379
            OK #表示成功
            ```

    - 通过info replication可以查看主从状态

        - 主

            - ```bash
                127.0.0.1:6379> info replication
                # Replication
                role:master
                connected_slaves:2 # 从机数量
                slave0:ip=172.18.0.3,port=6379,state=online,offset=2867,lag=1 #从机信息
                slave1:ip=172.18.0.4,port=6379,state=online,offset=2867,lag=1 #从机信息
                master_replid:a88e6136194a0ab33ee515b15e9bff2d27d711eb
                master_replid2:0000000000000000000000000000000000000000
                master_repl_offset:2867
                second_repl_offset:-1
                repl_backlog_active:1
                repl_backlog_size:1048576
                repl_backlog_first_byte_offset:1
                repl_backlog_histlen:2867
                ```

        - 从

            - ```bash
                127.0.0.1:6379> info replication
                # Replication
                role:slave #角色slave
                master_host:172.18.0.2
                master_port:6379
                master_link_status:up
                master_last_io_seconds_ago:1
                master_sync_in_progress:0
                slave_repl_offset:2797
                slave_priority:100
                slave_read_only:1
                connected_slaves:0
                master_replid:a88e6136194a0ab33ee515b15e9bff2d27d711eb
                master_replid2:0000000000000000000000000000000000000000
                master_repl_offset:2797
                second_repl_offset:-1
                repl_backlog_active:1
                repl_backlog_size:1048576
                repl_backlog_first_byte_offset:1
                repl_backlog_histlen:2797
                ```

## 分布式锁

> 分布式锁的实现（Redis）
> 分布式锁实现的关键是在分布式的应用服务器外，搭建一个存储服务器，存储锁信息，这时候我们很容易就想到了Redis。首先我们要搭建一个Redis服务器，用Redis服务器来存储锁信息。
>
> 在实现的时候要注意的几个关键点：
>
> 1、锁信息必须是会过期超时的，不能让一个线程长期占有一个锁而导致死锁；
>
> 2、同一时刻只能有一个线程获取到锁。
> 几个要用到的redis命令：
> setnx(key, value)：“set if not exits”，若该key-value不存在，则成功加入缓存并且返回1，否则返回0。
> get(key)：获得key对应的value值，若不存在则返回nil。
> getset(key, value)：先获取key对应的value值，若不存在则返回nil，然后将旧的value更新为新的value。
> expire(key, seconds)：设置key-value的有效期为seconds秒。
> del(key):删除给定的一个或多个key，不存在的key会被忽略
>
> ![](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20191120165923.jpg)

