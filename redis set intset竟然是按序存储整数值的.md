
# 前言
redis有一个数据类型"set"，它的特性是"无序，不重复"。遇到一些需要去重的场景可以使用redis的"set"进行存储。它的底层实现有两种，一个是"intset整数集合"是一种比较节省内存的结构，另一个是"ht哈希表"。本篇文章先介绍"intset"，"ht"在之后的文章介绍。

<font color='red'>在读源码的过程中，发现在redis7.2.2版本中set的底层多了一种实现"listpack"，我又去看了之前的版本发现之前的版本set的实现只有两种"intset"和"ht"。对于[上一篇文章](https://blog.csdn.net/weixin_46290302/article/details/134182811?spm=1001.2014.3001.5502)的出现的错误已改正。</font>


# saddCommand
先从"sadd(向集合中添加元素)"命令开始分析，

```c
typedef struct client {
	//此处省略
	..........
	int argc;//Num of arguments of current command. 
    robj **argv;//Arguments of current command. 
    .....................
    //此处省略
}client;
```

```c
void saddCommand(client *c) {
    robj *set;
    int j, added = 0;

	/*
	要操作的key已经被创建但是这个key的类型却不是"OBJ_SET"报错之后直接返回;
	比方说，先使用"set m a"创建了一个string Object；
	如果此时再使用"sadd m 2"命令欲创建一个set Object会报错如下图所示。
	因为redis有一个全局的数据字典，所有的key都存储在里面，所以不同类型的key不能够重名。
	*/
    set = lookupKeyWrite(c->db,c->argv[1]);
    if (checkType(c,set,OBJ_SET)) return;
    /*
    int checkType(client *c, robj *o, int type) {
    	if (o && o->type != type) {
        	addReplyErrorObject(c,shared.wrongtypeerr);
        	return 1;
    	}
    	return 0;
	}
                              */
    
    if (set == NULL) {
    //根据第一个value新建一个set Object，并选择合适的编码，
    //"setTypeCreate"，第一个参数表示命令中第一个加入集合中的value值，第二个参数表示加入集合的元素总数，具体分析在下面
        set = setTypeCreate(c->argv[2]->ptr, c->argc - 2);
        //将key-value的映射加入到全局dict中
        dbAdd(c->db,c->argv[1],set);
    } else {
    //依据要加入集合中元素的个数判断是否需要转换编码
    //需要的话则转换
        setTypeMaybeConvert(set, c->argc - 2);
    }
    //将value值全部加入到set中
    //"setTypeAdd"的分析在下面
    for (j = 2; j < c->argc; j++) {
        if (setTypeAdd(set,c->argv[j]->ptr)) added++;
    }
    if (added) {
        signalModifiedKey(c,c->db,c->argv[1]);
        notifyKeyspaceEvent(NOTIFY_SET,"sadd",c->argv[1],c->db->id);
 -   }
    server.dirty += added;
    addReplyLongLong(c,added);
}
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/3250167817bdae7b57dcab81e47ad4e0.png)


# setTypeCreate
首先看一下如何根据value创建合适的set Object
[redis-7.2.2\src\t_set.c]
```c
/*
Factory method to return a set that *can* hold "value". 
When the object has an integer-encodable value, an intset will be returned. 
Otherwise a listpack or a regular hash table. 
The size hint indicates approximately how many items will be added which is used to determine the initial representation. 
*/
/*
setTypeCreate方法返回一个可以存储元素的集合.
如果可以整形编码，则会返回一个intset.
否则的话，返回一个listpack或者常规的ht.
size_hint表示大约有多少元素将被加入"set"
*/
robj *setTypeCreate(sds value, size_t size_hint) {
    if (isSdsRepresentableAsLongLong(value,NULL) == C_OK && size_hint <= server.set_max_intset_entries)
        return createIntsetObject();
    if (size_hint <= server.set_max_listpack_entries)
        return createSetListpackObject();

    /* We may oversize the set by using the hint if the hint is not accurate, but we will assume this is acceptable to maximize performance. */
    robj *o = createSetObject();
    dictExpand(o->ptr, size_hint);
    return o;
}
```

 - isSdsRepresentableAsLongLong(sds s,long long *llval);函数的作用是判断能否将sds表示的字符串转化为"long long"型的整数。

```c
int isSdsRepresentableAsLongLong(sds s, long long *llval) {
    return string2ll(s,sdslen(s),llval) ? C_OK : C_ERR;
}
/*
Convert a string into a long long. 
Returns 1 if the string could be parsed into a (non-overflowing) long long, 0 otherwise. 
The value will be set to the parsed value when appropriate.
*/
/*
将字符串转换为long long类型。如果字符串可以在无溢出的情况下转化为long long类型返回1，反之返回0。
传输参数value会在适当的时候被设置成解析后的整数值。
*/
int string2ll(const char *s, size_t slen, long long *value) {
	//此处省略具体的代码
	........................
}
```
**setTypeCreate函数的整体思想:**


 - 如果加入到集合中的元素能转换为"long long"类型并且元素总数小于等于"server.set_max_intset_entries"，set的底层编码是intset。
 - 如果加入到集合中的元素不能转化为"long long"类型，但是元素总数小于等于"server.set_max_listpack_entries"，set底层编码实现为"listpack".
 - 如果加入到集合中的元素能转化为"long long"类型，但是元素总数大于"server.set_max_intset_entries" 并且小于等于"server.set_max_listpack_entries"，set底层实现为"listpack"，<font color='red'>但是事实上这种情况不会发生。</font>。因为"server.set_max_listpack_entries"的数值小于"server.set_max_intset_entries"的数值。
 - 如果加入到集合中的元素能转化为"long long"类型，但是元素总数大于"server.set_max_intset_entries" ，set底层编码为"ht"。


**总结**

 - 对于全是整数的情况，选用"intset"编码节省内存，<font color='blue'>至于为什么不选择"ziplist"而是选择一种新的结构"intset"看完下面的分析就会理解。</font>
 - 对于含有非整数的情况，如果元素总数在一定范围内，选用"listpack"节省内存。

接下来看看"server.set_max_intset_entries"和"server.set_max_listpack_entries"的具体默认数值是多少，以下是在"redis.conf"文件中找到的


**redis.conf[7.2.2]**
>Sets have a special encoding when a set is composed
of just strings that happen to be integers in radix 10 in the range
of 64 bit signed integers.
The following configuration setting sets the limit in the size of the
set in order to use this special memory saving encoding.
set-max-intset-entries 512

>Sets containing non-integer values are also encoded using a memory efficient data structure when they have a small number of entries, and the biggest entry does not exceed a given threshold. These thresholds can be configured using the following directives.
set-max-listpack-entries 128
set-max-listpack-value 64
-------------------------
对比看看之前的版本，"setTypeCreate"的实现。

**redis6.2.14\redis5.0.13\3.2.100版本**

```c
robj *setTypeCreate(sds value) {
    if (isSdsRepresentableAsLongLong(value,NULL) == C_OK)
        return createIntsetObject();
    return createSetObject();
}
```
>Sets have a special encoding in just one case: when a set is composed
of just strings that happen to be integers in radix 10 in the range
of 64 bit signed integers.
The following configuration setting sets the limit in the size of the
set in order to use this special memory saving encoding.
set-max-intset-entries 512

---------------------

 - 之前版本set的底层编码只有两种"intset"和"ht"

# createIntsetObject
[redis-7.2.2\src\object.c]
分析完了什么情况下选择"intset"作为set Object的编码，接下来看看如何创建"intset"对象。

```c
robj *createIntsetObject(void) {
	//创建intset Object，具体分析在下面
    intset *is = intsetNew();
    //创建一个type是OBJ_SET的redisObject，并且ptr指向is
    //有关redisObject在上一篇文章对sds源码解读中分析过
    robj *o = createObject(OBJ_SET,is);
    //设置redisObject的编码为intset
    o->encoding = OBJ_ENCODING_INTSET;
    return o;
}
```

# intsetNew

分析如何创建intset对象之前，先来看看intset结构长什么样子
[redis-7.2.2\src\intset.h]
```c
typedef struct intset {
    uint32_t encoding;//4bytes,记录整数值的位宽
    uint32_t length;//4bytes，记录集合中元素的个数
    int8_t contents[];//1byte整数数组存储整数值
} intset;
```

intset结构中的encoding有如下三种

```c
/* Note that these encodings are ordered, so:
 * INTSET_ENC_INT16 < INTSET_ENC_INT32 < INTSET_ENC_INT64. */
#define INTSET_ENC_INT16 (sizeof(int16_t))//整数值的位宽是16
#define INTSET_ENC_INT32 (sizeof(int32_t))//整数值的位宽是32
#define INTSET_ENC_INT64 (sizeof(int64_t))//整数值的位宽是64
```

[redis-7.2.2\src\intset.c]
```c
/* Create an empty intset. */
intset *intsetNew(void) {
    intset *is = zmalloc(sizeof(intset));
    //新建立的intset对象的encoding为占用内存空间最小的"INTSET_ENC_INT16"
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);
    is->length = 0;
    return is;
}
```

 - 新创建的intset结构的encoding为"INTSET_ENC_INT16"，但是intset对应的encoding 有三种，那何时才会更换encoding的类型呢
 
 - intsetUpgradeAndAdd函数必要时候更换intset的encoding类型

# intsetUpgradeAndAdd
先介绍一些辅助函数
```c
/* Return the required encoding for the provided value. */
//根据v的大小返回合适的编码，用来决定整数值存储时的位宽
static uint8_t _intsetValueEncoding(int64_t v) {
    if (v < INT32_MIN || v > INT32_MAX)
        return INTSET_ENC_INT64;
    else if (v < INT16_MIN || v > INT16_MAX)
        return INTSET_ENC_INT32;
    else
        return INTSET_ENC_INT16;
}
/* Resize the intset */
//intset升级到更大范围的encoding就需要扩充内存
static intset *intsetResize(intset *is, uint32_t len) {
    uint64_t size = (uint64_t)len*intrev32ifbe(is->encoding);
    assert(size <= SIZE_MAX - sizeof(intset));//判断是否溢出
    //原地扩容内存
    is = zrealloc(is,sizeof(intset)+size);
    return is;
}
/* Return the value at pos, given an encoding. */
//根据当前intset所使用的编码确定一个整数值所占内存的大小，从而确定处于pos位置的整数值并返回
static int64_t _intsetGetEncoded(intset *is, int pos, uint8_t enc) {
    int64_t v64;
    int32_t v32;
    int16_t v16;

    if (enc == INTSET_ENC_INT64) {
        memcpy(&v64,((int64_t*)is->contents)+pos,sizeof(v64));
        memrev64ifbe(&v64);
        return v64;
    } else if (enc == INTSET_ENC_INT32) {
        memcpy(&v32,((int32_t*)is->contents)+pos,sizeof(v32));
        memrev32ifbe(&v32);
        return v32;
    } else {
        memcpy(&v16,((int16_t*)is->contents)+pos,sizeof(v16));
        memrev16ifbe(&v16);
        return v16;
    }
}

