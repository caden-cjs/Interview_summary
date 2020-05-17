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

![img](https://mmbiz.qpic.cn/mmbiz/Fb60NIoTYzZGgmXb1mjKvE0EkX4IhomzBxOlEHvbLFQXDGiaMAqFlaj6jBV0Iia0xJj8IeDWybaYcttZY2wzW4rA/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

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

