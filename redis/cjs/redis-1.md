# redis总结

## 分布式锁

> redis分布式锁,主要解决分布式系统中 无法通过本地锁实现多线程数据冲突,从而需要一个**中间件**来保证**多个分布式服务服务共享数据的一致性**



### 梳理需求

> - 支持**立即获取锁方式**，如果**获取到返回true**，获取不到则返回false；
> - 支持等待获取锁方式，如果获取到，直接返回true，获取不到在等待一小段时间，在这一小段时间内反复尝试，如果尝试成功，则返回true，等待时间过后还获取不到则返回false；
> - **不能产生死锁**的情况；
> - **不能释放非自己**加的锁；

#### 加锁

- ![img](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/640)
- 

> 通过上面流程图,我们编写下面第一版

##### 第一版

```go
func (self *RedisLock) LockV1() (bool, error) {
	//判断key是否存在
	exists := redisClient.Clinet.Exists(self.TypeStr)
	if exists.Val() == 1 {
		//若存在则证明已经加锁
		return false, errors.New("已锁")
	} else {
		//不存在则设置一个值表示加锁
		set := redisClient.Clinet.Set(self.TypeStr, self.UUID, time.Second*10)
		if strings.ToUpper(set.String()) == "OK" {
			//加锁成功
			return true, nil
		} else {
			return false, errors.New("加锁失败")
		}
	}
}
```

- 表面上看代码确实不存在问题,但是实际上,**判断key是否存在**，**对key设置值和设置过期时间**，这些操作必须是原子性的，目前的代码可能出现多个客户端都获取到锁的情况，当**第一个客户端判断key是否存在时**，**第二个客户端对key加锁**了，但是**第一个key看到的是当前的key没有锁**，这个时候就存在多个客户端同时获取锁的情况。（还有不允许将set和设置过期时间的操作拆分开，否则可能出现设置值成功后，程序突然崩溃造成的**死锁现象**）

##### 第二版（NX）

- 在redis中解决原子问题可以使用Lua脚本来实现,但是对于上面的场景,在2.6.12版本后,redis官方提供了一个更好的方式，可以看下面的演示，nx选项更类似一个锁，**只有set的value和存储的value相同时才会返回OK，否则无法设置值**

```bash
SET key value [EX seconds] [PX milliseconds] [NX|XX]

# EX seconds – 设置键 key 的过期时间，单位时秒
# PX milliseconds – 设置键 key 的过期时间，单位时毫秒
# NX – 只有键 key 不存在的时候才会设置 key 的值
# XX – 只有键 key 存在的时候才会设置 key 的值

> get test
null
> set test 20 px 10000 nx
OK
> set test 30  px 10000 nx
null
> set test 20  px 10000 nx
OK
> get test
20
```

- go代码

```go
const timeOut time.Duration = 500
func (self *RedisLock) Lock() (bool, error) {
	for i := 0; i < 5; i++ {
        //循环5次,表示如果无法获取到分布式锁则等待,如果5次都没获取到则返回false
		set := redisClient.Clinet.SetNX(self.TypeStr, self.UUID, time.Second*10)
        //这个就是通过NX选项set值
		err := set.Err()
		if err != nil {
			return false, err
		}
        if set.Val() {
            //判断是否set成功
			return true, nil
		} else {
            //未成功则等待半秒
			time.Sleep(timeOut * time.Millisecond)
			//return false, errors.New("无法加锁")
		}
	}
	return false, nil
}
```

#### 解锁

![image-20200524215757655](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/image-20200524215757655.png)

> 因为有了设计加锁的经验，我们应该尽量对解锁的所有操作都实现原子性，但是目前官方并没有类似于setnx这种好用的方式，所以我们需要借用lua脚本来达到原子性解锁操作

```lua
if redis.call('get',KEYS[1])==ARGV[1] 
    then return redis.call('del',KEYS[1]) 
else return 0 end
```

- 上面这个脚本中,我们通过获取key对应的value如果等于我们当前的id,那就删除,否则的话则返回0
- 我们通过代码实现

