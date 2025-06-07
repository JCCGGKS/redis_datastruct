
本篇文章基于redis7.22
源码位置：redis-7.2.2\src\t_zset.c
有关skiplist的介绍可以参考：[skiplist](https://blog.csdn.net/weixin_46290302/article/details/125513113?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522169813370516800227475389%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=169813370516800227475389&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-1-125513113-null-null.nonecase&utm_term=skiplist&spm=1018.2226.3001.4450)
# Zset
Zset是redis五种基本数据类型之一。底层有两种编码"OBJ_ENCODING_SKIPLIST"和"OBJ_ENCODING_ZIPLIST(新版本中ziplis已经被listpack取代)"
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/f85cc259b99f08a10d0a2d8de5387616.png#pic_center)

这篇文章只介绍编码为"OBJ_ENCODING_SKIPLIST"的底层实现，即Zset，先来看一下zset的结构


```c
/* ZSETs use a specialized version of Skiplists */
//ZSET使用一种特殊版本的skiplist
typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```
**为什么zset结构中要包含一个dict？**

 - 使用redis命令的时候，可以发现跟zset有关的命令有"zadd key score value"、"zrange key start stop withscores"、"zincrby key field value"....
 - 在获取zset相关信息时只需要输入value的相关值即可并不需要该value对应的score值，但是在实际操作时却需要用到score值，因为zset本身就是按照score值排序的。所以将value到score的映射关系存到了dict，这样可以方便快速的通过value找到与之对应的score值。
 - 哈希表[存储redis object到score之间的映射]另一种是skiplist[存储score到redis object之间的映射]

# zset
首先看一下源码中，对于zset的介绍

>ZSETs are ordered sets using two data structures to hold the same elements in order to get O(log(N)) INSERT and REMOVE operations into a sorted data structure.

>zset是有序集合，使用两种数据结构存储相同的elements，以此保证插入和删除的时间复杂度为O(logn)；

>The elements are added to a hash table mapping Redis objects to scores.
At the same time the elements are added to a skiplist mapping scores to Redis objects (so objects are sorted by scores in this "view").

><font color='red'>hash table用来存储“redis object”到“score”之间的映射，skiplist存储“score”到"redis object"之间的映射</font>，所以在skiplist中按照score进行排序

>Note that the SDS string representing the element is the same in both the hash table and skiplist in order to save memory. What we do in order to manage the shared SDS string more easily is to free the SDS string only in zslFreeNode(). The dictionary has no value free method set.So we should always remove an element from the dictionary, and later from
the skiplist.

>skiplist和hash table中的表示元素的sds是同一个，这样做可以节省内存；为了更简单的管理共享的sds字符串，只在zslFreeNode()函数中释放内存；字典没有设置值释放方法，所以应该先从字典中删除元素，再从skiplist中删除元素。

>This skiplist implementation is almost a C translation of the original algorithm described by William Pugh in "Skip Lists: A Probabilistic Alternative to Balanced Trees", modified in three ways:
a) this implementation allows for repeated scores.
b) the comparison is not just by key (our 'score') but by satellite data.
c) there is a back pointer, so it's a doubly linked list with the back pointers being only at "level 1". This allows to traverse the list from tail to head, useful for ZREVRANGE. 

>这个跳表的实现，几乎是对“William Pugh 在 "Skip Lists: A Probabilistic Alternative to Balanced Trees",中描述的原始算法的C语言翻译，并作出了以下三个修改
>1.skiplist允许插入重复的score
>2.skiplist,排序的第一关键字是"score"，第二关键字是“satellite data”也就是原始数据value值
>skiplist的实现中包含一个backward后退指针(只有最后一层才有)，为了能够实现从后往前遍历，对应的命令是"zrevrange"





# Data_Struct
## zskiplist
看"zskiplist"的结构之前，先回顾一下"single LinkedList单链表"的结构
```c
typedef ListNode struct{
	int val;
	struct ListNode *next;
}ListNode;

typedef SingleList struct{
	struct ListNode *head;
	struct ListNode *tail;
}SingleList;
```

其实"zskiplist"的本质就是链表

**zskiplist**

```c
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;//表示元素的个数，所以跳表获取元素个数的时间复杂度为O(1),等下我们再从源码中进行验证
    int level;//表示跳表的层数
} zskiplist;
```

 - 通过源码可以发现，zskiplist就是由zkiplistNode组成的包含有头尾指针的一个链表，并且还存储这个链表中的元素个数以及当前的总层数。


**zskiplistNode**
```c
typedef char *sds;
typedef struct zskiplistNode {
	//命令[zadd key score member [score member..]]
	//存储命令中的member值，
    sds ele;
    //存储命令中的score值
    double score;
    //最后一层的后退指针，为了实现从后往前遍历，对应命令zrevrange
    struct zskiplistNode *backward;
    //下面详细解释
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;
```
我们单独看一下zskiplistNode中的“level数组”是什么；
拆开来看，先看最里面的结构，

