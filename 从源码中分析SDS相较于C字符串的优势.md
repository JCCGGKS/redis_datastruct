
# 前言
从本篇文章开始会持续更新有关"redis数据结构源码"的分析。[分析的源码是redis7.2.2版本的，有时候会结合之前的版本]。由于能力有限，有些地方可能有些错误，还望指正。
<font color='red' > 在分析"intset"的源码的发现。在7.2.2版本，数据类型"set"的底层编码多了一种"listpack"，但是之前的版本并没有。本着严谨的态度重新查看了源码以及配置文件，并作出相应的修改。主要修改的地方就是各个版本的数据类型和底层编码的对应图以及编码之间的阈值。</font>
# Type  && Encoding
redis的每种数据结构都被封装在"redisObject"中，先来看一下"redisObject"长什么样子。

```c
struct redisObject {
    unsigned type:4;//表示数据结构的类型,占4bits
    unsigned encoding:4;//数据结构底层使用的编码,占4bits
    unsigned lru:LRU_BITS; //用于LRU或者LFU内存淘汰算法,占24bits
    int refcount;//引用计数,占4bytes,引用计数变为0对象被销毁,内存被回收
    void *ptr;//指向具体的数据,占8bytes
};
```
接下来看一下"type"都有哪些[以下是五种基本的数据结构]

```c
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */
```
接下来看一下数据结构对应的"encoding"
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/f751db13669e2629b5eb6ff02db345ba.png)

redis在3.2版本时引入了"quicklist"[也就是linkedlist+ziplist]
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/9e6038f419c98aa65fb6f47849837e31.png)


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/0d82d68a7e6625dc0d4aebbb74bedd4e.png#pic_center)
redis在5.0版本引入"listpack"

 - 虽然5.0版本引入了"listpack"解决了"ziplist"级联更新的问题，但是hash和zset结构使用的仍然是ziplist。listpack目前只使用在新增加的Stream数据结构中。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d8610f593c09a1296b48e46764eec310.png#pic_center)


redis在7版本将"ziplist"替换为"listpack"
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d79d14f8bbdaac1d09ff96c94961d4c6.png)


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/e80bfaf01fd91237f8b08ffc8651759c.png#pic_center)

接下来分析"sds"的源码
# sds
首先看一下sds的结构
redis-7.2.2\src\sds.h

```c
typedef char * sds;//sds的本质就是char*指向一块连续的内存
```
sds分为“sdshdr5”、“sdshdr8”、“sdshdr16”、“sdshdr32”、“sdshdr64”五种结构。其中"sdshdr5"从未使用过。

```c
/* Note: sdshdr5 is never used, we just access the flags byte directly.However is here to document the layout of type 5 SDS strings. */
//sdshdr5从未被使用过，只是用它来直接访问flags字节
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
```

```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; //1byte
    uint8_t alloc;//1byte,不包括头部和终止符
    unsigned char flags; /* 3 lsb of type, 5 unused bits  1byte*/
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used  2byte*/
    uint16_t alloc; /* excluding the header and null terminator  2byte*/
    unsigned char flags; /* 3 lsb of type, 5 unused bits  1byte*/
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used  4byte*/
    uint32_t alloc; /* excluding the header and null terminator  4byte*/
    unsigned char flags; /* 3 lsb of type, 5 unused bits  1byte*/
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used  8byte*/
    uint64_t alloc; /* excluding the header and null terminator  8byte*/
    unsigned char flags; /* 3 lsb of type, 5 unused bits  1byte*/
    char buf[];
};
```
以上几种结构的布局很相似都是包含"len\alloc\flags\buf"不同的是记录“len\alloc”的字节数不相同。
其中flags的类型如下

```c
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
```


# encoding
sds底层有三种编码方式
redis-7.2.2\src\server.h

```c
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
```

接下来依次看一下编码方式的选择
## createStringObject
[redis-7.2.2\src\object.c]

