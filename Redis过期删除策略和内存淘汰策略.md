
# 过期删除策略
redis所有的数据结构都可以设置过期时间，时间一到就会被自动删除。
由于redis的核心处理逻辑是单线程的，如果同一时间有过多的key同时过期就会导致主线程处理过期的key花费太多的时间，从而导致主线程阻塞可能无法执行线上的读写指令。
## 如何设置过期时间
创建字符串的同时设置过期时间
 - set  key value ex n：精确到秒
 - set  key value px n：精确到毫秒
 - setex  key n value :精确到秒
 - setpx key n value：精确到毫秒

设置key之后，设置过期时间
 - expire key n : 设置key在n秒之后过期
 - expireat key n：设置key在某个时间戳(精确到秒)之后过期
 - expiretime key：获取key的过期时间戳
 - pexpire key n：设置key在n毫秒之后过期
 - pexpireat  key n：设置key在某个时间戳(精确到毫秒)之后过期
 - pexpiretime key：获取key的过期时间戳

有关过期时间的一些操作
 
 - 查看某个key剩余的存活时间   ttl  key[-1表示永不过期，-2表示已经过期，>=0表示剩余的存活时间]
 - 取消key的过期时间，persist key
## 过期的key被放置在哪里?
redis会将每个设置了过期时间的key以及过期的时间存储在一个独立的字典(dict)中，称之为过期字典。[有关字典的源码会在后续的文章中分析]

```c
typedef struct redisDb {
    dict *dict;                 //数据字典，存储所有的key
    dict *expires;              //过期字典，存储带有过期时间的key
	..................
} redisDb;
```

过期字典中的key该如何删除呢？先来介绍一下常见的过期删除策略。
## 常见的过期删除策略
### 过期删除策略
过期删除，顾名思义一旦过期立马删除。
优点：
内存会尽快得到释放
缺点：
同一时间过期的key过多会导致主线程阻塞，无法继续执行线上的指令。
### 惰性删除策略
惰性删除的策略是，不会主动删除过期的key，只有当客户端访问该key的时候，redis才从过期字典中获取key的过期时间进行判断，如果过期立即删除。
优点
不会因为同一时间处理大量过期的key而造成主线程的阻塞
缺点
过期的key会一直占用着内存
### 定期扫描策略
定期删除也可以称之为定时删除，具体的做法是：每隔一段时间进行过期key的清理
优点
一种处在过期删除和惰性删除中间的策略，不会频繁的导致主线程阻塞也不会长时间占用内存。
缺点
内存清理方面没有过期删除效果好，同时没有惰性删除使用的系统资源少
难以确定删除操作执行的时长和频率。
## Redis过期删除策略
**redis选择惰性删除和定期删除**
如果说定期删除是集中处理，那么惰性删除就是零散处理。
### 惰性删除策略
Redis 的惰性删除策略由 db.c 文件中的 expireIfNeeded 函数实现，代码如下：[redis-5.0.13\src\db.c]

```c
/*
当我们操作一些过期仍然存在redis中的key时，调用该函数。
从节点不会主动删除任何过期key ，它等待主节点的del指令。
在主节点中，发现key过期将其从redis中驱除，并将del指令记录在AOF中.
在主从节点中，key依然有效返回0，反之返回1
*/
int expireIfNeeded(redisDb *db, robj *key) {
//key依然有效，直接返回0
    if (!keyIsExpired(db,key)) return 0;

    //该节点是从节点，直接返回。key有效返回1
    if (server.masterhost != NULL) return 1;

    /* Delete the key */
    //主节点删除过期key
    //过期key的个数自增
    server.stat_expiredkeys++;
    /*
    Propagate到slave和AOF文件过期。
    当一个key在主服务器中过期时，针对该key的DEL操作将被发送到所有从服务器和AOF文件(如果启用)。
    这样键的过期就集中在一个地方，而且由于AOF和主-从链路都保证了操作的顺序，所以即使我们允许对过期的键进行写操作，一切都将是一致的。
    */
    propagateExpire(db,key,server.lazyfree_lazy_expire);
    notifyKeyspaceEvent(NOTIFY_EXPIRED,
        "expired",key,db->id);
        //根据server.lazyfree_lazy_expire的值决定是异步删除还是同步删除
    return server.lazyfree_lazy_expire ? dbAsyncDelete(db,key) :
                                         dbSyncDelete(db,key);
}
```

