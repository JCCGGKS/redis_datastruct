

#  前言
ziplist是一种节省内存的数据结构，是旧版本中"list"、"hash"、"zset"的底层编码之一。由于ziplist存在一些问题，于是在新版本中用listpack代替ziplist。更具体地数据类型和底层编码的对应关系，可以参考文章"[sds源码剖析](https://blog.csdn.net/weixin_46290302/article/details/134182811?spm=1001.2014.3001.5501)"

先来分析一下"ziplist"和"listpack"的结构以及一些基本的操作，然后再分析"list"、"hash"、"set"、"zset"如何利用它节省内存的。
# ZipList
## ziplist__struct
redis-7.2.2\src\ziplist.c
首先看一下，源码中关于ziplist的介绍
> The ziplist is a specially encoded dually linked list that is designed to be very memory efficient. It stores both strings and integer values, where integers are encoded as actual integers instead of a series of characters. It allows push and pop operations on either side of the list in O(1) time. However, because every operation requires a reallocation of the memory used by the ziplist, the actual complexity is related to the amount of memory used by the ziplist.

>ziplist是双向链表的特殊编码方式，被设计用来节省内存；存储字符串和整数值，其中整数值被编码为实际的整数而不是字符。<font color='red'>允许在任意位置push和pop，时间复杂度为O(1)但是由于每一个操作都会引起内存的重新分配，所以实际的复杂度和使用的内存成正比。</font>

>从上述可以看出，ziplist是一个紧凑型的数据结构，相对于“Linked List”更加节省内存，但是删除和插入会引起内存的重新分配。

接下来看一下，ziplist大致的布局
> \<zlbytes> \<zltail> \<zllen> \<entry> \<entry>... \<entry>  \<zlend>

<font color='blue'>zlbytes[uint32_t]</font>
> \<uint32_t zlbytes> is an unsigned integer to hold the number of bytes that the ziplist occupies, including the four bytes of the zlbytes field itself. This value needs to be stored to be able to resize the entire structure without the need to traverse it first.

>zlbytes是一个无符号整型，存储ziplist总共占有的字节数[包括4个字节的zlbytes]。存储字节数的目的是为了能够在必要时刻重新调整ziplist的结构，而不需要先遍历ziplist获取其占用的字节数。

<font color='blue'>zltail[uint32_t]</font>
>\<uint32_t zltail> is the offset to the last entry in the list. This allows a pop operation on the far side of the list without the need for full  traversal.

>zltail存储的是list中最后一个entry的偏移量，能够帮助我们快速定位最后一个entry,在执行pop最后一个元素时无需遍历ziplist

<font color='blue'>zllen[uint16_t]</font>
>\<uint16_t zllen> is the number of entries. When there are more than 2^16^-2 entries, this value is set to 2^16^-1 and we need to traverse the entire list to know how many items it holds.