```go
const script = "if redis.call('get',KEYS[1]) == ARGV[1] then" +
	"   return redis.call('del',KEYS[1]) " +
	"else" +
	"   return 0 " +
	"end"
func (self *RedisLock) UnLock() (bool, error) {
	eval := redisClient.Clinet.Eval(script, []string{self.TypeStr}, []string{self.UUID})
	val, err := eval.Int()
	if err != nil {
		fmt.Println(err.Error())
		return false, err
	} else if val == 1 {
		return true, nil
	} else {
		return false, errors.New("取消分布式锁失败")
	}
}
```

### 测试

#### client.go

```go
// redis_demo/redisClient/client.go
/*
@Time :  2020-05-15 16:26
@Author : Caden
@File : client
@Software: GoLand
*/
package redisClient

import (
	"github.com/go-redis/redis"
)

var Clinet *redis.Client

func init() {
	Clinet = redis.NewClient(&redis.Options{
		Password: "paswd",
		Addr:     "ip:port",
	})
}

```



#### redisLock.go

```go
// redis_demo/redisClient/distributedLock/redisLock.go
/*
@Time :  2020-05-15 16:42
@Author : Caden
@File : redisLock
@Software: GoLand
*/
package distributedLock

import (
	"bytes"
	"errors"
	"fmt"
	uuid "github.com/satori/go.uuid"
	"redis_demo/redisClient"
	"runtime"
	"strconv"
	"strings"
	"sync"
	"time"
)

/**
要实现一个redis分布式锁
	1.支持获取锁,成功true,错误返回false
	2.支持等待获取锁,如果获取到返回true,否则在一小段时间内反复尝试,如果尝试成功,则返回true,否则返回false;
	3.不能产生死锁的情况
	4.
*/
type RedisLock struct {
	UUID    string
	TypeStr string
}

var Sm sync.Map

const timeOut time.Duration = 500

func GetGoroutineID() uint64 {
	b := make([]byte, 64)
	runtime.Stack(b, false)
	b = bytes.TrimPrefix(b, []byte("goroutine "))
	b = b[:bytes.IndexByte(b, ' ')]
	n, _ := strconv.ParseUint(string(b), 10, 64)
	return n
}

const script = "if redis.call('get',KEYS[1]) == ARGV[1] then" +
	"   return redis.call('del',KEYS[1]) " +
	"else" +
	"   return 0 " +
	"end"

func NewRedisLock(typeStr string) *RedisLock {
	v4 := uuid.NewV4()
	return &RedisLock{
		UUID:    v4.String(),
		TypeStr: typeStr,
	}
}
func (self *RedisLock) Lock() (bool, error) {
	//SET lock_key random_value NX PX 5000,表示的是 typestr锁定,value是一个uuid,锁定时间10s
	//nx的方式表示的是,如果这个key存在了,则不会set并且会有错
	//nx的方式和普通的set方式,如果在于增加了NX标识符,将之前exists,set or exists 这两组操作都变成了原子操作,避免了
	//加锁不成功的问题
	for i := 0; i < 5; i++ {
		set := redisClient.Clinet.SetNX(self.TypeStr, self.UUID, time.Second*10)
		err := set.Err()

		if err != nil {
			return false, err
		}
		if set.Val() {
			return true, nil
		} else {
			time.Sleep(timeOut * time.Millisecond)
			//return false, errors.New("无法加锁")
		}
	}
	return false, nil
}
func (self *RedisLock) LockV1() (bool, error) {
	//判断key是否存在
	exists := redisClient.Clinet.Exists(self.TypeStr)
	if exists.Val() == 1 {
		//若存在则证明已经加锁
		return false, errors.New("已锁")
	} else {
		//不存在则设置一个值表示加锁
		set := redisClient.Clinet.Set(self.TypeStr, self.UUID, time.Second*10)
		if strings.ToUpper(set.String()) == "OK" {
			//加锁成功
			return true, nil
		} else {
			return false, errors.New("加锁失败")
		}
	}
}
func (self *RedisLock) UnLock() (bool, error) {
	eval := redisClient.Clinet.Eval(script, []string{self.TypeStr}, []string{self.UUID})
	val, err := eval.Int()
	if err != nil {
		fmt.Println(err.Error())
		return false, err
	} else if val == 1 {
		return true, nil
	} else {
		return false, errors.New("取消分布式锁失败")
	}
}

```