```c
struct zskiplistNode * forward;
//这个结构是不是很类似于单链表中的next指针
//struct ListNode *next;
```
然后看外面一层的结构

```c
struct zskiplistLevel {
        struct zskiplistNode *forward;
    };
//定义了一个结构体类型zskiplistLevel，这个结构体中包含一个字段zskiplistNode*类型的forward指针，相当于对单链表的next指针做了一层包装，将其放入一个结构体中。
```
接着往下看

```c
struct zskiplistLevel {
        struct zskiplistNode *forward;
    } level[];
//用刚刚定义好的结构体定义了zskiplistLevel类型的数组level[也即是定义结构体的同时定义变量]。
//跳表相对于单链表来说多了好几层，每一层都相当于一个单链表[最底层相当于一个双向链表]，所以每一层都需要一个类似next指针的forward指针
```
如果你还不明白，再解释的仔细一点，来看看C语言中结构体定义变量的方式

```c
//1.结构体变量的定义，放在结构体声明之后
struct student{
	int id;
	char sex;
	char name[23];
};
struct student stu[];
//2.结构体声明的同时定义变量，
struct student{
	int id;
	char sex;
	char name[23];
}stu[];
//3.匿名方式定义变量，但是只能定义一次
struct {
	int id;
	char sex;
	char name[23];
}stu[];
//定义数组的时候可以指定长度也可以不指定长度
int arr[3];
int arr[];//两种方式都是可以的
```
好的，接下来继续看看zskiplistNode里面还有什么

```c
struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
```
还有zskiplistLevel中的span没有介绍
先透露一下，这个<font color='red'>span是用来记录当前节点到下一个节点经历的跨度，有了这个字段可以很快的得出当前节点的排名，怎么做到的呢？就是在寻找目标节点的过程中，将遇到节点的span加起来就是目标节点的rank，对应的命令就是[zrank key member ].</font>

为了加深对"zskiplist"结构的理解，我画了一张示意图，如下所以
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d761970e5405ce6f39a480d85254a7d9.png)

<center>zskiplist结构示意图</center>

有关skiplist的结构定义就介绍完了接下来介绍有关skiplist的一些函数。介绍相关函数之前再来总结一下skiplist的结构：
其实跳表的本质，是一个拥有多层，每一层都是一个单链表，最底层是一个双链表的链表结构
# Function
逐个分析有关跳表的各个函数
## zslCreateNode

```c
//利用所给的节点的level,以及score值和ele值[也就是对应命令中的member]创建一个新的节点
zskiplistNode *zslCreateNode(int level, double score, sds ele) {
	//分配内存，这里分配了两部分的内存，一部分是节点本身的内存，也就是为ele,score,backward
	//另一部分是为forward和span也就是level数组分配内存
    zskiplistNode *zn =
        zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    zn->score = score;
    zn->ele = ele;
    return zn;
}
```
## zslCreate
创建一个新的skiplist

```c
#define ZSKIPLIST_MAXLEVEL 32 
#define ZSKIPLIST_P 0.25  
```

```c
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;
	//为zskiplist本身的结构分配内存
    zsl = zmalloc(sizeof(*zsl));
    //将skiplist的层数设置为1
    zsl->level = 1;
    //元素个数设置为0
    zsl->length = 0;
    //初始化头结点,ZSKIPLIST_MAXLEVEL==32
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    //初始化level中的forward
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    //初始化头节点的backward指针
    zsl->header->backward = NULL;
    //设置尾节点为null
    zsl->tail = NULL;
    return zsl;
}
```
## zslFreeNode
//释放zskiplistNode节点
```c
void zslFreeNode(zskiplistNode *node) {
	//先释放sds指向的字符串
    sdsfree(node->ele);
    //释放node本身
    zfree(node);
}
```
## zslFree
//释放zskiplist
```c
void zslFree(zskiplist *zsl) {
    zskiplistNode *node = zsl->header->level[0].forward, *next;
	//释放头节点
    zfree(zsl->header);
    //依次释放每个节点
    while(node) {
        next = node->level[0].forward;
        zslFreeNode(node);
        node = next;
    }
    //释放zslskiplist
    zfree(zsl);
}
```
这里解释一下，为什么只是释放，“node->level[0].forward”而不是遍历每一层，进行释放；因为最底层也就是0层包含所有的节点，我们在插入一个节点的时候，将其插入在了第0层，上层只是拥有该节点的相应层的forward指针。
## zslRandomLevel

```c
//为即将插入的节点生成一个随机level，表示该节点所拥有的层数
//level处于[1,ZSKIPLIST_MAXLEVEL(32)]之间
//类似于幂律分布，不会产生太大level，这样一来所有节点的层数都能控制在一个相对平衡的状态
int zslRandomLevel(void) {
	//
    static const int threshold = ZSKIPLIST_P*RAND_MAX;
    int level = 1;
    while (random() < threshold)
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```
这是在redis7.2.2版本内的代码，但是我没有找到“RAND_MAX”,我们来看一下redis_3.2.100版本的代码

