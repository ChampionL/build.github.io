[TOC]

# Redis内存分配原理

Redis内存分配源码文件
zmalloc.h zmalloc.c

## 内存分配

```
void *zmalloc(size_t size) {
    void *ptr = malloc(size+PREFIX_SIZE);

    if (!ptr) zmalloc_oom_handler(size);
#ifdef HAVE_MALLOC_SIZE
    update_zmalloc_stat_alloc(zmalloc_size(ptr));
    return ptr;
#else
    *((size_t*)ptr) = size;
    update_zmalloc_stat_alloc(size+PREFIX_SIZE);
    return (char*)ptr+PREFIX_SIZE;
#endif
}
```
redis内存分配 
size为实际需要分配的内存，首先通过malloc分配。对于tcmalloc redis 1.6以上 以及jemalloc redis2.1以上。会在分配内存时候增加一个PREFIX_SIZE。看下PREFIX_SIZE定义
```
#ifdef HAVE_MALLOC_SIZE  /*tcmalloc 1.6 或者 jemalloc 2.1 以上才会加上malloc size*/
	#define PREFIX_SIZE (0)
#else
	#if defined(__sun) || defined(__sparc) || defined(__sparc__)
		#define PREFIX_SIZE (sizeof(long long))
	#else
		#define PREFIX_SIZE (sizeof(size_t))
	#endif
#endif
```
如果定义了HAVE_MALLOC_SIZE 那么前缀长度为0，如果未定义则是sizeof(size_t)，size_t一般是系统子长，32位机器是4字节，64位机器是8字节。malloc会自动对齐。
如果定义了，那么直接返回指针即可。如果没有定义，那么将前面size_t值设置为申请内存大小。
注意到不管是哪种方式都调用了update_zmalloc_stat_alloc，该函数用来准确地更新**used_memory**内存字段，redis使用的内存量。
```
#define update_zmalloc_stat_alloc(__n) do { \
    size_t _n = (__n); \
    if (_n&(sizeof(long)-1)) _n += sizeof(long)-(_n&(sizeof(long)-1)); \
    if (zmalloc_thread_safe) { \
        update_zmalloc_stat_add(_n); \
    } else { \
        used_memory += _n; \
    } \
} while(0)
```
zmalloc_thread_safe 如果是1那么执行线程安全的增加内存计费量
如果是0那么就直接在used_memory 上增加内存。不过在redis3.0里面，在开始启动时候调用*zmalloc_enable_thread_safeness* 将该值设置为1.所以内存计算量是线程安全地增加。

可以看到_n&()sizeof(long)-1) 等价于_n % sizeof(long) 其实就是在取余数，但是位操作显然更加高效。
这一一步不是在将申请内存大小进行对齐，malloc会把内存进行对齐。这一步实际是在做malloc实际真实分配的内存。因为malloc会对齐，所以实际分配内存实际大于等于申请的内存，所以这一步是在准确计算申请的内存。
接着我们再来看下update_zmalloc_stat_add操作
```
#ifdef HAVE_ATOMIC
#define update_zmalloc_stat_add(__n) __sync_add_and_fetch(&used_memory, (__n))
#define update_zmalloc_stat_sub(__n) __sync_sub_and_fetch(&used_memory, (__n))
#else
#define update_zmalloc_stat_add(__n) do { \
    pthread_mutex_lock(&used_memory_mutex); \
    used_memory += (__n); \
    pthread_mutex_unlock(&used_memory_mutex); \
} while(0)

#define update_zmalloc_stat_sub(__n) do { \
    pthread_mutex_lock(&used_memory_mutex); \
    used_memory -= (__n); \
    pthread_mutex_unlock(&used_memory_mutex); \
} while(0)

#endif
```
可以看到如果定义了HAVE_ATOMIC会采用__sync_add_and_fetch进行原子加以及原子减操作。
如果未定义，又要保证线程安全，那么就会进行加锁操作，采用pthread库进行加锁，解锁。
那么HAVE_ATOMIC是如何定义的呢？我们接着看下。
```
#if (__i386 || __amd64) && __GNUC__
#define GNUC_VERSION (__GNUC__ * 10000 + __GNUC_MINOR__ * 100 + __GNUC_PATCHLEVEL__)
#if (GNUC_VERSION >= 40100) || defined(__clang__)
#define HAVE_ATOMIC
#endif
#endif
```
可以看到如果是i386或者amd64架构，且gcc版本大于4.0.1或者采用clang编译器，那么是支持原子操作的。

[do while 解释](http://www.spongeliu.com/415.html)


zmalloc zcalloc zremalloc 三者功能类似，和上述malloc一样。
malloc分配的内存可能是空的，如果之前被使用过，那么也可能是旧的数据。
calloc会对分配内存进行置0。
realloc对内存进程重新内存分配和释放。