Redis在访问key之前，会先调用expireifNeeded函数对其进行检查，检查key是否过期

 - 没有过期依然有效直接返回0。过期无效有以下两种情况：
 - 从节点，不删除key直接返回1
 - 主节点删除key，并将删除key的del指令同步到所有从节点和AOF(如果开启的话)

### 定期扫描策略
#### 扫描的频率
在 Redis 中，<font color='blue'>默认每秒进行 10 次过期扫描</font>。过期扫描不会遍历过期字典中的所有key，而是采用一种随机的策略。

#### 定期扫描的流程
 - (1)从过期字典中随机选出20个key
 - (2)删除这20个key中已经过期的key
 - (3)如果过期的key的比例超过1/4，重复步骤(1)。直到已过期key的数量的占比小于25%。同时，为了保证过期扫描不会出现循环过度，导致线程卡死的现象，增加扫描时间的上限，默认不会超过 25ms。
 
 **假设redis实例中的所有key在同一时间过期，会出现什么样的情况**
 
 - redis会持续扫描过期字典，直到过期字典中过期的key变得稀疏。会导致线上读写请求出现明显卡顿现象。
 - 导致这种卡顿现象的另外一种原因是，内存管理器需要频繁回收内存页。
 - 如果客户端请求到来时，redis正好进入过期扫描状态，客户端的请求将会等待至少25ms后才会进行，如果客户端将超时时间设置得比较短，就会出现大量的链接因为超时而关闭，业务端就会出现异常，并且这些异常无法从日志中查到慢查询记录，因为slowlog记录的是逻辑处理慢，不包括等待时间。
 - 最好的解决方式就是将key的过期时间随机化。

### 	从节点的过期策略
 - 从节点不会主动进行过期扫描，从节点对过期key的处理是被动的。
 - 主节点在删除过期key时，会在AOF[一种持久化的方式]文件里增加一条del指令，同步到所有的从节点，从节点通过执行这个del指令删除过期的key。
 - 由于指令同步是异步进行的，所以如果主节点过期的key的del指令没有及时同步到从节点上，就会出现主从数据的不一致，主节点没有的数据依然存在从节点。

# 内存淘汰策略
当redis内存超过物理内存限制时，内存的数据会开始和磁盘产生频繁的交换(swap)。交换使得redis的性能急剧下降。
在生产环境中是不允许redis出现交换行为的，为了限制最大的使用内存，redis提供了配置参数"maxmemory"限制内存超出期望大小。
## Redis最大运行内存
利用"config get maxmemory"获取最大运行内存
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/0eb1b4fbd31cee4eda27546780aef2f0.png)

在配置文件 redis.conf 中，可以通过参数 maxmemory <bytes> 来设定最大运行内存，

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/f04e0e5ad3dcd72e5edb1e138bd6d11e.png)
## 内存淘汰策略
在redis实例的使用内存达到内存上限之后，会触发"内存淘汰"。
源码中，有关"内存淘汰策略"的解释

**进行数据淘汰的策略**
**淘汰过期字典中的key**

```c
/*采用LRU算法驱逐过期字典中的key*/
#define MAXMEMORY_VOLATILE_LRU 
/*采用LFU算法驱逐过期字典中的key*/
#define MAXMEMORY_VOLATILE_LFU //4.0版本之后引入的
/*根据ttl的值驱逐过期字典中的key*/
#define MAXMEMORY_VOLATILE_TTL 
/*随机驱逐过期字典中的key*/
#define MAXMEMORY_VOLATILE_RANDOM 
```
**从数据字典中淘汰key**
```cpp
/*采用LRU算法驱逐数据字典的key*/
#define MAXMEMORY_ALLKEYS_LRU 
/*采用LFU算法驱逐数据字典的key*/
#define MAXMEMORY_ALLKEYS_LFU //4.0版本之后引入的
/*采用随机算法驱逐数据字典的key*/
#define MAXMEMORY_ALLKEYS_RANDOM 
```