```c
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
robj *createStringObject(const char *ptr, size_t len) {
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}
```
从以上代码中可以看出，会根据字符串的长度选择合适的编码方式[embstr  or  raw]。长度的阈值为44，为什么是44呢，稍后再做解释。接下来看一下"createEmbeddedStringObject"和"createRawStringObject"
### createEmbeddedStringObject

```c
/* Create a string object with encoding OBJ_ENCODING_EMBSTR, that is
 * an object where the sds string is actually an unmodifiable string
 * allocated in the same chunk as the object itself. */
 /*
创建一个底层编码为"OBJ_ENCODING_EMBSTR"的string对象
实际上，"embstr"编码的字符串是不能修改的字符串
和"redisobject"分配在同一个内存块中
 */
robj *createEmbeddedStringObject(const char *ptr, size_t len) {
	//将redisObject和sdshdr8分配在同一个chunk中
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);//这里加1指的是结尾的'\0'字符
    struct sdshdr8 *sh = (void*)(o+1);

    o->type = OBJ_STRING;
    o->encoding = OBJ_ENCODING_EMBSTR;
    o->ptr = sh+1;
    o->refcount = 1;
    o->lru = 0;

    sh->len = len;
    sh->alloc = len;//新创建的字符串的cap和len保持一致
    sh->flags = SDS_TYPE_8;
    //const char *SDS_NOINIT = "SDS_NOINIT";
    //将传入参数ptr指向内存的数据存储到buf中
    if (ptr == SDS_NOINIT)
        sh->buf[len] = '\0';
    else if (ptr) {
        memcpy(sh->buf,ptr,len);
        sh->buf[len] = '\0';
    } else {
        memset(sh->buf,0,len+1);
    }
    return o;
}
```
---------------
#### 总结
 - <font color='red'>编码为embstr的字符串是无法被修改的字符串</font>
 - 对于编码为embstr的字符串使用的sds结构为"sdshdr8"
 - 新创建的字符串的cap和len保持一致，并且字符的结尾含有'\0'结束符[猜想这里应该是为了兼容c中字符数组并且必要时候能够使用c的库函数]
 - 编码为embstr的字符串，redisObject和sds分配在同一个chunk中，如下图所示

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ed9e51678f91615e43e134014a74b87b.png)


### createRawStringObject

```c
/* Create a string object with encoding OBJ_ENCODING_RAW, that is a plain
 * string object where o->ptr points to a proper sds string. */
//创建一个底层编码为"raw"的字符串,可选择的sds的结构为
//[sdshdr5,sdshdr8,sdshdr16,sdshdr32,sdshdr64]
robj *createRawStringObject(const char *ptr, size_t len) {
    return createObject(OBJ_STRING, sdsnewlen(ptr,len));
    //createObject根据传入的参数"type、sds"创建新的"redisObject"
    //sdsnewlen根据传入的参数"char*、size_t"创建新的sds
}
```


```c
sds sdsnewlen(const void *init, size_t initlen) {
    return _sdsnewlen(init, initlen, 0);
}

/*
根据"init指向的content"和initlen创建一个新的sds
如果init指向的内容是NULL，则将字符串初始化为zero bytes
如果init是SDS_NOINIT,buf将不会被初始化
*/
/*
The string is always null-terminated (all the sds strings are, always) 
so even if you create an sds string with:
mystring = sdsnewlen("abc",3);
You can print the string with printf() 
as there is an implicit \0 at the end of the string.
However the string is binary safe and can contain\0 
characters in the middle, 
as the length is stored in the sds header.

将上述一段英文翻译成中文:
所有的sds string都是以空字符即'\0'结尾;
用如下的方式创建一个新的sds string:
mystring = sdsnewlen("abc",3);
可以使用"printf()"函数打印mystring，因为mystring的结尾有个隐式的'\0'字符。
但是mystring是二进制安全的并且可以在在字符串的中间包含'\0'的字符，
因为字符串的长度会被记录在sds的header中。
*/
sds _sdsnewlen(const void *init, size_t initlen, int trymalloc) {
    void *sh;
    sds s;
    //根据initlen大小获取sds的结构类型
    char type = sdsReqType(initlen);
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    //获取头部大小
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; /* flags pointer. */
    size_t usable;

    assert(initlen + hdrlen + 1 > initlen); /* Catch size_t overflow */
    /*
    	Allocate memory or panic.
    	usable is set to the usable size if non NULL. 
    	如果能够成功分配内存，那么usable=hdrlen+initlen+1
    */
    sh = trymalloc?
        s_trymalloc_usable(hdrlen+initlen+1, &usable) :
        s_malloc_usable(hdrlen+initlen+1, &usable);
    if (sh == NULL) return NULL;
    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
        memset(sh, 0, hdrlen+initlen+1);
    s = (char*)sh+hdrlen;
    fp = ((unsigned char*)s)-1;
    //正常情况下，usable和initlen保持一致
    usable = usable-hdrlen-1;
    if (usable > sdsTypeMaxSize(type))
        usable = sdsTypeMaxSize(type);
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
    }
    //将init指向的内容复制到buf中
    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0';
    return s;
}
```