```c
/*
	random()返回的是一个随机数
	0xFFFF转换为二进制是“1111 1111 1111 1111”十进制是65535
	random()&0xFFFF可以将随机数的高位清0，只留下低16位，也就是生成了一个16位的随机数
	(random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF)这行代码的意思就是：
	生成的一个16位的随机数小于(ZSKIPLIST_P * 0xFFFF)的概率，16位随机数的取值在[1,0xFFFF], (ZSKIPLIST_P * 0xFFFF)的取值在[1,(ZSKIPLIST_P * 0xFFFF)],用古典概率计算，
	p=(ZSKIPLIST_P * 0xFFFF)/0xFFFF == ZSKIPLIST_P 
	
	感叹一下这个算法真的巧妙
*/
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```
>随机算法是一个概率算法，很大程度上能保证skiplist处于一个相对平衡的状态，但是在极端情况下仍然可能退化为单链表。
>随机算法是skiplist实现的关键，在插入一个新的节点的之前，需要先调用随机函数生成新节点所在的level，然后根据level创建新的节点，之后寻找插入位置并插入。
>一个节点的level在插入之前就已经决定好了，被删除之前都不会发生变化。

## zslInsert

```c
//向skiplist中插入一个newnode
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
	//记录每一层插入位置的前一个节点
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    
    //暂时还不知道这个rank记录的是什么，我们先向下看
    unsigned long rank[ZSKIPLIST_MAXLEVEL];
    
    int i, level;
	//检查score是否合法，如果score not a number则会引发错误，程序停止运行
    serverAssert(!isnan(score));
    //从头结点出发
    x = zsl->header;
    //从最高层开始向下寻找
    for (i = zsl->level-1; i >= 0; i--) {
        //初始化rank,如果i指向最高一层，则将rank初始化为0
        //i指向非最高层初始化为上一层的rank；仔细一想会发现所有的层rank其实都初始化为了0，rank[i]=0
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        //如果当前节点不为空，并且当前节点的Score值小于目标score值继续向下一个节点探寻；第二种情况是，当前节点不为空并且当前节点的Score值等于目标score值，但是当前节点的ele小于目标ele，继续探寻下一个节点。[这是为了保证score值相等的时候，按照ele进行排序]；
        //sdscmp(s1,s2)s1<s2返回-1，s1==s2返回0，s1>s2返回1
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
        	/*
        		x->level[i].span表示的是第i层，x节点到下一个节点之间的跨度
        		从这里可以推断出rank[i]记录的是第i层，目标节点之前所有节点的span的总和；也就是目标节点在当前层的排名
        		
        	*/
            rank[i] += x->level[i].span;
            //继续探寻下一个节点
            x = x->level[i].forward;
        }
        //寻找到插入位置，将插入位置的前一个节点存储到update[i]，即将newnode插入到update[i]后
        update[i] = x;
    }
    //score值允许相同，但是必须要确保ele不相同；重新插入相同的ele是永远不会发生的，因为插入之前会在hashTable中进行测试该ele是否存在
    //获取插入节点的随机level，表示该节点的层级
    level = zslRandomLevel();
    //节点的随机level大于当前skiplist的level
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
        	//初始化rank
            rank[i] = 0;
            //这一层，将该节点插入到head后面
            update[i] = zsl->header;
            //初始化span，从这里也可以看出span表示的是当前节点到下一个节点之间的跨度，
            //因为此时的第i层并没有实际的节点，先将其初始化为zsl->length
            update[i]->level[i].span = zsl->length;
        }
        //更新skiplist的level
        zsl->level = level;
    }
    //根据level,score,ele创建新的节点
    x = zslCreateNode(level,score,ele);
    //执行插入操作
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        //插入新节点更新span
        
        //rank[0]记录的是第0层目标节点之前的所有节点span总和，	
        //rank[i]记录的是第i层目标节点之前的所有节点的span总和；
        //rank[0]-rank[i]表示的就是第i层，x节点距离update[i]->level[i]节点之间的span，
        //那update[i]->level[i].span - (rank[0] - rank[i])记录的就是x节点到update[i].level[i].forward节点之间的跨度，也就是x->level[i].span.
        
        
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        
                                                                                                                                                                          //update[i]-level[i].forward由于新节点x的插入而被改变，所以需要修改
        //这里的加1表示的是，算上新插入的节点x
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }
	//新增节点span增加，
	//虽然说newnode并没有插入这一层，但是我们在寻找节点的时候是从最上层往下开始寻找的；
	//由于新结点没有插入这一层，所以从最高层只需向下走一步就可以找到该节点
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

	//更新backward;update[0]==zsl->header表明当前节点是一个节点，将backward设置为null
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    //如果x的level数组不为空，更新x->level[0].forward的backward
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
    	//x是最后一个节点
        zsl->tail = x;
    //元素个数加1
    zsl->length++;
    return x;
}
```

## zslDeleteNode