**不进行淘汰的策略**
```cpp
/*内存达到上限之后，不驱逐任何key。线上只能执行读操作不能执行写操作*/
#define MAXMEMORY_NO_EVICTION
```





## 默认的内存淘汰策略

```c
#define CONFIG_DEFAULT_MAXMEMORY_POLICY MAXMEMORY_NO_EVICTION
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/a5b890fe6c942f6a926758955cdba32c.png)

## 设置内存淘汰策略

 - 方式一：通过“config set maxmemory-policy <策略>”命令设置。它的优点是设置之后立即生效，不需要重启 Redis 服务，缺点是重启 Redis 之后，设置就会失效。
 - 方式二：通过修改 Redis 配置文件修改，设置“maxmemory-policy <策略>”，它的优点是重启 Redis 服务后配置不会丢失，缺点是必须重启 Redis 服务，设置才能生效。

## LRU && LFU

 -  LRU means Least Recently Used；LFU means Least Frequently Used
 - 更多有关"LRU"和"LFU"的知识可以参考文章"[https://blog.csdn.net/weixin_46290302/article/details/132570300?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522170586522916800226589359%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=170586522916800226589359&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-1-132570300-null-null.nonecase&utm_term=LRU&spm=1018.2226.3001.4450](https://blog.csdn.net/weixin_46290302/article/details/132570300?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522170586522916800226589359%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=170586522916800226589359&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-1-132570300-null-null.nonecase&utm_term=LRU&spm=1018.2226.3001.4450)"



redis中的"LRU"和"LFU"算法采用的是一种随机化算法。
### 近似 LRU算法

redis使用的是一种近似LRU算法。之所以不使用LRU算法，是因为其需要消耗大量的内存存储额外的指针。


 - 近似LRU算法，在现有数据结构的基础上使用随机采样法淘汰元素。随机采样法需要用到"key最后一次被访问的时间戳"，这个字段存储在"redisObject"中的"lru"字段

```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

 - 在LRU模式下。lru字段存储的是Redis时钟server.lruclock。Redis时钟是一个24bit的整数，默认是Unix时间戳对2^24^取模的结果，大约97天清零一次。<font color='red'>当某个key被访问一次，对象头结构的lru字段值被更新为server.lruclock。</font>
 - 如果server.lruclock没有折返[对2^24^取模]，则一直是递增的，意味着lru字段不会超过server.lrulock的值。如果超过了，则表明server.lrulock折返了。

接下来看看如何计算，一个key的空闲时间
#### idletime的计算

```c
#define LRU_BITS 24
#define LRU_CLOCK_MAX ((1<<LRU_BITS)-1) /* Max value of obj->lru */
#define LRU_CLOCK_RESOLUTION 1000 /* LRU clock resolution in ms */
```
```c
//计算一个对象的idletime
unsigned long long estimateObjectIdleTime(robj *o) {
//获取当前的lruclock
    unsigned long long lruclock = LRU_CLOCK();
    //说明时钟没有折返，一直处于递增的状态
    if (lruclock >= o->lru) {
        return (lruclock - o->lru) * LRU_CLOCK_RESOLUTION;
    } else {
    //时钟折返
        return (lruclock + (LRU_CLOCK_MAX - o->lru)) *
                    LRU_CLOCK_RESOLUTION;
    }
}

```

下图呈现了计算的规则
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/de217c2ddae281f9a59f62c25b041954.png)

#### 为什么redis在获取时间戳的时候需要原子获取