```c
robj *createObject(int type, void *ptr) {
//ptr指向创建好的sds
//为redisObject分配一块新的内存空间，由此也可以得出底层编码为"raw"的redisObject和sds分开存储属于两块不同的连续内存块
    robj *o = zmalloc(sizeof(*o));
    o->type = type;
    o->encoding = OBJ_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;
    o->lru = 0;
    return o;
}
```
#### 总结

 - 底层编码为"raw"的string是可以修改的，编码为"embstr"的string若想要修改需要先将其的编码变为"raw"才可以被修改。如下图所示
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/f4f91cba2c325e3e23235704a139d972.png)

 - 根据传入的字符串的initlen从[sdshdr5,sdshdr8,sdshdr16,sdshdr32,sdshdr64]选择合适的sds头部结构
 - 所有的字符串都以'\0'作为结束符，可以使用printf()函数进行打印；二进制安全的，字符串中间可以存储'\0'字符，长度信息存储在sds_header中
 - 编码为"raw"的字符串，redisObject和sds存储在不同的内存块中，如下图所示
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/56aa62bb2803cbba93cc082629d02b7d.png)
## createStringObjectFromLongDouble

```c
#define MAX_LONG_DOUBLE_CHARS 5*1024
 /*
 从长双精度类型创建字符串对象。如果humanfriendly为非零，则不使用指数
 格式并在末尾修剪尾随的零，但是这会导致精度的损失。否则使用exp格式，并
 且不修改snprintf()的输出。“humanfriendly”选项用于INCRBYFLOAT和
 HINCRBYFLOAT。
 */
robj *createStringObjectFromLongDouble(long double value, int humanfriendly) {
    char buf[MAX_LONG_DOUBLE_CHARS];
    int len = ld2string(buf,sizeof(buf),value,humanfriendly? LD_STR_HUMAN: LD_STR_AUTO);
    return createStringObject(buf,len);
}
```
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/40b844784b72224a1cc5e65e294c5abc.png)

### 总结
从以上代码可以看出“long double”类型的浮点数以“string”的形式存储，底层的编码方式为"embstr"或者“raw”
## createStringObjectFromLongLongWithOptions
```c
/* Create a string object from a long long value according to the specified flag. */
#define LL2STROBJ_AUTO 0       /* automatically create the optimal string object */
#define LL2STROBJ_NO_SHARED 1  /* disallow shared objects */
#define LL2STROBJ_NO_INT_ENC 2 /* disallow integer encoded objects. */
robj *createStringObjectFromLongLongWithOptions(long long value, int flag) {
    robj *o;
    //#define OBJ_SHARED_INTEGERS 10000
    if (value >= 0 && value < OBJ_SHARED_INTEGERS && flag == LL2STROBJ_AUTO) {
    //value值处于[0,10000)进行共享
        o = shared.integers[value];
    } else {
    //在long的范围内，用int进行编码
        if ((value >= LONG_MIN && value <= LONG_MAX) && flag != LL2STROBJ_NO_INT_ENC) {
            o = createObject(OBJ_STRING, NULL);
            o->encoding = OBJ_ENCODING_INT;
            //value值内嵌在在ptr中
            o->ptr = (void*)((long)value);
        } else {
        //超出long表示的范围则根据长度在"embstr or raw"中选择编码
            char buf[LONG_STR_SIZE];
            int len = ll2string(buf, sizeof(buf), value);
            o = createStringObject(buf, len);
        }
    }
    return o;
}
```