```c
  /*
 	被zslDelete,zslDeleteRangeByScore,zslDeleteRangeByRank
 	使用的内部函数
 */
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    //update存储的是每一层预删除节点的前一个节点
    //根据update对每一层执行删除操作
    for (i = 0; i < zsl->level; i++) {
    	//只有当前层的update指向的下一个节点是预删除节点才会执行删除操作
        if (update[i]->level[i].forward == x) {
        //更新span
        /*
            	明明是删除节点为什么span要累加呢，这样不是越加越大。
            	因为删除update[i].level[i]之后的节点会使update[i].level[i]指向的下一个节点有所变化，删除的节点越多，距离下一个节点越远，所以要相加。
            	
            	
            */
            update[i]->level[i].span += x->level[i].span - 1;
            //删除操作
            update[i]->level[i].forward = x->level[i].forward;
        } else {
        //x节点被删除，span减1
            update[i]->level[i].span -= 1;
        }
    }
    //修改backward
    //x节点有后继节点
    if (x->level[0].forward) {
        x->level[0].forward->backward = x->backward;
    } else {
    //x节点没有后继节点，x是尾节点，删除之后需要修改tail
        zsl->tail = x->backward;
    }
    //修改zsl->level，因为在删除的过程中有可能会删除某一层的所有节点导致那一层变为空，
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;
     //元素个数减1
    zsl->length--;
}
```
## zslDelete

```c
/*
	根据传入的score和ele删除匹配的节点[为什么要传入score和ele呢，因为score可以相等，所以此时需要根据ele进行判断]
	节点被找到并删除返回1，其它情况返回1
*/
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
	//update记录的是每一层删除目标节点的前一个节点
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
    //这个逻辑和zslInsert那里的逻辑一样
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    
     /*
     	因为有的元素的score值会重复，所以需要根据score和ele找到正确元素对象
     */
    x = x->level[0].forward;
    //如果节点的score和ele都和目标的score和ele匹配则说明找到目标节点
    if (x && score == x->score && sdscmp(x->ele,ele) == 0) {
    	//执行删除操作
        zslDeleteNode(zsl, x, update);
        //如果传进来的node为空，释放节点x
        if (!node)
            zslFreeNode(x);
        else
        	//将目标节点赋值给node，并传出便于调用者重用该节点
            *node = x;
        return 1;
    }
    return 0;//element没有找到
}
```
## zslGetRank
为了更好的理解zslskiplistlevel中的span我们来看一下zslRank是如何通过span获取当前节点的排名的

```c
/*
	根据传入的score和ele找到匹配的节点，返回其排名。没有匹配到节点返回0；排名从1开始，因为zsl->header的rank为0
*/
unsigned long zslGetRank(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *x;
    //排名从0开始计算
    unsigned long rank = 0;
    int i;

    x = zsl->header;
    /*
			这个地方和"insert、delete、update"的逻辑不太一样，"insert\delete\update"这里的判断逻辑是
		”cur.level[i].forward.key < key“而getrank这里的逻辑是”cur.level[i].forward.key <= key“
		首先呢span记录的是当前节点到下一个节点之间的跨度，所以计算当前节点的排名，需要将当前节点之前节点的span相加
		所以当”cur.level[i].forward.key == key“时，此时"cur.level[i].forward"是当前节点的前一个节点，还需要加上
		cur.level[i].forward.span才对
		*/
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                sdscmp(x->level[i].forward->ele,ele) <= 0))) {
            /*
            	rank记录的就是当前层目标节点之前所有节点的span总和
            	即目标节点的排名
            */
            rank += x->level[i].span;
            //更新下一个节点
            x = x->level[i].forward;
        }

        //x有可能指向zsl->head,suoyi
        if (x->ele && x->score == score && sdscmp(x->ele,ele) == 0) {
            return rank;
        }
    }
    return 0;//没有找到目标节点
}
```

## zslGetElementByRank
```c
/* Finds an element by its rank. The rank argument needs to be 1-based. */
//根据元素的rank查找元素，rank以1为基础。
zskiplistNode* zslGetElementByRank(zskiplist *zsl, unsigned long rank) {
    zskiplistNode *x;
    /*
    记录扫描到节点的span之和,可以得到当前节点的rank
    然后与被给的rank进行比较，
    以此判断是否找到排名为rank的zskiplistNode
    */
    unsigned long traversed = 0;
    int i;

    x = zsl->header;
    //从上往下进行遍历查找
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward && (traversed + x->level[i].span) <= rank)
        {
            traversed += x->level[i].span;
            x = x->level[i].forward;
        }
        //退出while循环的条件有两个
        /*
        	1.span总和traversed等于传入的参数rank
        	2.span总和traversed大于传入的参数rank
        */
        //如果此时的travsered等于rank，说找到排名为rank的节点，作为结果返回该zskiplistNode节点。
        if (traversed == rank) {
            return x;
        }
    }
    //没有找到返回NULL
    return NULL;
}
```

## zslUpdateScore