#### main.go

```go
// redis_demo/main.go
/*
@Time :  2020-05-15 16:31
@Author : Caden
@File : main
@Software: GoLand
*/
package main

import (
	"fmt"
	"math/rand"
	"redis_demo/redisClient/distributedLock"
	"sync"
	"time"
)

func main() {
	typeSlice := [3]string{"type1", "type2", "type3"}
	group := sync.WaitGroup{}
	var f func(int)
	f = func(i int) {
		id := distributedLock.GetGoroutineID()
		lockNum := typeSlice[i%3]
		lock := distributedLock.NewRedisLock(lockNum)
		rand.Seed(time.Now().UnixNano())
		for {
			intn := rand.Intn(2000)
			lockSuc, err := lock.Lock()
			if err != nil {
				fmt.Println(err)
			}
			if lockSuc {
				//fmt.Printf("%v获取到%v锁了\n", id, lockNum)
				distributedLock.Sm.Store(id, lockNum)
				time.Sleep(time.Millisecond * time.Duration(intn))
				unLock, err := lock.UnLock()
				if err != nil {
					fmt.Println(err)
				}
				if unLock {
					distributedLock.Sm.Delete(id)
					//fmt.Printf("%v解锁成功\n", id)
				} else {
					fmt.Printf("%v解锁失败\n", id)
				}
			}
		}
	}
	group.Add(1)
	go func() {
		for {
			time.Sleep(time.Millisecond*500)
			i := 0
			distributedLock.Sm.Range(func(key, value interface{}) bool {
				fmt.Printf("value=%v\t", value)
				i++
				return true
			})
			fmt.Println()
			if i > 3 {
				fmt.Printf("出错了\n")
				distributedLock.Sm.Range(func(key, value interface{}) bool {
					fmt.Printf("key=%v,value=%v\t", key, value)
					return true
				})
			}
		}
	}()
	for i := 0; i < 100; i++ {
		go f(i)
	}
	group.Wait()
	fmt.Println("运行结束")
}
```

### **多节点 Redis 分布式锁：Redlock 算法**

> 获取当前时间（start）。
>
> 依次向 N 个 `Redis`节点请求锁。请求锁的方式与从单节点 `Redis`获取锁的方式一致。为了保证在某个 `Redis`节点不可用时该算法能够继续运行，获取锁的操作都需要设置超时时间，需要保证该超时时间远小于锁的有效时间。这样才能保证客户端在向某个 `Redis`节点获取锁失败之后，可以立刻尝试下一个节点。
>
> 计算获取锁的过程总共消耗多长时间（consumeTime = end - start）。如果客户端从大多数 `Redis`节点（>= N/2 + 1) 成功获取锁，并且获取锁总时长没有超过锁的有效时间，这种情况下，客户端会认为获取锁成功，否则，获取锁失败。
>
> 如果最终获取锁成功，锁的有效时间应该重新设置为锁最初的有效时间减去 `consumeTime`。
>
> 如果最终获取锁失败，客户端应该立刻向所有 `Redis`节点发起释放锁的请求。

## 数据结构[5种]

![image-20200524215947120](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/image-20200524215947120.png)

- string