```

```c
/* Upgrades the intset to a larger encoding and inserts the given integer. */
//更新intset到更大范围的encoding，并插入新的元素
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
	//获取当前intset对象的编码
    uint8_t curenc = intrev32ifbe(is->encoding);
    //根据value的大小，返回合适的编码
    uint8_t newenc = _intsetValueEncoding(value);
    //获取当前集合中的元素个数
    int length = intrev32ifbe(is->length);
    //判断value是否是负数，辅助下面的插入操作
    int prepend = value < 0 ? 1 : 0;

	//改变集合的编码
    /* First set new encoding and resize */
    is->encoding = intrev32ifbe(newenc);
    
    
    //重新为intset分配内存
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    //从后往前升级，这样不会覆盖intset中原有的元素
    //如果新插入的元素是负数那么prepend的值为1，也就是每一个元素都要在原先的基础上多往后移动一个位置，这么做是为了将这个负数值添加在intset的首部。[因为intset内部存储元素是按照从小到大的顺序排序的，下面会分析]
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    /* Set the value at the beginning or the end. */
    //如果value是负数将其插入在新的intset的首部位置
    //反之寻找到合适的位置之后再插入
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    //加入新的元素，修改集合的len
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}

```
**总结**

 - 首先根据命令中的第一个value值选择合适的编码[从"intset、listpack、ht"中选择]创建新的set Object。
 - 底层结构intset随着数据的加入会发生encoding的升级，并且一旦升级不会再次降级。也即是说如果编码从一个小范围升级到更大的范围，但此时元素值都在更小的范围内，也不会发生encoding的降级，多少会浪费掉一些内存。
 - 之后将所有的value值一并加入到set中去。是通过函数"setTypeAdd"完成的。

 
# setTypeAdd
现在根据命令中的第一个value值创建好了set Object对象，接下来将元素都添加到集合中

```c
//值不存在成功插入返回1
//反之插入失败返回0
int setTypeAdd(robj *subject, sds value) {
    return setTypeAddAux(subject, value, sdslen(value), 0, 1);
}