```c
 /*
 	在skiplist中更新一个对象的score值[这个函数无法更新hashatable中的对象的score]，并返回被更新的对象；
 	需要注意的是如果score修改之后引起位置的变化需要先删除后重新插入，反之直接修改即可
 */
zskiplistNode *zslUpdateScore(zskiplist *zsl, double curscore, sds ele, double newscore) {
	//update数组记录更新节点的前一个节点
	//记录前一个节点的目的就在于，
	//如果score的修改引起位置的变化，便于将当前节点删除
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

 
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
    	//与zslInsert一样的逻辑
        while (x->level[i].forward &&
                (x->level[i].forward->score < curscore ||
                    (x->level[i].forward->score == curscore &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;
    }


    x = x->level[0].forward;
    //如果没有匹配到节点则报错
    serverAssert(x && curscore == x->score && sdscmp(x->ele,ele) == 0);

   //修改完score之后位置不发生变化，直接修改即可无需额外的操作
    if ((x->backward == NULL || x->backward->score < newscore) &&
        (x->level[0].forward == NULL || x->level[0].forward->score > newscore))
    {
        x->score = newscore;
        return x;
    }

    //修改之后位置发生变化，需要先进行删除后重新插入
    zslDeleteNode(zsl, x, update);
    zskiplistNode *newnode = zslInsert(zsl,newscore,x->ele);
    //将old节点的ele置为null然后再将其释放，因为此时新节点与old节点的ele指向的是同一个，
    //不置为空就释放的话，会将新结点指向的ele也释放掉
    x->ele = NULL;
    zslFreeNode(x);
    return newnode;
}

```

## zsetLength
看一下对于zset来说获取元素个数的时间复杂度为多少

```c
//获取zset对象的元素个数
unsigned long zsetLength(const robj *zobj) {
    unsigned long length = 0;
    //根据不同的编码方式选择不同的算法
    //编码方式为listpack
    if (zobj->encoding == OBJ_ENCODING_LISTPACK) {
        length = zzlLength(zobj->ptr);
        //编码方式为skiplist，直接从字段length中获取
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        length = ((const zset*)zobj->ptr)->zsl->length;
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    return length;
}
//编码方式为listpack[redis7.2.2]
unsigned int zzlLength(unsigned char *zl) {
    return lpLength(zl)/2;
}
//如果此时listpack的元素个数可以用2bytes表示，直接读取记录长度字段的值
//不然的话，需要遍历该listpack确认具体的元素个数
unsigned long lpLength(unsigned char *lp) {
    uint32_t numele = lpGetNumElements(lp);
    //#define LP_HDR_NUMELE_UNKNOWN UINT16_MAX
    if (numele != LP_HDR_NUMELE_UNKNOWN) return numele;

    /* Too many elements inside the listpack. We need to scan in order
     * to get the total number. */
    uint32_t count = 0;
    unsigned char *p = lpFirst(lp);
    while(p) {
        count++;
        p = lpNext(lp,p);
    }

    /* If the count is again within range of the header numele field,
     * set it. */
    //如果查询到的元素个数在范围内，可以再次将记录长度字段的设置为具体的元素个数
    if (count < LP_HDR_NUMELE_UNKNOWN) lpSetNumElements(lp,count);
    return count;
}
//ziplist也是一样的，如果元素个数没超过UINT16_MAX就直接读取字段记录的值，否则遍历计算
//编码方式为skiplis，直接从字段中获取
length = ((const zset*)zobj->ptr)->zsl->length;
```
综上呢，编码方式为skiplist获取长度的时间复杂度为O(1);
编码方式为listpack[或者ziplist]获取长度的时间复杂度为O(1)[元素的个数<uint16_max]或者O(n)[元素的个数>=uint16_max]
## zsetScore

```c
//根据sds存储的值，寻找与之对应的score
int zsetScore(robj *zobj, sds member, double *score) {
    if (!zobj || !member) return C_ERR;
	
    //编码方式为listpack
    if (zobj->encoding == OBJ_ENCODING_LISTPACK) {
        if (zzlFind(zobj->ptr, member, score) == NULL) return C_ERR;
        //编码方式为skiplist 
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        zset *zs = zobj->ptr;
        //首先在zset中的hashtable中寻找该member
        dictEntry *de = dictFind(zs->dict, member);
        if (de == NULL) return C_ERR;
        //找到对象获取score值
        *score = *(double*)dictGetVal(de);
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    return C_OK;
}
//编码方式为listpack

//编码方式为skiplist
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he;
    uint64_t h, idx, table;

    if (dictSize(d) == 0) return NULL; /* dict is empty */
   	//这里涉及到hashtable的渐进式rehash，下一篇文章详细介绍
    if (dictIsRehashing(d)) _dictRehashStep(d);
    //获取key对应的hash值
    h = dictHashKey(d, key);
    //dict中的哈希表有两个table[0]是平常使用的表，table[1]是发生rehash时使用的表
    for (table = 0; table <= 1; table++) {
        //根据hash获取其在一维数组hashtable中的位置
        idx = h & DICTHT_SIZE_MASK(d->ht_size_exp[table]);
        //获取hashtable中下标为idx的第一个元素
        he = d->ht_table[table][idx];
        while(he) {
            //由于采用的拉链法解决哈希冲突，所以需要沿着next一路向后寻找
            void *he_key = dictGetKey(he);
            //找到目标返回
            if (key == he_key || dictCompareKeys(d, key, he_key))
                return he;
            //寻找下一个
            he = dictGetNext(he);
        }
        //如果当前没有rehash则表明table[1]中没有元素，退出即可
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL;
}
```