> 二进制安全的,可以存储任何类型的对象。
>
> 操作：get，set，del，incr，decr
>
> 场景：
> 	缓存：可以把任意的对象数据序列化之后直接使用**redis**的**string**类型存储到redis服务器中，使用redis作为**缓存层**,持久层使用常用的数据库,可以**降低持久层的读写压力**。
>
> ​	计数器：**redis**是单线程模型，可以使用字段的**自增功能**，保证分布式系统的序号计数不重复。
>
> ​	session：常见的方案将原本缓存在本地服务中的session存到redis中，通过redis实现session的共享
>
> 命令：
>
> ```bash
> 127.0.0.1:6379> AUTH ohaRUIuX
> OK
> 127.0.0.1:6379> set hello world
> OK
> 127.0.0.1:6379> get hello
> "world"
> 127.0.0.1:6379> del hello
> (integer) 1
> 127.0.0.1:6379> get count
> (nil)
> 127.0.0.1:6379> set count 1
> OK
> 127.0.0.1:6379> get count
> "1"
> 127.0.0.1:6379> incr count
> (integer) 2
> 127.0.0.1:6379> incrby count 100
> (integer) 102
> 127.0.0.1:6379> get count
> "102"
> 127.0.0.1:6379> DECR count 
> (integer) 101
> 127.0.0.1:6379> get count
> "101"
> 127.0.0.1:6379> DECRBY count 100
> (integer) 1
> ```
>
> 

- hash（类似HashMap）

![image-20200524215916168](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/image-20200524215916168.png)

> 本身就是一种键值对的类型
>
> 操作：hget、hset、hdel
>
> 场景：
>
> ​	缓存： 能直观，相比string更节省空间的维护缓存信息，如用户信息，视频信息等。
>
> ```bash
> 127.0.0.1:6379> hset user name ccccc
> (integer) 0
> 127.0.0.1:6379> hget user name
> "ccccc"
> 127.0.0.1:6379> HKEYS user
> 1) "name"
> 2) "age"
> 127.0.0.1:6379> hgetall user
> 1) "name"
> 2) "ccccc"
> 3) "age"
> 4) "26"
> 127.0.0.1:6379> hdel user name
> (integer) 1
> 127.0.0.1:6379> hgetall user
> 1) "age"
> 2) "26"
> ```

- List

![image-20200518203907521](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/image-20200518203907521.png)

> 实际上就是链表(Linked list),有序的,value可以重复

- Set
- sorted Set

## 特殊结构（bitMap）

> 一般用来解决**缓存穿透**（热点数据，数据库中不存在）
>
> ## **布隆过滤器**
>
> bloomfilter就类似于一个hash set，用于快速判某个元素是否存在于集合中，其典型的应用场景就是快速判断一个key是否存在于某容器，不存在就直接返回。布隆过滤器的关键就在于hash算法和容器大小
>
> ## **Redis Module 实现布隆过滤器**
>
> Redis module 是Redis 4.0 以后支持的新的特性，这里很多国外牛逼的大学和机构提供了很多牛逼的Module 只要编译引入到Redis 中就能轻松的实现我们某些需求的功能。在Redis 官方Module 中有一些我们常见的一些模块，我们在这里就做一个简单的使用。
>
> - neural-redis 主要是神经网络的机器学，集成到redis 可以做一些机器训练感兴趣的可以尝试
> - RedisSearch 主要支持一些富文本的的搜索
> - RedisBloom 支持分布式环境下的Bloom 过滤器

## Reids6种淘汰策略