### 总结

 - value值在[0,10000)内共享redisobject
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/25650acc8b189e35d79adfc02fdd1ccb.png)

 - value在long范围内，底层编码为int。并且value值内嵌在ptr中
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/5ae99b32c589792f47d5b224a116defc.png)

 - value不在long范围内，根据len的长度选择合适的编码，"embstr"或者"raw"

# 相关操作
redis-7.2.2\src\sds.c
通过以上源码的分析，我们发现不论字符串以何种编码方式创建。sds的cap都和len保持一致，这是为了节约内存，因为绝大多数情况下都不会使用"append"操作。
接下来就看一下"append"操作的流程，
## sdscatlen


```c

/*
将t指向的字符串拼接到sds指向的字符串的末尾，
拼接之后，传入的sds不再有效，需要使用返回的新sds代替旧的传入的sds
*/
sds sdscatlen(sds s, const void *t, size_t len) {
	//当前sds指向字符串的长度
    size_t curlen = sdslen(s);
    //通过sdsMakeRoomFor计算拼接之后新的字符串的长度并返回新的sds
    s = sdsMakeRoomFor(s,len);
    //没有足够的空间，拼接失败
    if (s == NULL) return NULL;
    //将t指向的字符串拼接到s的末尾,s+curlen存储的是'\0'结尾符，也即是t指向的字符串会覆盖sds指向字符串的'\0'结尾符
    memcpy(s+curlen, t, len);
    //设置新sds的长度
    sdssetlen(s, curlen+len);
    //加上'\0'结尾符
    s[curlen+len] = '\0';
    return s;
}
```
接下来看一下如何"扩容"的
```c
/*
为sds分配更多的空间避免重复的进行append操作
*/
sds sdsMakeRoomFor(sds s, size_t addlen) {
    return _sdsMakeRoomFor(s, addlen, 1);
}
/* Enlarge the free space at the end of the sds string so that the caller
 * is sure that after calling this function can overwrite up to addlen
 * bytes after the end of the string, plus one more byte for null term.
 * If there's already sufficient free space, this function returns without any
 * action, if there isn't sufficient free space, it'll allocate what's missing,
 * and possibly more:
 * When greedy is 1, enlarge more than needed, to avoid need for future reallocs
 * on incremental growth.
 * When greedy is 0, enlarge just enough so that there's free space for 'addlen'.
 *
 * Note: this does not change the *length* of the sds string as returned
 * by sdslen(), but only the free buffer space we have. */
/*
扩大sds字符串末尾的可用空间，以便调用者在调用此函数后可以覆盖原字符串的addlen字节和'\0'字符
如果已经有足够的空闲空间，这个函数返回时不做任何操作，
如果没有足够的空闲空间，它将分配缺失的部分，甚至更多:
当greedy为1时，分配比需要的更多，以避免再次扩容时需要重新分配。
当greedy为0时，将其放大到足够大以便为addlen腾出空间即可。
注意:这不会改变sdslen()返回的sds字符串的长度，而只会改变我们拥有的空闲缓冲区空间也即是alloc。
*/
sds _sdsMakeRoomFor(sds s, size_t addlen, int greedy) {
    void *sh, *newsh;
    //获取旧sds的可用space,[s->alloc - s->len];
    size_t avail = sdsavail(s);
    size_t len, newlen, reqlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;
    size_t usable;

    //可用空闲空间满足要求直接返回，什么也不做
    if (avail >= addlen) return s;
    
    
	//获取旧sds字符串的长度len
    len = sdslen(s);
    
    sh = (char*)s-sdsHdrSize(oldtype);
    //拼接之后新字符串的长度
    reqlen = newlen = (len+addlen);
    
    assert(newlen > len);   /* Catch size_t overflow */
    
    //greedy=1表示需要预先多分配一些内存
    if (greedy == 1) {
    //#define SDS_MAX_PREALLOC (1024*1024) [1MB]
    //newlen小于1MB，翻倍扩容
        if (newlen < SDS_MAX_PREALLOC)
            newlen *= 2;
        else
        //需要的长度大于等于1MB，多扩容1MB
            newlen += SDS_MAX_PREALLOC;
    }
	//根据newlen计算需要使用的sds类型
    type = sdsReqType(newlen);

   
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

    hdrlen = sdsHdrSize(type);
    //多分配1byte是为了存储末尾的'\0'字符
    assert(hdrlen + newlen + 1 > reqlen);  /* Catch size_t overflow */
    //sds头部类型不变，在原地扩容内存
    if (oldtype==type) {
        newsh = s_realloc_usable(sh, hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        //获取新sds的buf
        s = (char*)newsh+hdrlen;
    } else {
    //sds头部类型改变，重新分配内存
        newsh = s_malloc_usable(hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        //拷贝旧sds中的内容到新的sds中
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        //设置len
        sdssetlen(s, len);
    }
    //计算空闲空间
    usable = usable-hdrlen-1;
    if (usable > sdsTypeMaxSize(type))
        usable = sdsTypeMaxSize(type);
        //设置alloc
    sdssetalloc(s, usable);
    return s;
}
```
### 总结
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/2ac902e27dfa818daba688b959cf1ee0.png#pic_center)
# 阈值44
接下来分析一下为什么编码"embstr"和编码"raw"的长度界限值为44
先来回顾一下"redisObject"的样子