## zsetRank

```c
/*
	排名从0开始，节点不存在返回-1.
	如果reverse为non-zero表明需要返回倒序排名
*/
long zsetRank(robj *zobj, sds ele, int reverse, double *output_score) {
    unsigned long llen;
    unsigned long rank;

    //获取zset的元素个数
    llen = zsetLength(zobj);

    //编码方式为listpack
    if (zobj->encoding == OBJ_ENCODING_LISTPACK) {
        unsigned char *zl = zobj->ptr;
        unsigned char *eptr, *sptr;

        eptr = lpSeek(zl,0);
        serverAssert(eptr != NULL);
        sptr = lpNext(zl,eptr);
        serverAssert(sptr != NULL);

        rank = 1;
        while(eptr != NULL) {
            if (lpCompare(eptr,(unsigned char*)ele,sdslen(ele)))
                break;
            rank++;
            zzlNext(zl,&eptr,&sptr);
        }

        if (eptr != NULL) {
            if (output_score) 
                *output_score = zzlGetScore(sptr);
            if (reverse)
                return llen-rank;
            else
                return rank-1;
        } else {
            return -1;
        }
        //编码方式为skiplist
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        zset *zs = zobj->ptr;
        zskiplist *zsl = zs->zsl;
        dictEntry *de;
        double score;
		//首先通过hashtable寻找该ele对应的key-value是否存在
        de = dictFind(zs->dict,ele);
        if (de != NULL) {
            //key-value存在
            //在hashtable找到ele对应的score
            score = *(double*)dictGetVal(de);
            //根据score和ele获取rank
            rank = zslGetRank(zsl,score,ele);
            /* Existing elements always have a rank. */
            serverAssert(rank != 0);
            if (output_score)
                *output_score = score;
            if (reverse)
                //返回倒序排名
                return llen-rank;
            else
                //正序排名
                return rank-1;
        } else {
            //key-value不存在直接返回-1
            return -1;
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
}
```

## zsetAdd