- **noeviction**: 不删除策略, 达到最大内存限制时, 如果需要更多内存, 直接返回错误信息。大多数写命令都会导致占用更多的内存(有极少数会例外。
- **allkeys-lru:**所有key通用; 优先删除最近最少使用(less recently used ,LRU) 的 key。
- **volatile-lru:**只限于设置了 expire 的部分; 优先删除最近最少使用(less recently used ,LRU) 的 key。
- **allkeys-random:**所有key通用; 随机删除一部分 key。
- **volatile-random**: 只限于设置了 **expire** 的部分; 随机删除一部分 key。
- **volatile-ttl**: 只限于设置了 **expire** 的部分; 优先删除剩余时间(time to live,TTL) 短的key。

## 持久化

> AOF 和 RDB
>
> ## **Redis 开启AOF**
>
> Redis服务器默认开启RDB，关闭AOF；要开启AOF，需要在配置文件中配置：
>
> appendonly yes
>
> ## **AOF常用配置总结**
>
> 下面是AOF常用的配置项，以及默认值；前面介绍过的这里不再详细介绍。
>
> - appendonly no：是否开启AOF
> - appendfilename "appendonly.aof"：AOF文件名
> - dir ./：RDB文件和AOF文件所在目录
> - appendfsync everysec：fsync持久化策略
> - no-appendfsync-on-rewrite no：AOF重写期间是否禁止fsync；如果开启该选项，可以减轻文件重写时CPU和硬盘的负载（尤其是硬盘），但是可能会丢失AOF重写期间的数据；需要在负载和安全性之间进行平衡
> - auto-aof-rewrite-percentage 100：文件重写触发条件之一
> - auto-aof-rewrite-min-size 64mb：文件重写触发提交之一
> - aof-load-truncated yes：如果AOF文件结尾损坏，Redis启动时是否仍载入AOF文件
>
> ## **RDB和AOF的优缺点**
>
> **RDB持久化**
>
> 优点：RDB文件紧凑，体积小，网络传输快，适合全量复制；恢复速度比AOF快很多。当然，与AOF相比，RDB最重要的优点之一是对性能的影响相对较小。
>
> 缺点：RDB文件的致命缺点在于其数据快照的持久化方式决定了必然做不到实时持久化，而在数据越来越重要的今天，数据的大量丢失很多时候是无法接受的，因此AOF持久化成为主流。此外，RDB文件需要满足特定格式，兼容性差（如老版本的Redis不兼容新版本的RDB文件）。
>
> **AOF持久化**
>
> 与RDB持久化相对应，AOF的优点在于支持秒级持久化、兼容性好，缺点是文件大、恢复速度慢、对性能影响大。
>
> ## **持久化策略选择**
>
> （1）如果Redis中的数据完全丢弃也没有关系（如Redis完全用作DB层数据的cache），那么无论是单机，还是主从架构，都可以不进行任何持久化。
>
> （2）在单机环境下（对于个人开发者，这种情况可能比较常见），如果可以接受十几分钟或更多的数据丢失，选择RDB对Redis的性能更加有利；如果只能接受秒级别的数据丢失，应该选择AOF。
>
> （3）但在多数情况下，我们都会配置主从环境，slave的存在既可以实现数据的热备，也可以进行读写分离分担Redis读请求，以及在master宕掉后继续提供服务。
>
> ## **为什么需要持久化？**
>
> 由于Redis是一种内存型数据库，即服务器在运行时，系统为其分配了一部分内存存储数据，一旦服务器挂了，或者突然宕机了，那么数据库里面的数据将会丢失，为了使服务器即使突然关机也能保存数据，必须通过持久化的方式将数据从内存保存到磁盘中。

## Redis是单线程的，但Redis为什么这么快？

> 1、完全基于内存，绝大部分请求是纯粹的内存操作，非常快速。数据存在内存中，类似于HashMap，HashMap的优势就是查找和操作的时间复杂度都是O(1)；
>
> 2、数据结构简单，对数据操作也简单，Redis中的数据结构是专门进行设计的；
>
> 3、采用单线程，避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗 CPU，不用去考虑各种锁的问题，不存在加锁释放锁操作，没有因为可能出现死锁而导致的性能消耗；
>
> 4、使用多路I/O复用模型，非阻塞IO；**这里“多路”指的是多个网络连接，“复用”指的是复用同一个线程**
>
> 5、使用底层模型不同，它们之间底层实现方式以及与客户端之间通信的应用协议不一样，Redis直接自己构建了VM 机制 ，因为一般的系统调用系统函数的话，会浪费一定的时间去移动和请求；

## **为什么Redis是单线程的？**

> Redis是基于内存的操作，CPU不是Redis的瓶颈，Redis的瓶颈最有可能是机器内存的大小或者网络带宽。既然单线程容易实现，而且CPU不会成为瓶颈，那就顺理成章地采用单线程的方案了（毕竟采用多线程会有很多麻烦！）。

## Redis内存模型

> **used_memory**：Redis分配器分配的内存总量（单位是字节），包括使用的虚拟内存（即swap）；Redis分配器后面会介绍。used_memory_human只是显示更友好。
>
> **used_memory_rss****：**Redis进程占据操作系统的内存（单位是字节），与top及ps命令看到的值是一致的；除了分配器分配的内存之外，used_memory_rss还包括进程运行本身需要的内存、内存碎片等，但是不包括虚拟内存。
>
> **mem_fragmentation_ratio****：**内存碎片比率，该值是used_memory_rss / used_memory的比值。
>
> **mem_allocator****：**Redis使用的内存分配器，在编译时指定；可以是 libc 、jemalloc或者tcmalloc，默认是jemalloc；截图中使用的便是默认的jemalloc。

## Redis内存划分

> ## **数据**
>
> 作为数据库，数据是最主要的部分；这部分占用的内存会统计在used_memory中。
>
> ## **进程本身运行需要的内存**
>
> Redis主进程本身运行肯定需要占用内存，如代码、常量池等等；这部分内存大约几兆，在大多数生产环境中与Redis数据占用的内存相比可以忽略。这部分内存不是由jemalloc分配，因此不会统计在used_memory中。
>
> ## **缓冲内存**
>
> 缓冲内存包括客户端缓冲区、复制积压缓冲区、AOF缓冲区等；其中，客户端缓冲存储客户端连接的输入输出缓冲；复制积压缓冲用于部分复制功能；AOF缓冲区用于在进行AOF重写时，保存最近的写入命令。在了解相应功能之前，不需要知道这些缓冲的细节；这部分内存由jemalloc分配，因此会统计在used_memory中。
>
> ## **内存碎片**
>
> 内存碎片是Redis在分配、回收物理内存过程中产生的。例如，如果对数据的更改频繁，而且数据之间的大小相差很大，可能导致redis释放的空间在物理内存中并没有释放，但redis又无法有效利用，这就形成了内存碎片。内存碎片不会统计在used_memory中。

## **Reids主从复制**

> 复制是高可用Redis的基础，哨兵和集群都是在复制基础上实现高可用的。复制主要实现了数据的多机备份，以及对于读操作的负载均衡和简单的故障恢复。缺陷：故障恢复无法自动化；写操作无法负载均衡；存储能力受到单机的限制。
>
> 在复制的基础上，哨兵实现了自动化的故障恢复。缺陷：写操作无法负载均衡；存储能力受到单机的限制。

## **redis缓存被击穿处理机制**

> 使用mutex。简单地来说，就是在缓存失效的时候（判断拿出来的值为空），不是立即去load db，而是先使用缓存工具的某些带成功操作返回值的操作（比如Redis的SETNX或者Memcache的ADD）去set一个mutex key，当操作返回成功时，再进行load db的操作并回设缓存；否则，就重试整个get缓存的方法

## **缓存雪崩问题**

> 存在同一时间内大量键过期（失效），接着来的一大波请求瞬间都落在了数据库中导致连接异常。
>
> 解决方案：
>
> 1、也是像解决缓存穿透一样加锁排队。
>
> 2、建立备份缓存，缓存A和缓存B，A设置超时时间，B不设值超时时间，先从A读缓存，A没有读B，并且更新A缓存和B缓存;

## **缓存并发问题**

> 这里的并发指的是多个redis的client同时set key引起的并发问题。比较有效的解决方案就是把redis.set操作放在队列中使其串行化，必须的一个一个执行，具体的代码就不上了，当然加锁也是可以的，至于为什么不用redis中的事务，留给各位看官自己思考探究。

## **Redis分布式**

> redis支持主从的模式。原则：Master会将数据同步到slave，而slave不会将数据同步到master。Slave启动时会连接master来同步数据。
>
> 这是一个典型的分布式读写分离模型。我们可以利用master来插入数据，slave提供检索服务。这样可以有效减少单个机器的并发访问数量

## **读写分离模型**

> 通过增加Slave DB的数量，读的性能可以线性增长。为了避免Master DB的单点故障，集群一般都会采用两台Master DB做双机热备，所以整个集群的读和写的可用性都非常高。读写分离架构的缺陷在于，不管是Master还是Slave，每个节点都必须保存完整的数据，如果在数据量很大的情况下，集群的扩展能力还是受限于单个节点的存储能力，而且对于Write-intensive类型的应用，读写分离架构并不适合。

## **数据分片模型**

> 为了解决读写分离模型的缺陷，可以将数据分片模型应用进来。
>
> 可以将每个节点看成都是独立的master，然后通过业务实现数据分片。
>
> 结合上面两种模型，可以将每个master设计成由一个master和多个slave组成的模型。

## **Redis还提供的高级工具**

> 像慢查询分析、性能测试、Pipeline、事务、Lua自定义命令、Bitmaps、HyperLogLog、发布/订阅、Geo等个性化功能。

## **Redis常用管理命令**

> ```bash
> # dbsize 返回当前数据库 key 的数量。
> # info 返回当前 redis 服务器状态和一些统计信息。
> # monitor 实时监听并返回redis服务器接收到的所有请求信息。
> # shutdown 把数据同步保存到磁盘上，并关闭redis服务。
> # config get parameter 获取一个 redis 配置参数信息。（个别参数可能无法获取）
> # config set parameter value 设置一个 redis 配置参数信息。（个别参数可能无法获取）
> # config resetstat 重置 info 命令的统计信息。（重置包括：keyspace 命中数、
> # keyspace 错误数、 处理命令数，接收连接数、过期 key 数）
> # debug object key 获取一个 key 的调试信息。
> # debug segfault 制造一次服务器当机。
> # flushdb 删除当前数据库中所有 key,此方法不会失败。小心慎用
> # flushall 删除全部数据库中所有 key，此方法不会失败。小心慎用
> ```

## **Redis中海量数据的正确操作方式**

> 利用SCAN系列命令（SCAN、SSCAN、HSCAN、ZSCAN）完成数据迭代。
>
> ## **SCAN系列命令注意事项**
>
> - SCAN的参数没有key，因为其迭代对象是DB内数据；
> - 返回值都是数组，第一个值都是下一次迭代游标；
> - 时间复杂度：每次请求都是O(1)，完成所有迭代需要O(N)，N是元素数量；
> - 可用版本：version >= 2.8.0；

## 缓存和数据库间数据一致性问题

> 分布式环境下（单机就不用说了）非常容易出现缓存和数据库间的数据一致性问题，针对这一点的话，只能说，如果你的项目对缓存的要求是强一致性的，那么请不要使用缓存。我们只能采取合适的策略来降低缓存和数据库间数据不一致的概率，而无法保证两者间的强一致性。合适的策略包括 合适的缓存更新策略，更新数据库后要及时更新缓存、缓存失败时增加重试机制，例如MQ模式的消息队列。

## **redis常见性能问题和解决方案：**

> Master最好不要做任何持久化工作，如RDB内存快照和AOF日志文件
>
> 如果数据比较重要，某个Slave开启AOF备份数据，策略设置为每秒同步一次
>
> 为了主从复制的速度和连接的稳定性，Master和Slave最好在同一个局域网内
>
> 尽量避免在压力很大的主库上增加从库

## **redis通讯协议**

> RESP 是redis客户端和服务端之前使用的一种通讯协议；RESP 的特点：实现简单、快速解析、可读性好

## **Redis做异步队列**

> 一般使用list结构作为队列，rpush生产消息，lpop消费消息。当lpop没有消息的时候，要适当sleep一会再重试。缺点：在消费者下线的情况下，生产的消息会丢失，得使用专业的消息队列如rabbitmq等。**能不能生产一次消费多次呢？**使用pub/sub主题订阅者模式，可以实现1:N的消息队列。

## **Redis 管道 Pipeline**

> 在某些场景下我们在**一次操作中可能需要执行多个命令**，而如果我们只是一个命令一个命令去执行则会浪费很多网络消耗时间，如果将命令一次性传输到 `Redis`中去再执行，则会减少很多开销时间。但是需要注意的是 `pipeline`中的命令并不是原子性执行的，也就是说管道中的命令到达 `Redis`服务器的时候可能会被其他的命令穿插

## 手写LRU算法

> ```java
> class LRUCache<K, V> extends LinkedHashMap<K, V> {
>     private final int CACHE_SIZE;
> 
>     /**
>      * 传递进来最多能缓存多少数据
>      *
>      * @param cacheSize 缓存大小
>      */
>     public LRUCache(int cacheSize) {
>         // true 表示让 linkedHashMap 按照访问顺序来进行排序，最近访问的放在头部，最老访问的放在尾部。
>         super((int) Math.ceil(cacheSize / 0.75) + 1, 0.75f, true);
>         CACHE_SIZE = cacheSize;
>     }
> 
>     @Override
>     protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
>         // 当 map中的数据量大于指定的缓存个数的时候，就自动删除最老的数据。
>         return size() > CACHE_SIZE;
>     }
> }
> ```

## **Reids三种不同删除策略**

> **定时删除**：在设置键的过期时间的同时，创建一个定时任务，当键达到过期时间时，立即执行对键的删除操作
>
> **惰性删除**：放任键过期不管，但在每次从键空间获取键时，都检查取得的键是否过期，如果过期的话，就删除该键，如果没有过期，就返回该键
>
> **定期删除**：每隔一点时间，程序就对数据库进行一次检查，删除里面的过期键，至于要删除多少过期键，以及要检查多少个数据库，则由算法决定。
>
> ## **定时删除**
>
> - **优点：**对内存友好，定时删除策略可以保证过期键会尽可能快地被删除，并释放国期间所占用的内存
> - **缺点：**对cpu时间不友好，在过期键比较多时，删除任务会占用很大一部分cpu时间，在内存不紧张但cpu时间紧张的情况下，将cpu时间用在删除和当前任务无关的过期键上，影响服务器的响应时间和吞吐量
>
> ## **定期删除**
>
> 由于定时删除会占用太多cpu时间，影响服务器的响应时间和吞吐量以及惰性删除浪费太多内存，有内存泄露的危险，所以出现一种整合和折中这两种策略的定期删除策略。
>
> 1. 定期删除策略每隔一段时间执行一次删除过期键操作，并通过限制删除操作执行的时长和频率来减少删除操作对CPU时间的影响。
> 2. 定时删除策略有效地减少了因为过期键带来的内存浪费。
>
> ## **惰性删除**
>
> - **优点：**对cpu时间友好，在每次从键空间获取键时进行过期键检查并是否删除，删除目标也仅限当前处理的键，这个策略不会在其他无关的删除任务上花费任何cpu时间。
> - **缺点：**对内存不友好，过期键过期也可能不会被删除，导致所占的内存也不会释放。甚至可能会出现内存泄露的现象，当存在很多过期键，而这些过期键又没有被访问到，这会可能导致它们会一直保存在内存中，造成内存泄露。

## **Redis常见的几种缓存策略**

- Cache-Aside
- Read-Through
- Write-Through
- Write-Behind

## **Redis 到底是怎么实现“附近的人”**

> ### **使用方式**
>
> ```bash
> GEOADD key longitude latitude member [longitude latitude member ...]
> ```
>
> 将给定的位置对象（纬度、经度、名字）添加到指定的key。其中，key为集合名称，member为该经纬度所对应的对象。在实际运用中，当所需存储的对象数量过多时，可通过设置多key(如一个省一个key)的方式对对象集合变相做sharding，避免单集合数量过多。
>
> 成功插入后的返回值：
>
> ```bash
> (integer) N
> ```
>
> 其中N为成功插入的个数。

## 事物