```c
/*
如果当前分辨率低于我们刷新LRU时钟的频率(在生产服务器中应该是这样)，
我们返回预先计算的值，否则我们需要求助于系统调用。
*/
unsigned int LRU_CLOCK(void) {
    unsigned int lruclock;
    if (1000/server.hz <= LRU_CLOCK_RESOLUTION) {
    //原子获取之前存储过的lruclock的值
        atomicGet(server.lruclock,lruclock);
    } else {
    //server.hz配置的很低的情况下，lruclock来不及更新，通过系统调用直接获取。
        lruclock = getLRUClock();
    }
    return lruclock;
}
```
**redis是单线程的，为什么要使用原子操作获取lruclock**
 - redis的核心处理是单线程的，其实还有其它的异步线程在工作，比如大key的异步删除unlink；清空数据库flushdb async，flushall async[redis 4.0引入]；AOF的异步持久化。
 - 这些线程也要访问redis时钟，所以lruclock字段是需要支持多线程读写的。

从配置文件中找到的一些异步工作
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d48e5cc903c345f3e5a3f19fb2effbe7.png)


####  LRU淘汰策略
执行写操作时，发现内存超出"maxmemory"，执行一次LRU算法。

 - 随机采样出5个key
 - 淘汰掉最旧的，也即是最后一次访问时间戳最小的[空闲时间最大的]

看一下，配置文件"redis.conf"对此的介绍
>LRU, LFU and minimal TTL algorithms are not precise algorithms but approximated algorithms (in order to save memory), so you can tune it for speed or accuracy. For default Redis will check five keys and pick the one that was used less recently, you can change the sample size using the following configuration directive. The default of 5 produces good enough results. 10 Approximates very closely true LRU but costs more CPU. 3 is faster but not very accurate.
>maxmemory-samples 5


>LRU、LFU和最小TTL算法不是精确算法，而是近似算法(为了节省内存)，因可以对其进行速度或准确性调整。默认情况下，Redis会检查五个键，并选择最近使用较少的键，你可以使用以下配置指令更改样本大小。
默认值5可以产生足够好的结果。10非常接近真实的LRU，但需要更多的CPU。3比较快，但不是很准确。
>maxmemory-samples




### 近似LFU算法
LFU算法在Redis4.0中引入。
首先看一下，源码对于"lfu"的说明

```c
/* ----------------------------------------------------------------------------
 * LFU (Least Frequently Used) implementation.

 * We have 24 total bits of space in each object in order to implement
 * an LFU (Least Frequently Used) eviction policy, since we re-use the
 * LRU field for this purpose.
 *
 * We split the 24 bits into two fields:
 *
 *          16 bits      8 bits
 *     +----------------+--------+
 *     + Last decr time | LOG_C  |
 *     +----------------+--------+
 *
 * LOG_C is a logarithmic counter that provides an indication of the access
 * frequency. However this field must also be decremented otherwise what used
 * to be a frequently accessed key in the past, will remain ranked like that
 * forever, while we want the algorithm to adapt to access pattern changes.
 *
 * So the remaining 16 bits are used in order to store the "decrement time",
 * a reduced-precision Unix time (we take 16 bits of the time converted
 * in minutes since we don't care about wrapping around) where the LOG_C
 * counter is halved if it has an high value, or just decremented if it
 * has a low value.
 *
 * New keys don't start at zero, in order to have the ability to collect
 * some accesses before being trashed away, so they start at COUNTER_INIT_VAL.
 * The logarithmic increment performed on LOG_C takes care of COUNTER_INIT_VAL
 * when incrementing the key, so that keys starting at COUNTER_INIT_VAL
 * (or having a smaller value) have a very high chance of being incremented
 * on access.
 *
 * During decrement, the value of the logarithmic counter is halved if
 * its current value is greater than two times the COUNTER_INIT_VAL, otherwise
 * it is just decremented by one.
 * --------------------------------------------------------------------------*/
```

```c
/*
LOG_C是记录访问频率的对数计数器。
这个字段必须进行递减，否则过去经常访问的键将永远保持这样的排名。
为能够适应访问模式[lru  or  lfu]的变化，剩余的16位用于存储“衰退时间”，这是一种降低精度的Unix时间(以分钟为单位转换16位时间)，
其中LOG_C计数器如果值高则减半，如果值低则递减。
新键不是从零开始的，为了能够在被丢弃之前收集一些访问，所以它们从COUNTER_INIT_VAL开始。
在LOG_C上执行的对数增量在增加键时照顾到COUNTER_INIT_VAL，
因此从COUNTER_INIT_VAL开始(或具有更小的值)的键在访问时有很高的机会被增加。
在递减过程中，如果对数计数器的当前值大于COUNTER_INIT_VAL的两倍，则其值减半，否则仅递减1
*/
```