```c
/*
在已排序的集合中添加新元素或更新现有元素的分数，而不考虑其编码。标志集改变命令的行为。输入标志如下:
ZADD_INCR:以'score'增加当前元素的分数，而不是更新当前元素的分数。如果该元素不存在，则假定前一个分数为0。
ZADD_NX:仅当该元素不存在时执行该操作。
ZADD_XX:仅当元素已经存在时才执行该操作。
ZADD_GT:仅当新分数大于当前分数时，才对现有元素执行操作。
ZADD_LT:仅当新分数小于当前分数时，才对现有元素执行操作。当使用ZADD_INCR时，如果'newscore'不为NULL，则元素的新分数存储在newscore'中。返回的标志如下:
ZADD_NAN:结果分数不是一个数字。
ZADD_ADDED:添加了元素(在调用之前不存在)。
ZADD_UPDATED:元素得分已更新。
ZADD_NOP:由于NX或XX，未执行任何操作。
返回值:函数在成功时返回1，并设置适当的标志ADDED或UPDATED来表示操作期间发生了什么(注意，如果我们使用相同的分数重新添加元素，则不能设置任何标志)
(在这种情况下使用零增量)。
该函数在错误时返回0，目前仅当增量产生NAN条件时，或者当'score'值从一开始就是NAN时。该命令作为添加新元素的副作用可能会将已排序集的内部编码从listpack转换为hashtable+skiplist。'ele'的内存管理:该函数不获取'ele' SDS字符串的所有权，但在需要时复制它。
*/
//向zset中添加一个新的score-ele
int zsetAdd(robj *zobj, double score, sds ele, int in_flags, int *out_flags, double *newscore) {
    //将选项转化为易于检查的变量
    /* Turn options into simple to check vars. */
    int incr = (in_flags & ZADD_IN_INCR) != 0;
    int nx = (in_flags & ZADD_IN_NX) != 0;
    int xx = (in_flags & ZADD_IN_XX) != 0;
    int gt = (in_flags & ZADD_IN_GT) != 0;
    int lt = (in_flags & ZADD_IN_LT) != 0;
    *out_flags = 0; /* We'll return our response flags. */
    double curscore;

    /* NaN as input is an error regardless of all the other parameters. */
    //score is not a number,设置response flags为ZADD_OUT_NAN 
    if (isnan(score)) {
        *out_flags = ZADD_OUT_NAN;
        return 0;
    }
	
    //根据编码选用不同的策略
    /* Update the sorted set according to its encoding. */
    if (zobj->encoding == OBJ_ENCODING_LISTPACK) {
        unsigned char *eptr;
		//在listpack中寻找ele，eptr!=NULL表明找到
        if ((eptr = zzlFind(zobj->ptr,ele,&curscore)) != NULL) {
            /* NX? Return, same element already exists. */
            //nx表明只有当元素不存在时插入，但现在元素存在故不做任何操作
            if (nx) {
                *out_flags |= ZADD_OUT_NOP;
                return 1;
            }

            /* Prepare the score for the increment if needed. */
            //增加score
            if (incr) {
                score += curscore;
                //验证score是否是一个number
                if (isnan(score)) {
                    *out_flags |= ZADD_OUT_NAN;
                    return 0;
                }
            }

            /* GT/LT? Only update if score is greater/less than current. */
            //不满足lt,gt的要求，不做任何操作
            if ((lt && score >= curscore) || (gt && score <= curscore)) {
                *out_flags |= ZADD_OUT_NOP;
                return 1;
            }
            //newscore不为null，将score的值存储在newscore中
            if (newscore) *newscore = score;
			
            //当score发生变化先删除后插入
            /* Remove and re-insert when score changed. */
            if (score != curscore) {
                //先删除
                zobj->ptr = zzlDelete(zobj->ptr,eptr);
                //后更新
                zobj->ptr = zzlInsert(zobj->ptr,ele,score);
                //将标志设置为更新"ZADD_OUT_UPDATED"
                *out_flags |= ZADD_OUT_UPDATED;
            }
            return 1;
        } else if (!xx) {
            /* check if the element is too large or the list
             * becomes too long *before* executing zzlInsert. */
            if (zzlLength(zobj->ptr)+1 > server.zset_max_listpack_entries ||
                sdslen(ele) > server.zset_max_listpack_value ||
                !lpSafeToAdd(zobj->ptr, sdslen(ele)))
            {
                //达到阈值上限，更换编码方式为"skiplist"
                zsetConvertAndExpand(zobj, OBJ_ENCODING_SKIPLIST, zsetLength(zobj) + 1);
            } else {
                //没有达到上限在listpack中插入
                zobj->ptr = zzlInsert(zobj->ptr,ele,score);
                if (newscore) *newscore = score;
                //将标志设置为"added"
                *out_flags |= ZADD_OUT_ADDED;
                return 1;
            }
        } else {
            *out_flags |= ZADD_OUT_NOP;
            return 1;
        }
    }

    //编码方式为skiplist
    if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        zset *zs = zobj->ptr;
        zskiplistNode *znode;
        dictEntry *de;
		//在哈希表中找到ele对应的对象
        de = dictFind(zs->dict,ele);
        //找到该对象
        if (de != NULL) {
            /* NX? Return, same element already exists. */
            if (nx) {
                *out_flags |= ZADD_OUT_NOP;
                return 1;
            }

            curscore = *(double*)dictGetVal(de);

            /* Prepare the score for the increment if needed. */
            if (incr) {
                score += curscore;
                if (isnan(score)) {
                    *out_flags |= ZADD_OUT_NAN;
                    return 0;
                }
            }

            /* GT/LT? Only update if score is greater/less than current. */
            if ((lt && score >= curscore) || (gt && score <= curscore)) {
                *out_flags |= ZADD_OUT_NOP;
                return 1;
            }

            if (newscore) *newscore = score;

            /* Remove and re-insert when score changes. */
            if (score != curscore) {
                //在skiplist中更新
                znode = zslUpdateScore(zs->zsl,curscore,ele,score);
                
                //在hashtable中更新
                dictSetVal(zs->dict, de, &znode->score); /* Update score ptr. */
                *out_flags |= ZADD_OUT_UPDATED;
            }
            return 1;
        } else if (!xx) {
            //插入新对象
            ele = sdsdup(ele);
            //在skiplist中插入
            znode = zslInsert(zs->zsl,score,ele);
            //在哈希表中插入
            serverAssert(dictAdd(zs->dict,ele,&znode->score) == DICT_OK);
            *out_flags |= ZADD_OUT_ADDED;
            if (newscore) *newscore = score;
            return 1;
        } else {
            *out_flags |= ZADD_OUT_NOP;
            return 1;
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    return 0; /* Never reached. */
}

```
编码方式为listpack
    先在listpack中寻找到与ele对应的对象，存在的话更新
    不存在的话，插入新对象之前预检查插入之后会不会达到阈值上限，到达的话更换编码方式为"hashtable+listpack"
编码方式为skiplist
    现在哈希表中寻找该ele对应的对象，存在的话更新，同时更新skiplist和hashtable
    不存在的话，插入skiplist和hashtable，从这一步也可以看出skiplist和hashtable中的ele指向同一个数据

## zsetDel