>zllen存储的是entry的数量，也就是ziplist中元素的个数。但是其只有两个字节，最大只能存储(2^16^ -1）。一旦entry个数超过(2^16^ -2)该值会被设置为(2^16^ -1)；所以如果该值为(2^16^ -1)需要遍历获取元素的个数，反之直接由该字段获取。

<font color='blue'>zlend[uint8_t]</font>
>\<uint8_t zlend> is a special entry representing the end of the ziplist. Is encoded as a single byte equal to 255. No other normal entry starts with a byte set to the value of 255.

>zlend是一个特殊的entry用来标识ziplist的结尾， 被单独编码为255，其他的正常entry都不会被编码为255.


<font color='blue'>ziplist entries</font>

>Every entry in the ziplist is prefixed by metadata that contains two pieces of information. First, the length of the previous entry is stored to be able to traverse the list from back to front. Second, the entry encoding is provided. It represents the entry type, integer or string, and in the case of strings it also represents the length of the string payload. So a complete entry is stored like this:

>\<prevlen> \<encoding> \<entry-data>

>ziplist中的每一个entry都包含两项前缀信息；第一个，前一个entry的长度，记录这个是为了实现从后往前遍历；第二个entry的encoding表示entry的类型，“integer or string”。在encoding为string的情况下，还表示字符串有效载荷的长度。完整的结构如下：

>\<prevlen> \<encoding> \<entry-data>

在数据类型为小整数的情况下，encoding表示entry自身，所以此时不需要<entry-data>

<font color='blue'>\<prevlen></font>
>表示前一个entry的len，编码方式如下：
>前一个entry的len小于254，只需占用1个字节[unsigned 8 bits interger]
>前一个entry长度len大于等于254，需要占用5个字节，其中第一个字节被设置为254(FE)表示这是一个更大的数，剩余的4个字节存储前一项的长度
>\<prevlen from 0 to 253> \<encoding> \<entry>
>0xFE \<4 bytes unsigned little endian prevlen> \<encoding> \<entry>

介绍完"ziplist"的结构，看看它的一些基本操作
redis-7.2.2\src\ziplist.c
## 宏定义

```c
//返回ziplist占用的总字节数
#define ZIPLIST_BYTES(zl)       (*((uint32_t*)(zl)))

//返回zltail的偏移量
#define ZIPLIST_TAIL_OFFSET(zl) (*((uint32_t*)((zl)+sizeof(uint32_t))))

//返回ziplist的元素个数，如果返回的是UINT16_MAX[255]需要遍历获取元素总数
#define ZIPLIST_LENGTH(zl)      (*((uint16_t*)((zl)+sizeof(uint32_t)*2)))

//头部包含两个32位整数分别记录占用的总的字节数和最后一个entry的偏移量。一个16位的整数记录ziplist的长度
#define ZIPLIST_HEADER_SIZE     (sizeof(uint32_t)*2+sizeof(uint16_t))

//返回endsize的大小，只占用一个字节
#define ZIPLIST_END_SIZE        (sizeof(uint8_t))

//返回第一个entry
#define ZIPLIST_ENTRY_HEAD(zl)  ((zl)+ZIPLIST_HEADER_SIZE)

//返回最后一个entry
#define ZIPLIST_ENTRY_TAIL(zl)  ((zl)+intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl)))

//返回zlend
#define ZIPLIST_ENTRY_END(zl)   ((zl)+intrev32ifbe(ZIPLIST_BYTES(zl))-ZIPLIST_END_SIZE)

```

## ziplistBlobLen

```c
//返回ziplist占用的总字节数
size_t ziplistBlobLen(unsigned char *zl) {
    return intrev32ifbe(ZIPLIST_BYTES(zl));
}
```

## ziplistLen

```c
//获取ziplist的entry总数
unsigned int ziplistLen(unsigned char *zl) {
    unsigned int len = 0;
    //如果head中的zllen字段存储的数值小于等于(2^16^-2)，直接返回zllen存储的长度
    if (intrev16ifbe(ZIPLIST_LENGTH(zl)) < UINT16_MAX) {
        len = intrev16ifbe(ZIPLIST_LENGTH(zl));
    } else {
    //遍历ziplist获取元素总数
    //p指向第一个entry
        unsigned char *p = zl+ZIPLIST_HEADER_SIZE;
        size_t zlbytes = intrev32ifbe(ZIPLIST_BYTES(zl));
        while (*p != ZIP_END) {
        //获取当前entry的长度,p加上这个长度得到p的下一个entry
            p += zipRawEntryLengthSafe(zl, zlbytes, p);
            len++;
        }

        //如果元素总数小于等于(2^16^-2)修改zllen字段为实际的元素总数
        //大于(2^16^-2)依然存储为UINT16_MAX
        if (len < UINT16_MAX) ZIPLIST_LENGTH(zl) = intrev16ifbe(len);
    }
    return len;
}
```
## ziplistPush

```c
unsigned char *ziplistPush(unsigned char *zl, unsigned char *s, unsigned int slen, int where) {
    unsigned char *p;
    p = (where == ZIPLIST_HEAD) ? ZIPLIST_ENTRY_HEAD(zl) : ZIPLIST_ENTRY_END(zl);
    //将新元素s插入在zl的p位置处，slen表示s的长度
    return __ziplistInsert(zl,p,s,slen);
}
```

 - 通过代码可以发现，新来的元素要么插入在ziplist的末尾，要么插入在ziplist的开头。具体的位置由where决定


## ziplistNext
```c
//返回entry_p的下一个entry，基于此可以实现链表的从前向后遍历
unsigned char *ziplistNext(unsigned char *zl, unsigned char *p) {
    ((void) zl);
    //获取ziplist的总字节数
    size_t zlbytes = intrev32ifbe(ZIPLIST_BYTES(zl));

   //p指向的位置是ziplist的末尾
    if (p[0] == ZIP_END) {
        return NULL;
    }
    //p的下一个entry是末尾
    p += zipRawEntryLength(p);
    if (p[0] == ZIP_END) {
        return NULL;
    }
    
    zipAssertValidEntry(zl, zlbytes, p);
    return p;
}
```

## ziplistPrev

```c
//返回entry_p的上一个entry
unsigned char *ziplistPrev(unsigned char *zl, unsigned char *p) {
    unsigned int prevlensize, prevlen = 0;

	//p指向zlend返回zltail_entry
    if (p[0] == ZIP_END) {
        p = ZIPLIST_ENTRY_TAIL(zl);
        return (p[0] == ZIP_END) ? NULL : p;
    } else if (p == ZIPLIST_ENTRY_HEAD(zl)) {
        return NULL;
    } else {
    //从entry_p中解析出上一个entry的长度prevlen
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
        assert(prevlen > 0);
        //向前
        p-=prevlen;
        size_t zlbytes = intrev32ifbe(ZIPLIST_BYTES(zl));
        zipAssertValidEntry(zl, zlbytes, p);
        return p;
    }
}
```
## ziplistIndex
```c
/*
	如果传入的index<0,从后往前寻找；
	index>=0,从前往后寻找。
*/
unsigned char *ziplistIndex(unsigned char *zl, int index) {
    unsigned char *p;
    unsigned int prevlensize, prevlen = 0;
    if (index < 0) {
    	//从后往前寻找，先将index变正
        index = (-index)-1;
        //首先定位到ziplist的zltail,也就是最后一个entry
        p = ZIPLIST_ENTRY_TAIL(zl);
        //p[0]!=ZIP_END也就表明当前ziplist中的元素个数不为0
        if (p[0] != ZIP_END) {
        //获取上一个entry的prevlen
            ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
            while (prevlen > 0 && index--) {
            //根据prevlen获取上一个entry
                p -= prevlen;
                ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
            }
        }
    } else {
    //正序遍历从头开始寻找
        p = ZIPLIST_ENTRY_HEAD(zl);
        while (p[0] != ZIP_END && index--) {
            p += zipRawEntryLength(p);
        }
    }
    return (p[0] == ZIP_END || index > 0) ? NULL : p;
}
```
总结一下倒序遍历就是：

 - 首先根据zltail定位到最后一个entry
 - decode出当前entry中存储的表示上一个entry长度的prevlen
 - 根据prevlen定位到前一个entry
 - 继续上述过程，直到遍历完所有entry
正序遍历：
 - 首先定位到，第一个entry: p
 - 解析出当前entry的长度:len，p+len得到下一个entry

## ziplistFind

```c
unsigned char *ziplistFind(unsigned char *zl, unsigned char *p, unsigned char *vstr, unsigned int vlen, unsigned int skip) {
    int skipcnt = 0;
    unsigned char vencoding = 0;
    long long vll = 0;
    //获取ziplist占用的总字节数
    size_t zlbytes = ziplistBlobLen(zl);
	//顺序遍历进行查找
    while (p[0] != ZIP_END) {
        struct zlentry e;
        unsigned char *q;

        assert(zipEntrySafe(zl, zlbytes, p, &e, 1));
       	//q指向当前entry的具体内容
        q = p + e.prevrawlensize + e.lensize;

        if (skipcnt == 0) {
        //比较
            /* Compare current entry with specified entry */
            if (ZIP_IS_STR(e.encoding)) {
                if (e.len == vlen && memcmp(q, vstr, vlen) == 0) {
                //找到
                    return p;
                }
            } else {
                
                if (vencoding == 0) {
                    if (!zipTryEncoding(vstr, vlen, &vll, &vencoding)) {
                     
                        vencoding = UCHAR_MAX;
                    }
                    /* Must be non-zero by now */
                    assert(vencoding);
                }
                //比较是否是整数
                if (vencoding != UCHAR_MAX) {
                    long long ll = zipLoadInteger(q, e.encoding);
                    if (ll == vll) {
                        return p;
                    }
                }
            }

            /* Reset skip count */
            skipcnt = skip;
        } else {
            /* Skip entry */
            skipcnt--;
        }

        /* Move to next entry */
        //移动到下一个entry
        p = q + e.len;
    }

    return NULL;
}
```

 - 由代码可以发现找到特定entry的遍历方式为顺序遍历


## CascadeUpdate
在看源码的过程中，发现"insert、delete、update"的时候都会引起级联更新

 - _ziplistInsert
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/a20d91f675871992a04206b148bf20cc.png#pic_center)
 - _ziplistDelete
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/cd612a4104aa4ff80a9d6a89ff8f4257.png#pic_center)
 - ziplistReplace，更新的本质是"删除之后重新添加"
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/da8f3cb792f7d507d5c81104e8dc040b.png)
接下来看看什么是"级联更新"