#### 为什么引入LFU算法

**LRU算法存在的问题**

 - 缓存污染。如果一个key长时间不被访问，突然被用户访问一下，LRU算法会误认为这是很"hot"的key，不容易被淘汰。

**LFU算法解决LRU存在的问题**

 - LFU，全称是"Least Frequently Used"。根据"key被使用的频率"来判断一个key是否是"hot key"。
 - LFU依据访问的频率，而不是只看访问的时间，更加精准地表示一个key的热度。

#### LFU字段

 - 上面介绍过lru字段存储在"redisObject"中；同样的，lfu字段也存储在"redisObject"中，只是相比与"lru"字段存储的内容不同。

LFU字段

 - 一共24bits。
 - 高16bit存储"last decrement time"最后一次衰退的时间。
 - 低8bit存储"logistic counter"逻辑计数器。
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/3bc21211eb26c1af21fa64f134ef1510.png)

 - logc是8个bit，存储访问频次。但是由于8个bit能表示的最大整数值有限为255，存储访问频次肯定不够用，所以这8个bit存储的是频次的对数值，并且这个值会随着时间衰减。值越小越容易被驱逐。为了确保新创建的对象不被驱逐，在lfu模式下会将新对象的这8bit初始化为一个大于0的值"LFU_INIT_VAL"，默认值是5。

```c
#define LFU_INIT_VAL 5
```

 - LDT是16bit，用来存储上一次logc的更新(衰退)的时间。因为只有16bit，所以精确度不是很高，取的是分钟时间戳对2^16^进行取模，大约每隔45天就折返。

**如何计算lfu模式下key的空闲时间**

```c
/*
以分钟为单位返回当前的时间，并且只取其中的16bit
*/
unsigned long LFUGetTimeInMinutes(void) {
    return (server.unixtime/60) & 65535;
}

/*
lfu模式下，计算key的idletime
*/
unsigned long LFUTimeElapsed(unsigned long ldt) {
//以分钟为单位获取当前的时间
    unsigned long now = LFUGetTimeInMinutes();
    //时间递增
    if (now >= ldt) return now-ldt;
    //时间折返
    return 65535-ldt+now;
}
```

示意图如下
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/9c56844848ed583cd012de88e1672822.png)

#### LFU淘汰策略

 **ldt 的值在以下情况下被记录和更新：**

 - 初次创建对象时。首先调用函数"createObject"创建对象。之后调用函数"initObjectLRUOrLFU"根据当前的内存淘汰策略是LRU还是LFU初始化对象的lru字段。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/4717c01c07b24a64b309c62df536955c.png)

 - 对象被访问时。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7bd0634bc5fd4a97870c6e23a80f7aa8.png)
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/01c42f90f3c945c29b0a99b77c955000.png)




接下来看看如何衰减的
[src\evict.c]

```c
int lfu_decay_time;             /* LFU counter decay factor. LFU计数器衰变因子*/
```
```c

/*
如果达到了对象的递减时间，则递减LFU计数器，
但不更新对象的LFU字段，当对象真正被访问时，我们以显式的方式更新访问时间和计数器。
根据相比与server.lfu_decay_time经过的时间将计数器减半。
返回对象频率计数器。
此函数用于扫描数据集以寻找最适合的对象:当我们检查候选对象时，如果需要，我们会逐渐减少扫描对象的计数器。
*/
unsigned long LFUDecrAndReturn(robj *o) {
//获取对象的LDT[高16bit]
    unsigned long ldt = o->lru >> 8;
    //获取对数计数器[低8bit]
    unsigned long counter = o->lru & 255;
    //根据衰减因子，计算对数计数器需要衰减的个数
    unsigned long num_periods = server.lfu_decay_time ? LFUTimeElapsed(ldt) / server.lfu_decay_time : 0;
    //衰减个数不为0，进行对数计数器的衰减
    if (num_periods)
        counter = (num_periods > counter) ? 0 : counter - num_periods;
        //返回衰减之后的对数计数器的值，但是此时并没有真正的修改lfu字段。
    return counter;
}

```

 - <font color='red'>logc的更新和LRU模式的lru字段更新是一样的，每次访问key时更新</font>
 - 更新并不是简单的进行加1操作，而是采用概率算法进行递增，因为logc记录的是频率的对数值。