/* 
The value can be provided as an sds string (indicated by passing str_is_sds =1), as string and length (str_is_sds = 0) or as an integer in which case str is set to NULL and llval is provided instead.
*/
/*
如果str_is_sds==1,value以字符串的形式传入。
如果str_is_sds==0,value以long long类型整数值传入。
*/
int setTypeAddAux(robj *set, char *str, size_t len, int64_t llval, int str_is_sds) {
    char tmpbuf[LONG_STR_SIZE];
    if (!str) {
    //插入的值是能用long long表示的整数值，并且当前set的底层编码是"intset"直接插入这个整数值即可
        if (set->encoding == OBJ_ENCODING_INTSET) {
            uint8_t success = 0;
            set->ptr = intsetAdd(set->ptr, llval, &success);
            //依据集合的元素总数判断是否需要转化set的底层编码
            if (success) maybeConvertIntset(set);
            return success;
        }
        /* Convert int to string. */
        len = ll2string(tmpbuf, sizeof tmpbuf, llval);
        str = tmpbuf;
        str_is_sds = 0;
    }

    serverAssert(str);
    if (set->encoding == OBJ_ENCODING_HT) {
    	..........................
        
    } else if (set->encoding == OBJ_ENCODING_LISTPACK) {
    	...........................
        
    } else if (set->encoding == OBJ_ENCODING_INTSET) {
        long long value;
        //value能表示成long long类型的整数值
        if (string2ll(str, len, &value)) {
            uint8_t success = 0;
            set->ptr = intsetAdd(set->ptr,value,&success);
            if (success) {
                maybeConvertIntset(set);
                return 1;
            }
        } else {
            /* Check if listpack encoding is safe not to cross any threshold. */
            //插入的value值不能转化为long long类型，需要更换底层的编码
            size_t maxelelen = 0, totsize = 0;
            unsigned long n = intsetLen(set->ptr);
            if (n != 0) {
            //sdigits10返回value转化为字符串的长度
                size_t elelen1 = sdigits10(intsetMax(set->ptr));
                size_t elelen2 = sdigits10(intsetMin(set->ptr));
                maxelelen = max(elelen1, elelen2);
                size_t s1 = lpEstimateBytesRepeatedInteger(intsetMax(set->ptr), n);
                size_t s2 = lpEstimateBytesRepeatedInteger(intsetMin(set->ptr), n);
                totsize = max(s1, s2);
            }
            //元素总个数小于"listpack_entries"并且
            //数值字符串的长度小于等于"listpack_value"转化为"listpack"编码
            if (intsetLen((const intset*)set->ptr) < server.set_max_listpack_entries &&
                len <= server.set_max_listpack_value &&
                maxelelen <= server.set_max_listpack_value &&
                lpSafeToAdd(NULL, totsize + len))
            {
               
                setTypeConvertAndExpand(set, OBJ_ENCODING_LISTPACK,
                                        intsetLen(set->ptr) + 1, 1);
                unsigned char *lp = set->ptr;
                lp = lpAppend(lp, (unsigned char *)str, len);
                lp = lpShrinkToFit(lp);
                set->ptr = lp;
                return 1;
            } else {
            //编码转化为"HT"
                setTypeConvertAndExpand(set, OBJ_ENCODING_HT,
                                        intsetLen(set->ptr) + 1, 1);
                
                serverAssert(dictAdd(set->ptr,sdsnewlen(str,len),NULL) == DICT_OK);
                return 1;
            }
        }
    } else {
        serverPanic("Unknown set encoding");
    }
    return 0;
}
```
------------
## 总结
 - 虽然可能新建的set Object的底层编码是intset，但是会随着插入数据的类型而改变，如果插入的是非整数值会转化编码到"listpack"或者"ht"。
 - 也即是说，只有元素都是整数值的情况下底层编码才会选用"intset"。

演示一下，set底层编码是如何转换的
刚开始向set中添加一个能用long long类型表示的整数值
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/da1c163bbbab16c0c19e849aeffe7b07.png)
接着向set中添加一个字符串
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/da6bbce97df64d7825a128bd346ef77f.png)
其实按照源码的分析，新加入的字符串长度小于等于"listpack_value"，元素总数小于等于"listpack_entries"编码应该是"listpack"。但是由于用来测试的redis版本是3版本的所以显示的是"hashtable"。
 

# 相关操作
集合的特性就是去重，接下来看看intset是如何在执行插入操作时实现去重的。

先介绍一些辅助函数

```c
//将is中从from位置开始的元素，移动到to位置
static void intsetMoveTail(intset *is, uint32_t from, uint32_t to) {
    void *src, *dst;
    uint32_t bytes = intrev32ifbe(is->length)-from;
    uint32_t encoding = intrev32ifbe(is->encoding);

    if (encoding == INTSET_ENC_INT64) {
        src = (int64_t*)is->contents+from;
        dst = (int64_t*)is->contents+to;
        bytes *= sizeof(int64_t);
    } else if (encoding == INTSET_ENC_INT32) {
        src = (int32_t*)is->contents+from;
        dst = (int32_t*)is->contents+to;
        bytes *= sizeof(int32_t);
    } else {
        src = (int16_t*)is->contents+from;
        dst = (int16_t*)is->contents+to;
        bytes *= sizeof(int16_t);
    }
    memmove(dst,src,bytes);
}
/* Set the value at pos, using the configured encoding. */
//将value值存储到is中下标为pos的位置处
static void _intsetSet(intset *is, int pos, int64_t value) {
    uint32_t encoding = intrev32ifbe(is->encoding);

    if (encoding == INTSET_ENC_INT64) {
        ((int64_t*)is->contents)[pos] = value;
        memrev64ifbe(((int64_t*)is->contents)+pos);
    } else if (encoding == INTSET_ENC_INT32) {
        ((int32_t*)is->contents)[pos] = value;
        memrev32ifbe(((int32_t*)is->contents)+pos);
    } else {
        ((int16_t*)is->contents)[pos] = value;
        memrev16ifbe(((int16_t*)is->contents)+pos);
    }
}
```

##  intsetAdd

```c
/* Insert an integer in the intset */
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
	//根据value的大小选择合适的编码方式，为下面判断是否需要encoding升级做准备
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;//存储新元素插入的位置下标
    if (success) *success = 1;

    //value值超出当前编码范围，升级编码
    if (valenc > intrev32ifbe(is->encoding)) {
    //上面已经分析过，进行编码的升级为了能存储更大的数值
        return intsetUpgradeAndAdd(is,value);
    } else {
        //插入之前会先查找要插入的value值是否在集合中存在
        //若存在则不执行任何操作
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }
        //不存在才执行插入操作，从而达到去重的目的
		//插入新元素，先其分配空间
        is = intsetResize(is,intrev32ifbe(is->length)+1);
        //如果新元素插入在中间的位置，需要先将pos位置的元素进行移动为新元素留出空间然后插入
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }
	//在合适的位置插入元素
    _intsetSet(is,pos,value);
    //修改length
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}