```c
//在zset中删除ele对象
int zsetDel(robj *zobj, sds ele) {
    //编码方式为listpack
    if (zobj->encoding == OBJ_ENCODING_LISTPACK) {
        unsigned char *eptr;
		//找到并删除
        if ((eptr = zzlFind(zobj->ptr,ele,NULL)) != NULL) {
            zobj->ptr = zzlDelete(zobj->ptr,eptr);
            return 1;
        }
        //编码方式为skiplist
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        zset *zs = zobj->ptr;
        if (zsetRemoveFromSkiplist(zs, ele)) {
            //删除元素之后导致hashtable需要改变容量[扩容或者缩容],为其重新分配合适的内存
            if (htNeedsResize(zs->dict)) dictResize(zs->dict);
            return 1;
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    return 0; /* No such element found. */
}
//编码方式为skiplist
static int zsetRemoveFromSkiplist(zset *zs, sds ele) {
    dictEntry *de;
    double score;
	
    //在dict中找到ele对应的对象，并将其ulink
    de = dictUnlink(zs->dict,ele);
    if (de != NULL) {
        /* Get the score in order to delete from the skiplist later. */
        score = *(double*)dictGetVal(de);

        /* Delete from the hash table and later from the skiplist.
         * Note that the order is important: deleting from the skiplist
         * actually releases the SDS string representing the element,
         * which is shared between the skiplist and the hash table, so
         * we need to delete from the skiplist as the final step. */
        //先从哈希表中删除后从skiplist中删除，这个顺序不可发生变化，因为在skiplist
        //中删除会引起节点的释放"zslDelete---zslDeleteNode---zslFreenode(x)"
        dictFreeUnlinkedEntry(zs->dict,de);

        /* Delete from skiplist. */
        int retval = zslDelete(zs->zsl,score,ele,NULL);
        serverAssert(retval);

        return 1;
    }

    return 0;
}
//编码方式为listpack
```


# 实现
到此，我们已经介绍完了有关"skiplist"的相关知识，我们可以动手实现一下，实现之前先做一下力扣上的一道[跳表题](https://leetcode.cn/problems/design-skiplist/description/)加深对其的理解。

```go
const (
    maxLevel = 32
    pFactor = 0.25//概率因子
)
//定义跳表节点结构
type skiplistNode struct{
    val int
    forward []*skiplistNode
}
type Skiplist struct {
    head *skiplistNode
    level int //当前跳表的层数
}
func Constructor() Skiplist {
    return Skiplist{
        head:&skiplistNode{
            val:-1,
            forward:make([]*skiplistNode,maxLevel),
        },
        level:1,
    }
}
//获取节点的层级
func (this *Skiplist) randLevel()int{
    lv:=1
    if lv<maxLevel && rand.Float64()<pFactor{
        lv++
    }
    return lv
} 
func (this *Skiplist) Search(target int) bool {
    cur := this.head
    for i:=this.level-1;i>=0;i--{
        //当前节点的值小于target就一直向下寻找
        for cur.forward[i]!=nil && cur.forward[i].val<target{
            cur=cur.forward[i]
        }
    }
    cur=cur.forward[0]//找到最后一层的实际节点
    return cur!=nil && cur.val==target
}
func (this *Skiplist) Add(num int)  {
    cur:=this.head
    le:=this.randLevel()
    maxle:=max(le,this.level)
    //记录每一层插入节点的前一个节点
    update:= make([]*skiplistNode,maxle)
    for i:=maxle-1;i>=this.level;i--{
        update[i]=this.head
    }
	//寻找插入节点前一个节点
    for i:=this.level-1;i>=0;i--{
        for cur.forward[i]!=nil && cur.forward[i].val<num{
            cur=cur.forward[i]
        }
        update[i]=cur
    }
    newnode:=&skiplistNode{
        val:num,
        forward:make([]*skiplistNode,le),
    }
    //开始插入
    for i:=le-1;i>=0&&update[i].forward[i]!=newnode;i--{
        //每一层，将节点插入在前一个节点的后面
        newnode.forward[i] = update[i].forward[i]
        update[i].forward[i]=newnode
    }
    this.level=maxle
}

func (this *Skiplist) Erase(num int) bool {
    cur:=this.head
    update:=make([]*skiplistNode,this.level)
  
    //寻找删除节点前一个节点
    for i:=this.level-1;i>=0;i--{
        for cur.forward[i]!=nil && cur.forward[i].val<num{
            cur=cur.forward[i]
        }
        update[i]=cur
    }
    cur=cur.forward[0]
    if cur==nil || cur.val!=num {
        //该元素不存在
        return false
    }

    //开始删除
    for i:=0;i<this.level&&update[i].forward[i]==cur;i++{
        update[i].forward[i]=cur.forward[i]
    }
    for this.level>=1&&this.head.forward[this.level-1]==nil{
        this.level--
    }
    return true
}
func max(a,b int)int{
    if a>b{
        return a
    }
    return b
}
/**
 * Your Skiplist object will be instantiated and called as such:
 * obj := Constructor();
 * param_1 := obj.Search(target);
 * obj.Add(num);
 * param_3 := obj.Erase(num);
 */
```