```c
int lfu_log_factor;             /* LFU logarithmic counter factor. LFU对数增加因子*/
```
```c
/* Logarithmically increment a counter. The greater is the current counter value
 * the less likely is that it gets really implemented. Saturate it at 255. */
//以对数的方式增加计数器，值越大增加的可能性越小，最大值为255.
uint8_t LFULogIncr(uint8_t counter) {
	//计数器达到最大值，无需增加直接返回
    if (counter == 255) return 255;
    //生成一个随机数
    double r = (double)rand()/RAND_MAX;
    //为防止新创建的对象会因为counter的数值太小而被淘汰，
    //所以新创建对象counter的值是一个非零的初始值LFU_INIT_VAL,所以在计算的时候需要先减去这个初始值得到真正的counter
    double baseval = counter - LFU_INIT_VAL;
    //数值的矫正
    if (baseval < 0) baseval = 0;
    //根据对数增长因子计算计数器增长的值
    double p = 1.0/(baseval*server.lfu_log_factor+1);
    //概率增加计数器值
    if (r < p) counter++;
    //返回计数器的值
    return counter;
}

```
## 淘汰池优化
redis3.0在算法中增加了淘汰池进行优化，进一步提升了淘汰的效果。
淘汰池本质上是一个数组，在LRU模式下根据键值"idletime"由小到大的顺序存储过期key。
在LFU模式下，根据logc访问频率对数由小到大的顺序存储过期key。

在文件"evict.c"中

```c
/*
为了提高LRU近似的质量，我们取一组键(这些键是performEvictions函数中待删除的很好的候选者)
清除池中的条目按空闲时间idletime从小到大进行排序，将空闲时间较大的条目放在右边较小的放在左边。

当使用LFU策略时，使用反向频率[255-正向频率]指示而不是空闲时间，因此我们仍然
以较大的值驱逐键(较大的反向频率意味着以最不频繁的访问驱逐
键)。
*/
#define EVPOOL_SIZE 16 //淘汰池的大小
struct evictionPoolEntry {
    unsigned long long idle;    /* Object idle time (inverse frequency for LFU) */
    sds key;                    /* Key name. */
    sds cached;                 /* Cached SDS object for key name. */
    int dbid;                   /* Key DB number. */
};

static struct evictionPoolEntry *EvictionPoolLRU;
```

 接下来看一下如何向淘汰池中添加随机采样的expire key