```c
struct redisObject {
    unsigned type:4;//表示数据结构的类型,占4bits
    unsigned encoding:4;//数据结构底层使用的编码,占4bits
    unsigned lru:LRU_BITS; //用于LRU或者LFU内存淘汰算法,占24bits
    int refcount;//引用计数,占4bytes,引用计数变为0对象被销毁,内存被回收
    void *ptr;//指向具体的数据,占8bytes
};
```
通过上述代码，可以计算出redisObject所占用的内存大小为:
>4bits+4bits+24bits+4bytes+8bytes=16bytes

通过"createEmbeddedStringObject"的源码分析，发现"embstr"编码的字符串使用的sds结构为sdshdr8

```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; //1byte
    uint8_t alloc; //1byte
    unsigned char flags; //1byte
    char buf[];
};
```
通过上述代码，可以计算出"sds"中除了存储数据的buf，其余所占用的内存空间大小为:
>1byte+1byte+1byte=3bytes

通过对""和""的源码分析发现，不论哪种编码[embstr or raw]的字符串都是以'\0'作为结束字符的，所以还需要占用额外的1byte存储结尾的标识符。


整体计算下来，一个完整的字符串除了存储的数据所占用的内存空间大小为：
>16bytes+3bytes+1byte=20bytes


而，redis的内存分配器"jemalloc\tcmalloc"等分配内存大小的单位是"2\4\8\16\32\64"字节等，为了能容纳一个完整的"embstr"字符串，至少会分配32字节的内存，至多会分配64字节的内存。如果64字节不够存储的话，redis就将其视为一个大字符串，不再使用"embstr"进行编码，而是采用"raw"进行编码。

所以"embstr"和"raw"之间的阈值为44字节
>64bytes-20bytes=44bytes









# sds VS C字符串

 - redis规定字符串的长度为"512MB"。


**sds的本质还是char *，但是相比与"C字符串"多了一些字段"len，alloc，flag"，优势如下**

 - 以O(1)的时间复杂度快速获取字符串的长度。因为字符串的长度会记录在sds的header中。
 - 二进制安全。头部结构中记录字符串的长度使得字符串中间位置可以存储'\0'字符。从而可以存储任何类型的数据，比如文本，视频，图片，程序代码等。
 - 不同的数据对应不同的底层编码，更加灵活，更加节省内存。
 - 拼接(扩充)安全。字符串拼接之前会先加查是否有足够的空闲空间，没有的话先扩充内存然后才进行拼接操作。