### _ziplistCascadeUpdate[级联更新]
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/9b40d9cec2e4f704a4f555b95b34418d.png)

先来看看源码中是怎么说的，
>When an entry is inserted, we need to set the prevlen field of the next entry to equal the length of the inserted entry. It can occur that this length cannot be encoded in 1 byte and the next entry needs to be grow a bit larger to hold the 5-byte encoded prevlen. This can be done for free, because this only happens when an entry is already being inserted (which causes a realloc and memmove). However, encoding the prevlen may require that this entry is grown as well. This effect may cascade throughout the ziplist when there are consecutive entries with a size close to ZIP_BIG_PREVLEN, so we need to check that the prevlen can be encoded in every consecutive entry.

>当一个entry被插入的时候，我们需要设置下一个entry中的prevlen字段为新插入entry的长度。会发生如下的情况：新插入entry的长度超过了254[>=254]，那么下一个entry的prelen需要5个字节记录该长度，但是呢，此时下一个entry的prevlen字段此时只有一个字节，所以需要对下一个entry进行grown[扩容]，更糟糕的是，下个entry因为扩容导致长度超过254，还会引起下下个entry的扩容..........，这种现象呢就是级联更新，简单点来说就是，因为一个entry的插入导致之后的entry都需要重新扩容和记录前一个entry的prevlen现象称之为“级联更新”。