```c
/*
这是performEvictions()的辅助函数，当内存达到上限想要删除一些key时，会用entries填充evictionPool。按照空闲时间进行升序排序。如果淘汰池未满则总是添加entry。已经满的情况下，大于当前键的最小空闲时间才能被插入淘汰池。
*/

void evictionPoolPopulate(int dbid, dict *sampledict, dict *keydict, struct evictionPoolEntry *pool) {
    int j, k, count;
    //存储随机采样的key
    //由配置文件可得，maxmemory_samples的默认值是5
    dictEntry *samples[server.maxmemory_samples];
    //从sampledict中随机采样出server.maxmemory_samples个key存储到samples中
    count = dictGetSomeKeys(sampledict,samples,server.maxmemory_samples);
    //准备将随机采样的key依次加入到evictpool中
    for (j = 0; j < count; j++) {
        unsigned long long idle;
        sds key;
        robj *o;
        dictEntry *de;

        de = samples[j];//获取第j个entry
        key = dictGetKey(de);//获取entry对应的key

        //如果内存淘汰策略不是ttl,
        if (server.maxmemory_policy != MAXMEMORY_VOLATILE_TTL) {
        //找到entry对应的key
            if (sampledict != keydict) de = dictFind(keydict, key);
            //找到value
            o = dictGetVal(de);
        }

        //计算空闲时间idle，idle越大淘汰效果越好
        if (server.maxmemory_policy & MAXMEMORY_FLAG_LRU) {
            idle = estimateObjectIdleTime(o);
        } else if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
            /*
            当我们使用LRU策略时，我们按空闲时间对键进行排
            序，以便我们从空闲时间较长的时间开始过期键。然
            而，当策略是LFU策略时，我们有一个频率估计，并且
            我们希望首先驱逐频率较低的键。因此，在池中，我
            们使用反向频率减去实际频率到最大频率255来放置对象。
            */
            //获取反向频率
            idle = 255-LFUDecrAndReturn(o);
        } else if (server.maxmemory_policy == MAXMEMORY_VOLATILE_TTL) {
            //在这种情况下越早过期越好，获取反向ttl
            idle = ULLONG_MAX - (long)dictGetVal(de);
        } else {
            serverPanic("Unknown eviction policy in evictionPoolPopulate()");
        }

        /*
        将entry插入evictpool中，
        1.找到第一个空的bucket或者已经被填充的bucket并且这个被填充bucket指向的entry的idletime小于即将插入entry的idletime[因为先淘汰idletime更大的entry]
        */
        k = 0;
        while (k < EVPOOL_SIZE &&
               pool[k].key &&
               pool[k].idle < idle) k++;
        if (k == 0 && pool[EVPOOL_SIZE-1].key != NULL) {
            //池子满了并且插入的entry的idletime小于池中第一个entry的idletime
            continue;
        } else if (k < EVPOOL_SIZE && pool[k].key == NULL) {
            //找到empty bucket插入即可
        } else {
            //k指向第一个“idle大于插入entry的idle的”entry
            if (pool[EVPOOL_SIZE-1].key == NULL) {
               //池子未满，将k~end指向的元素向后移动空出第k个位置，将待插入的entry插入k指向的位置
                sds cached = pool[EVPOOL_SIZE-1].cached;
                memmove(pool+k+1,pool+k,
                    sizeof(pool[0])*(EVPOOL_SIZE-k-1));
                pool[k].cached = cached;
            } else {
               //池子满了，插入第k-1的位置
                k--;
                /*
                将k-1~start+1的元素向前移动，空出第k-1个位置，将待插入的元素插入第k-1位置，同时也丢弃了第0位置的元素
                也就是idletime最小的元素，idletime越小越不容易被淘汰。
                */
                sds cached = pool[0].cached; /* Save SDS before overwriting. */
                if (pool[0].key != pool[0].cached) sdsfree(pool[0].key);
                memmove(pool,pool+1,sizeof(pool[0])*k);
                pool[k].cached = cached;
            }
        }

        /* Try to reuse the cached SDS string allocated in the pool entry,
         * because allocating and deallocating this object is costly
         * (according to the profiler, not my fantasy. Remember:
         * premature optimization bla bla bla. */
        int klen = sdslen(key);
        if (klen > EVPOOL_CACHED_SDS_SIZE) {
            pool[k].key = sdsdup(key);
        } else {
            memcpy(pool[k].cached,key,klen+1);
            sdssetlen(pool[k].cached,klen);
            pool[k].key = pool[k].cached;
        }
        pool[k].idle = idle;
        pool[k].dbid = dbid;
    }
}

```
总结

 - 池子满了，并且插入entry的idle小于等于池子中min_small_idle，不插入该entry
 - 找到empty bucket直接插入
 - k指向的位置不是empty bucket
 	池子未满，将[k,end]移动到位置[k+1,end+1]空出位置k
 	池子满了，将[1,k]移动到位置[0,k-1]空出第k-1个位置，插入新的entry，丢弃掉池子中最小idle的entry

**总结为一句话就是：池子未满直接加入；池子满了：若idle小于等于淘汰池中最小的idle则不加入反之先丢弃掉最小idle的entry然后加入新的entry**

