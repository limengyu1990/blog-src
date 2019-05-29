---
title: Redis源码阅读-字符串
date: 2018-11-17 22:31:43
tags:
    - redis
---
>Redis动态字符串数据结构的定义及实现在sds.h和sds.c文件中。

* 字符串定义
* 字符串实现
* 方法实现

### 字符串定义
我们平时在使用Redis时，经常使用的数据结构就是字符串，例如：
```shell
127.0.0.1:6379> set name Redis
OK
127.0.0.1:6379> get name
"Redis"
127.0.0.1:6379> del name
(integer) 1
```
那么字符串在Redis是如何定义的呢？我们可以在sds.h文件中找到答案。
在该文件中，我们可以看到Redis字符串数据结构SDS由两部分组成，sds指针和sdshdr头部类型，
其中sdshdr类型定义了5种(sdshdr5/sdshdr8/sdshdr16/sdshdr32/sdshdr64)，主要是为了针对不同长度的字符串，节省内存：
```c
typedef char *sds;

struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len;
    uint8_t alloc;
    unsigned char flags;
    char buf[];
};

/*flags值定义*/
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4

//掩码
#define SDS_TYPE_MASK 7
#define SDS_TYPE_BITS 3

/*通过buf指针获取sds头指针，T为sds头类型值，s为buf指针
 *这里##会将两个字符串连接起来，如:T为8，则sdshdr##T为sdshrd8
 *sizeof(struct sdshdr##T)为结构体sdshdr8占用的字节数，这里为3字节
 */
#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));

#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))

#define SDS_TYPE_5_LEN(f) ((f)>>SDS_TYPE_BITS)
```
![](img/sds.png)
len   标识当前字节数组的长度，不包含字符串结束标识。
alloc 标识当前字节数组分配的内存大小，不包含字符串结束标识。
flags 低3位标识当前使用的是哪个sdshdr类型。
buf   保存字符串值和结束标识。
*sds  指向buf数组的起始地址
为SDS增加一个头部类型可以提高一些操作的效率，例如用O(1)的复杂度就可以从头部中取到字符串长度。
sdshdr头部类型是通过结构体定义的，默认情况下会进行内存对齐优化，即结构体分配的内存是内部最大元素的整数倍，例如sdshdr32将会分配12字节。
这里使用__attribute__ ((__packed__))关闭内存对齐优化，从而按照实际占用字节数来对齐，即sdshdr32将会分配9字节，节省了3字节。内存紧凑，使用sds-1就可以得到flags字段，进而得到其头部类型。没有内存对齐，cpu寻址效率就会降低，Redis是在内存分配前做了一些操作，解决内存对齐的，后面会看到。
buf数组初始化时不占用内存空间，使得头部内存和存储字符串的内存地址连续，另外结尾隐含一个'\0',而SDS是以len字段来判断是否是否到达字符串末尾的，因此在字符串中间可以出现'\0'，即SDS字符串是二进制安全的。

### 字符串实现
创建新字符串使用的是sds.c中的sdsnewlen函数：
```c
// 使用init指针指向的内容和initlen创建一个新的字符串
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    
    // buf数组起始地址
    sds s;
    
    // 根据长度选择合适的sds头部类型
    char type = sdsReqType(initlen);

    // 用type 8创建空字符串方便追加
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    
    // 根据sds头部类型获取头部大小
    int hdrlen = sdsHdrSize(type);
    
    // flag指针
    unsigned char *fp;

    // 为sds分配内存（后面会分析s_malloc）
    // 内存大小为：sds头部大小 + 存储字符串的长度initlen + 末尾空字符大小1字节
    sh = s_malloc(hdrlen+initlen+1); 

    if (!init)
        // 内存初始化为0（后面会分析memset）
        memset(sh, 0, hdrlen+initlen+1);
    // 内存分配失败
    if (sh == NULL) return NULL;

    // buf数组的起始地址
    s = (char*)sh+hdrlen;
    
    // buf数组起始地址-1，即为flags字段
    fp = ((unsigned char*)s)-1;

    // 初始化sds头部的len,alloc,flags字段
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            //根据buf起始地址获取指向sds头部的起始地址的指针
            SDS_HDR_VAR(8,s);
	    // 设置len字段值为initlen
            sh->len = initlen;
	    // 设置alloc字段值为initlen
            sh->alloc = initlen;
	    // 设置flags字段类型为type
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
    }
    // 初始化buf数组
    if (initlen && init)
        // 拷贝init到buf数组（后面会分析memcpy）
        memcpy(s, init, initlen);

    // 添加末尾空字符标识
    s[initlen] = '\0';
    return s;
}

/**
 * 根据字符串长度获取合适的头部类型
 * @param string_size 字符串初始长度
 * @return
 */
static inline char sdsReqType(size_t string_size) {
    if (string_size < 1<<5) //32
        return SDS_TYPE_5;
    if (string_size < 1<<8) //256
        return SDS_TYPE_8;
    if (string_size < 1<<16) //65536
        return SDS_TYPE_16;
#if (LONG_MAX == LLONG_MAX) //等于有符号long最大值
    if (string_size < 1ll<<32) //4gb
        return SDS_TYPE_32;
    return SDS_TYPE_64;
#else
    return SDS_TYPE_32;
#endif
}

/**
 * 通过字符串头部类型获取其头大小
 * @param type 字符串类型 0-4
 * @return
 */
static inline int sdsHdrSize(char type) {
    // type & SDS_TYPE_MASK = type
    // 例如：sdshdr8， 1 & 7 = 1
    switch(type&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return sizeof(struct sdshdr5);
        case SDS_TYPE_8:
            //sizeof计算结构体大小，取消了内存对齐
            return sizeof(struct sdshdr8); // 3字节
        case SDS_TYPE_16:
            return sizeof(struct sdshdr16); // 5字节
        case SDS_TYPE_32:
            return sizeof(struct sdshdr32); // 9字节
        case SDS_TYPE_64:
            return sizeof(struct sdshdr64); // 17字节
    }
    return 0;
}
```
关于内存操作的函数，将在内存操作小节介绍，这里就不在进行介绍了。
### 方法实现