>上面提到的级联更新是因为prevlen增大而导致的，如果prevlen变小会不会引起cascadeupdate呢？我们来看看源码如何描述的。

> Note that this effect can also happen in reverse, where the bytes required to encode the prevlen field can shrink. This effect is deliberately ignored, because it can cause a "flapping" effect where a chain prevlen fields is first grown and then shrunk again after consecutive inserts. Rather, the field is allowed to stay larger than necessary, because a large prevlen field implies the ziplist is holding large entries anyway.

>如果prevlen字段编码字节变小，会故意忽略掉这种情况。因为不忽略的话会引起”扑动“效应，即刚刚缩小就要扩大。所以，允许prevlen字段的长度比需要的大，这样可以保证总是可以存储大的entry





redis为了解决连锁更新这种现象，就引入了”listpack“我们来看看如何解决的吧

# listpack[基于redis7.2.2]
更具体的介绍需要到”[https://github.com/antirez/listpack](https://github.com/antirez/listpack)“
去看。

>A listpack is encoded into a single linear chunk of memory. It has a fixed length header of six bytes (instead of ten bytes of ziplist, since we no longer need a pointer to the start of the last element). The header is followed by the listpack elements. In theory the data structure does not need any terminator, however for certain concerns, a special entry marking the end of the listpack is provided, in the form of a single byte with value FF (255). The main advantages of the terminator are the ability to scan the listpack without holding (and comparing at each iteration) the address of the end of the listpack, and to recognize easily if a listpack is well formed or truncated. These advantages are, in the idea of the writer, worth the additional byte needed in the representation.


>一个listpack是被编码为一个单个的线性内存地址块，头部固定6字节，相比于ziplist的头部[uint32_t zlbytes uint32_t zltail uint16_t zllen]10个细节节省了4个字节。listpack不再需要指向最后一个entry起始地址的指针，仍然有特殊的entry标识listapck的结束。

看一下具体的结构
>\<total-bytes> \<num-elements> \<element-1> \<element-2> \<element-3> \... \<listpack-end-type>

<font color='blue'>total-bytes</font>
>32bit unsigned integer ，记录了listpack占用的字节数，包括头部和尾部，可以快速定位到最后一个element的末尾位置

<font color='blue'>num-elements</font>
>16 bit unsigned integer ，记录listpack的元素个数，但是由于16bit==2byte，只能记录到65535，如果元素个数超过65535需要从头到尾遍历listpack获取元素个数

<font color='blue'>看一下element的struct</font>
>listpack
>\<encoding-type> \<element-data> \<element--total-len>

<font color='blue'>再来回顾一下ziplist的entry的结构</font>
>ziplist
>\<prevlen> \<encoding> \<entry-data>

>listpack和ziplist对象结构的不同是，listpack将prevlen替换为了curlen，从而有效避免级联更新。
>并将且将culen字段放在entry结构的最后面。这样做是为了，能够通过total-bytes定位到最后一个element的末尾位置然后获取到curlen从而找到前一个element的位置，从而实现从后往前遍历。



接下来看看listpack的基本操作
## lpPrev


```c
#define LP_HDR_SIZE 6       /* 32 bit total len + 16 bit number of elements. */
unsigned char *lpPrev(unsigned char *lp, unsigned char *p) {
    assert(p);
    //p是第一个元素entry
    if (p-lp == LP_HDR_SIZE) return NULL;
    p--;
    //获取上一个entry的长度
    uint64_t prevlen = lpDecodeBacklen(p);
    prevlen += lpEncodeBacklen(NULL,prevlen);
    //加上原先减掉的1
    p -= prevlen-1; 
    lpAssertValidEntry(lp, lpBytes(lp), p);
    return p;
}
```
## lpNext

```c
 //返回p的下一个entry
unsigned char *lpSkip(unsigned char *p) {
//获取当前entry的字节数
    unsigned long entrylen = lpCurrentEncodedSizeUnsafe(p);
    entrylen += lpEncodeBacklen(NULL,entrylen);
    p += entrylen;
    return p;
}
unsigned char *lpNext(unsigned char *lp, unsigned char *p) {
    assert(p);
    //获取下一个entry
    p = lpSkip(p);
    if (p[0] == LP_EOF) return NULL;
    lpAssertValidEntry(lp, lpBytes(lp), p);
    return p;
}
```

## lpFirst


```c
//返回listpack的第一个entry
unsigned char *lpFirst(unsigned char *lp) {
    unsigned char *p = lp + LP_HDR_SIZE; /* Skip the header. */
    if (p[0] == LP_EOF) return NULL;
    lpAssertValidEntry(lp, lpBytes(lp), p);
    return p;
}
```

## lpLast

```c
//返回listback的最后一个element，若没有则返回null
unsigned char *lpLast(unsigned char *lp) {
//首先通过lptotalbytes定位到最后一个elemet的末尾位置，减1减的是末尾的表示zlend
    unsigned char *p = lp+lpGetTotalBytes(lp)-1; 
    return lpPrev(lp,p); 
}
```
## lpLength

```c
unsigned long lpLength(unsigned char *lp) {
//获取zllen字段存储的entry总数
    uint32_t numele = lpGetNumElements(lp);
    //LP_HDR_NUMELE_UNKNOWN：UINT16_MAX
    if (numele != LP_HDR_NUMELE_UNKNOWN) return numele;

    //遍历获取元素个数
    uint32_t count = 0;
    unsigned char *p = lpFirst(lp);
    while(p) {
    //计数累加元素值
        count++;
        p = lpNext(lp,p);
    }

    //元素总数小于UINT16_MAX，重新设置zllen
    if (count < LP_HDR_NUMELE_UNKNOWN) lpSetNumElements(lp,count);
    return count;
}
```
## lpSeek

```c
/*
查找指定元素并返回指向所查找元素的指针。正索引指定从头开始查找的零基元素尾部，负索引指定从尾部开始的元素，其中-1表示最后一个元素，-2表示倒数第二个元素，以此类推。如果索引超出范围，返回NULL。* /
*/
unsigned char *lpSeek(unsigned char *lp, long index) {
    int forward = 1; //默认正向遍历

    //获取listpack的元素总数
    uint32_t numele = lpGetNumElements(lp);
    if (numele != LP_HDR_NUMELE_UNKNOWN) {
    //index<0.负向第index个，即为正向numele+index个
        if (index < 0) index = (long)numele+index;
        if (index < 0) return NULL; //out of range
        if (index >= (long)numele) return NULL; // Out of range
        //如果index的位置是在listpack的后半段，采用从后向前遍历的方式
        if (index > (long)numele/2) {
            forward = 0;
          
            index -= numele;
        }
    } else {
       //长度不确定，index<0只能选择负向遍历
        if (index < 0) forward = 0;
    }

    //前向遍历
    if (forward) {
    //获取第一个entry
        unsigned char *ele = lpFirst(lp);
        while (index > 0 && ele) {
        //获取下一个entry
            ele = lpNext(lp,ele);
            index--;
        }
        return ele;
    } else {
    //获取最后一个entry
        unsigned char *ele = lpLast(lp);
        while (index < -1 && ele) {
        //获取前一个entry
            ele = lpPrev(lp,ele);
            index++;
        }
        return ele;
    }
}
```

# 总结

 - ziplist包括三大部分，\<头部，entry，尾部>。头部包含"zlbytes,zltail,zllen"；尾部包含"zlend标记ziplist的结尾"；entry包括"prevlen,encoding,entry_data"。由于prevlen记录方式存在级联更新的问题，小于254字节需要1字节内存，大于等于254字节需要5字节内存。
 - listpack包括三大部分，\<头部，entry，尾部>。头部包含"zlbytes,zllen"相比与ziplist少了zltail；entry包括"encoding,entry_data,curlen"；尾部包含"zlend标记listpack的结尾"。由于entry中的prevlen由curlen取代，所以不再有级联更新的问题。
 - 不论是"ziplist"还是"listpack"获取len的方式都是一样的：
 	先判断头部中zllen字段存储的数值与UNIT16_MAX的关系
 	小于UNIT16_MAX，直接返回zllen
 	大于等于UNIT16_MAX需要从头到尾遍历获取元素总数
 	如果新得到的元素总数小于UINT16_MAX，重新设置zllen的数值