```

 - 通过分析可以发现，在插入新的value之前会先进行查找操作，判断当前的value是否已经被加入到集合中。不存在的情况下才进行插入操作达到去重的目的。



## intsetSearch
接下来看看如何进行查找的。
```c
/* 
Search for the position of "value". 
Return 1 when the value was found and sets "pos" to the position of the value within the intset. 
Return 0 when the value is not present in the intset and sets "pos" to the position where "value" can be inserted. 
*/
//如果值被找到则返回1，并将传出参数pos设置为值所在的位置
//如果值没有被找到则返回0，并将传出参数设置成值应该存储的位置
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;

    //intset的元素个数为0，插入位置为下标为0的位置
    if (intrev32ifbe(is->length) == 0) {
        if (pos) *pos = 0;
        return 0;
    } else {
    //_intsetGet寻找在is中位于下标为max位置的value，也即是intset存储的最大的value值
    //如果新插入的value比intset中最大的value值还要大，将其存储在intset的末尾。
        if (value > _intsetGet(is,max)) {
            if (pos) *pos = intrev32ifbe(is->length);
            return 0;
            //比最小值还要小则插入首位置
        } else if (value < _intsetGet(is,0)) {
            if (pos) *pos = 0;
            return 0;
        }
    }
	//寻找插入位置
    while(max >= min) {
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }
	//在intset中找到该value，返回value所在的位置
    if (value == cur) {
        if (pos) *pos = mid;
        return 1;
    } else {
    //没有找到，返回value应当存储的位置
        if (pos) *pos = min;
        return 0;
    }
}
```
<font color='red'>把查找的核心代码提取出来，就会发现这是二分查找。既然是二分查找，那说明intset存储元素是有序存储的。因为二分查找的前提是"列表有序"。</font>
[更多有关"二分查找"知识可以看文章"[数组，链表专题](https://blog.csdn.net/weixin_46290302/article/details/135436771?spm=1001.2014.3001.5501)"]


```c
//采用的左闭右闭区间
int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
while(max >= min) {
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }
```




**总结**

 - 向intset中插入value之前，先查找value是否已经存在。存在则不插入，不存在才插入。从而实现去重。
 - 查找的核心是二分查找，也说明intset中的value都是"从小大大"按序存储的。
 - 找到插入的合适位置之后，先进行元素的迁移腾出位置之后再插入新的value。

来验证一下intset是否是有序的
首先以乱序的方式加入intset，然后查看intset中的所有元素
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ffba28373d83fb537f26f3bf66a7ef92.png)
然后再加入一个数值，再次查看
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/58840f17f2e0839a9a145169a8ed2120.png)
## intsetRemove
在"[数组，链表专题](https://blog.csdn.net/weixin_46290302/article/details/135436771?spm=1001.2014.3001.5501)"中介绍过有关数组的相关操作。对于删除操作有两种方式，一种是为删除的元素打上标记之后再进行数据的迁移可以减少数据迁移的工作量减少开销，另一种是直接进行数据的迁移。
那来看看intset结构的删除操作采用的是哪一中。

```c
/* Delete integer from intset */
intset *intsetRemove(intset *is, int64_t value, int *success) {
//根据value的大小判断value值的编码
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 0;
    
	//valenc>is->encoding表明，删除的value值超出了当前编码的范围，那此时value值一定不存在于当前的intset中。所以必须要先满足
	//valenc<=is->encoding
    if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,&pos)) {
    //在intset中找到目标value的位置为pos
        uint32_t len = intrev32ifbe(is->length);

        /* We know we can delete */
        if (success) *success = 1;

        /* Overwrite value with tail and update length */
        /*
        if (pos<(len-1))说明目标value的位置在intset的中间位置，需要先进行intsetMoveTail(intset*is,uint32_t from,uint32_t to)元素的覆盖操作再缩小空间并修改长度。如果删除的是末尾的元素，直接缩小空间修改长度即可。
        */
        if (pos < (len-1)) intsetMoveTail(is,pos+1,pos);
        //缩小内存
        is = intsetResize(is,len-1);
        //设置新的长度
        is->length = intrev32ifbe(len-1);
    }
    return is;
}
```

 - 通过上述分析呢发现，每添加删除一次元素都需要进行元素的迁移操作。当元素量很大的时候，这种迁移操作是非常耗时的，所以intset能容纳元素数量有限默认512个[这一点上面分析set编码选择的时候也提到过。]
 - 分析到这里其实也能明白，为什么对于全是整数的情况下没有选择"ziplist或者listpack"节省内存，因为为了提高查找的效率需要保持数值都是有序存储并且能够根据下标随机访问。

## spopCommand
集合除了具有唯一的特性之外，还有一个特性就是无序。对源码的分析过程中我们发现intset存储数据是有序的，那它是如何实现随机的呢？我在"spop keyname"或者"srandmember keyname"[两者的区别在于spop获取元素的同时将其删除，srandmember不会删除元素]底层编码为intset的集合的时候，会不会拿到的整数值是有序的呢?
先来实操一下吧

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/8f5cd494e8d8b94587d3e41f9dff9394.png#pic_center)
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/a38ef1d41782f8619180073cb9aee9cd.png#pic_center)

 - 从实操中可以看出，spop\srandmember的操作完全是随机的。但是当spop\srandmember length时得到的又完全是有序的，是巧合吗？为了验证是不是偶然现象，看一看源码是如何执行的。


由于"spop"命令和"srandmember"命令的区别在于删不删出元素，其它逻辑都是一致的，所以这里只分析"spop"命令


先从"spopCommand"函数看起

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/2def7655a88897576cca1655cd87b499.png#pic_center)

 - 找到两个关键的函数"setTypePopRandom"和"spopWithCountCommand"

先来看看"setTypePopRandom"函数

## setTypePopRandom
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/8a12b0a946bf1b5c7a3d5390142db0e1.png#pic_center)
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/178b8e7340fadd8fda444a907a6db3cf.png#pic_center)

```c
/* Return random member */
//返回一个随机数
int64_t intsetRandom(intset *is) {
	//获取总的元素个数
    uint32_t len = intrev32ifbe(is->length);
    assert(len); /* avoid division by zero on corrupt intset payload. */
    //_intsetGet在上面有提到过，作用是寻找is中下标位置为"rand()%len"的元素
    return _intsetGet(is,rand()%len);
}
```

 - 从上述源码可以发现，spop不使用count参数的时候，会根据rand函数随机找出intset里面的一个数值返回。


## spopWithCountCommand
接下来看看带有"count"参数的命令
由于代码太长，所以只截取对我们有帮助的代码。

 - 带有count参数的pop命令，根据count的大小分成了三种情况。先来看第一种
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ab364a274312755bf68330873502a85a.png#pic_center)

> 
  	 /* CASE 1:
     * The number of requested elements is greater than or equal to
     * the number of elements inside the set: simply return the whole set. */
     
     如果参数count大于等于集合的元素总数：那么原封不动的返回整个集合
     这也就解释了，当count参数等于集合元素总数时，得到的是一整个有序的序列

 - 第二种情况
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d6abd5402499e6d5394424f3b59c5da5.png#pic_center)

 

>
 	/* CASE 2: The number of elements to return is small compared to the
     * set size. We can just extract random elements and return them to
     * the set. */
     参数count小于集合的元素个数。从中随机抽取一部分元素。

 - 第三种情况
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/450c2dd03a8a9c4cfdd4ecae65b263ae.png#pic_center)


>
	/* CASE 3: The number of elements to return is very big, approaching
     * the size of the set itself. After some time extracting random elements
     * from such a set becomes computationally expensive, so we use
     * a different strategy, we extract random elements that we don't
     * want to return (the elements that will remain part of the set),
     * creating a new set as we do this (that will be stored as the original
     * set). Then we return the elements left in the original set and
     * release it. */
     参数count小于集合中的元素总数但是特别大十分接近元素总数。
     分多次从集合中随机抽取元素开销很大，所以采用一种不同的策略：
     首先准备空的辅助集合，从原集合中随机抽取一些想要留在集合中的元素将其加入到辅助集合
     然后返回原集合，将辅助集合提升为原集合。

	这是一个很好的思路，当从正面想问题感觉到困难的时候，
	不如从反面想一想。
	生活也是如此，希望看到文章的你能事事顺心，事事如意。没有解决不了的困难，只有不愿思考的大脑。

 - 如果相对数组有更进一步的了解，可以参考文章"[数组，链表专题](https://blog.csdn.net/weixin_46290302/article/details/135436771?spm=1001.2014.3001.5501)"
