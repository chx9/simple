---
title: redis sds
categories:
  - redis
slug: redis-sds
output:
  blogdown::html_page:
    toc: true

---

# 介绍

sds 简称 simple dynamic string，意思就是简单的动态字符串，类似于 C++中的 string，可以动态调整长度。

# 定义

他的定义非常简单

```c
typedef char *sds;

/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

这里sdshdr5、sdshdr8 等 hdr使用不同类型的 int 定义的长度和内存大小，目的也是节省一些内存。

`__attribute__ ((__packed__)) ` 是 gcc 编译器扩展，他告诉编译器取消结构体内的默认内存对齐优化，按照每个成员大小实际进行紧凑排列，节省内存空间。

`char buf[]`是 sds 实际上的 data，它是一个柔性数组（flexible array member），C99 标准引入的功能，柔性数组在结构体中不占用任何空间，只作为一个符号存在，但是结构体和数组在内存中是连续存储的，只需一次内存分配。

可以看出 sds 是一种紧凑节省内存，连续的结构，他将字符串的属性和存储数据放在同一块连续的内存中。 有两大优点：
1. 方便转换不同类型的 sds 

2. cache 优化，在 CPU 会连续读取内存，sds 的结构中 sdshdr 和 data 是连续的，可以降低 cache miss

# 创建

可以通过 sdsnewlen 函数创建 sds，该函数调用了_sdsnewlen

```c
/* Create a new sds string with the content specified by the 'init' pointer
 * and 'initlen'.
 * If NULL is used for 'init' the string is initialized with zero bytes.
 * If SDS_NOINIT is used, the buffer is left uninitialized;
 *
 * The string is always null-terminated (all the sds strings are, always) so
 * even if you create an sds string with:
 *
 * mystring = sdsnewlen("abc",3);
 *
 * You can print the string with printf() as there is an implicit \0 at the
 * end of the string. However the string is binary safe and can contain
 * \0 characters in the middle, as the length is stored in the sds header. */
sds _sdsnewlen(const void *init, size_t initlen, int trymalloc) {
    void *sh;

    char type = sdsReqType(initlen);
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type);
    size_t bufsize;

    assert(initlen + hdrlen + 1 > initlen); /* Catch size_t overflow */
    sh = trymalloc?
        s_trymalloc_usable(hdrlen+initlen+1, &bufsize) :
        s_malloc_usable(hdrlen+initlen+1, &bufsize);
    if (sh == NULL) return NULL;

    adjustTypeIfNeeded(&type, &hdrlen, bufsize);
    return sdsnewplacement(sh, bufsize, type, init, initlen);
}
```

`_sdsnewlen` 计算了 hdr 的 size，之后申请了`hdrlen+initlen+1`的内存，这里多申请的1 字节是为了字符串结尾的`\0`。

```c

/* Initializes an SDS within pre-allocated buffer. Like, placement new in C++. 
 * 
 * Parameters:
 * - `buf`    : A pre-allocated buffer for the SDS.
 * - `bufsize`: Total size of the buffer (>= `sdsReqSize(initlen, type)`). Can use 
 *              a larger `bufsize` than required, but usable size won't be greater 
 *              than `sdsTypeMaxSize(type)`. 
 * - `type`   : The SDS type. Can assist `sdsReqType(length)` to compute the type.
 * - `init`   : Initial string to copy, or `SDS_NOINIT` to skip initialization.
 * - `initlen`: Length of the initial string.
 * 
 * Returns:
 * - A pointer to the SDS inside `buf`. 
 */
sds sdsnewplacement(char *buf, size_t bufsize, char type, const char *init, size_t initlen) {
    assert(bufsize >= sdsReqSize(initlen, type));
    int hdrlen = sdsHdrSize(type);
    size_t usable = bufsize - hdrlen - 1;
    sds s = buf + hdrlen;
    unsigned char *fp = ((unsigned char *)s) - 1; /* flags pointer. */

    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            debugAssert(usable <= sdsTypeMaxSize(type));
            sh->alloc = usable;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            debugAssert(usable <= sdsTypeMaxSize(type));
            sh->alloc = usable;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            debugAssert(usable <= sdsTypeMaxSize(type));
            sh->alloc = usable;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            debugAssert(usable <= sdsTypeMaxSize(type));
            sh->alloc = usable;
            *fp = type;
            break;
        }
    }
    if (init == SDS_NOINIT)
        init = NULL;
    else if (!init)
        memset(s, 0, initlen);
    else if (initlen) 
        memcpy(s, init, initlen);

    s[initlen] = '\0';
    return s;
}
```

最后调用的`sdsnewplacement`函数中赋值了 flag、alloac 和 flag，最后确保字符串安全性，在结尾处加上`\0`

# 销毁

```c
/* Free an sds string. No operation is performed if 's' is NULL. */
void sdsfree(sds s) {
    if (s == NULL) return;
    s_free((char*)s-sdsHdrSize(s[-1]));
}
```

利用柔性数组特点，通过(char*)s-sdsHdrSize(s[-1]) 拿到 sds 连续内存的起始位置，进行释放。连续内存的方式可以通过一次释放释放所有内存。
