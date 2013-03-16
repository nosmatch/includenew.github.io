---
layout: post
title: "关于Memcache内存管理模型的理解"
date: 2012-08-31 07:52
comments: true
categories: 
---

<script async class="speakerdeck-embed" data-id="504049be5ec53c000202daa6" data-ratio="1.299492385786802" src="//speakerdeck.com/assets/embed.js"></script>

##说在前面

本文不包含为什么使用memcache，以及如何使用memcache等基础知识。相关知识请查阅各类手册。
另，为便于理解，最好手头准备一份memcache的源码，本文使用的是目前最新的1.4.4版本源码，可自行到github上clone。

## Item、Chunk、Page、Slab

### Data Item

    +---------------------------------------+
    |  key-value | cas | suffix | item head |  
    +---------------------------------------+

Item指实际存放到memcache中的数据对象结构，除key-value数据外，还包括memcache自身对数据对象的描述信息（Item=key+value+后缀长+32byte结构体）

### Chunk

Chunk指Memcache用来存放Data Item的最小单元，同一个Slab中的chunk大小是固定的。

    +------------------------------+
    |   data item    | empty space |
    +------------------------------+

## Page

    +-------------------------------------+
    |  chunk1 | chunk2 | chunk3 | chunk4  |
    +-------------------------------------+

每个Slab中按照Page来申请内存，Page的大小默认为1M，可以通过-l参数调整，最小1k，最大128m.

### Slab

    +--------------------------------+
    |  Page1 | Page2 | Page3 | Page4 |
    +--------------------------------+

Memcache将分配给它的内存（-m 参数指定，默认64m）按照Chunk大小不同，划分为多个slab。

他们三者的关系如下图所示:


                     Chunk
                       ^                                                         
    +------------------|------------------------------------------------------------+
    |   Memory         |                                                            | 
    |  +---------------|---------------------------------------------------------+  |
    |  |      +--------|---------------------+  +------------------------------+ |  |
    |  |      |Page1 +-|---+ +-----+ +-----+ |  |Page2 +-----+ +-----+ +-----+ | |  |
    |  | Slab |(1M)  | 96B | | 68B | | 72B | |  |(1M)  | 92B | | 76B | | 84B | | |  | 
    |  |  1   |      +-----+ +-----+ +-----+ |  |      +-----+ +-----+ +-----+ | |  |
    |  |      +------------------------------+  +------------------------------+ |  |
    |  +-------------------------------------------------------------------------+  |
    |                                                                               |
    |  +-------------------------------------------------------------------------+  |
    |  |      +------------------------------+  +------------------------------+ |  |
    |  |      |Page1 +------+    +------+    |  |Page2 +------+    +-------+   | |  |
    |  | Slab | (1M) | 128B |    | 120B |    |  |(1M)  | 128B |    | 97B   |   | |  |
    |  |   2  |      +------+    +------+    |  |      +------+    +-------+   | |  |
    |  |      +------------------------------+  +------------------------------+ |  |
    |  +-------------------------------------------------------------------------+  |
    +-------------------------------------------------------------------------------+


##Slab内存分配
###slab初始化

Memcache启动时会进行slab初始化（参见slabs.c中slabs_init()函数），默认最小的chunksize为80（查看源码会发现settings中chunk_size默认为48，但是实际还需要加上一个32bytes的item结构体），可以通过-n参数调整，按照然后按照factor（默认为1.25，可以通过-f参数调整）(_关于参数更多的memcache默认参数可以参考memcache.c中settings的设置_)比例递增，划分出多个不同chunk大小的slab空间，即slab1的chunk大小=80，slab2的chunk大小为80\*1.25=100，slab3的chunk大小为80\*1.25*1.25=125，但最大一个一个chunk不会大于一个Page的大小（默认1M）。

	一下代码节选自 slabs.c
	 95 void slabs_init(const size_t limit, const double factor, const bool prealloc) {
	 96     int i = POWER_SMALLEST - 1;
	 97     unsigned int size = sizeof(item) + settings.chunk_size;
	 98  
	 99     mem_limit = limit;
	100  
	101     if (prealloc) {
	102         /* Allocate everything in a big chunk with malloc */
	103         mem_base = malloc(mem_limit);
	104         if (mem_base != NULL) {
	105             mem_current = mem_base;
	106             mem_avail = mem_limit;
	107         } else {
	108             fprintf(stderr, "Warning: Failed to allocate requested memory in"
	109                     " one large chunk.\nWill allocate in smaller chunks\n");
	110         }
	111     }
	112  
	113     memset(slabclass, 0, sizeof(slabclass));
	114  
	115     while (++i < POWER_LARGEST && size <= settings.item_size_max / factor) {
	116         /* Make sure items are always n-byte aligned */
	117         if (size % CHUNK_ALIGN_BYTES)
	118             size += CHUNK_ALIGN_BYTES - (size % CHUNK_ALIGN_BYTES);
	119  
	120         slabclass[i].size = size;
	121         slabclass[i].perslab = settings.item_size_max / slabclass[i].size;
	122         size *= factor;
	123         if (settings.verbose > 1) {
	124             fprintf(stderr, "slab class %3d: chunk size %9u perslab %7u\n",
	125                     i, slabclass[i].size, slabclass[i].perslab);
	126         }
	127     }
	128  
	129     power_largest = i;
	130     slabclass[power_largest].size = settings.item_size_max;
	131     slabclass[power_largest].perslab = 1;
	132     if (settings.verbose > 1) {
	133         fprintf(stderr, "slab class %3d: chunk size %9u perslab %7u\n",
	134                 i, slabclass[i].size, slabclass[i].perslab);
	135     }
	136  
	137     /* for the test suite:  faking of how much we've already malloc'd */
	138     {
	139         char *t_initial_malloc = getenv("T_MEMD_INITIAL_MALLOC");
	140         if (t_initial_malloc) {
	141             mem_malloced = (size_t)atol(t_initial_malloc);
	142         }
	143  
	144     }
	145  
	146     if (prealloc) {
	147         slabs_preallocate(power_largest);
	148     }
	149 }                             

PS：prealloc指的是直接申请一个大的chunk存放所有数据，默认是不采用这种方式的。

###数据存储过程
一个数据项的大致存储量过程可以理解为（完整代码较长，不在粘贴，具体可参见items.c中do_item_alloc()方法）：

1. 构造一个数据项结构体，计算数据项的大小，（假设默认配置下，数据项大小为102B）
2. 根据数据项的大小，找到最合适的slab，（100<102<125，所以存储在slab3中）
3. 检查该slab中是否有过期的数据，如有清理掉
4. 如果没有过期的数据项，则从当前slab中申请空间，参见slabs.c中slab_alloc()方法。
5. 如果当前slab中申请失败，则尝试根据LRU算法逐出一个数据项，默认memcache是允许逐出的，如果被设置为禁止逐出，那么这是会反生悲剧的oom了
6. 获取到item空间后将数据存储到改空间中，并追加到该slab的item列表中

一个slab的申请一个chunk空间的过程大致如下（以下代码节选自slabs.c）：

	195 static int do_slabs_newslab(const unsigned int id) { 
	196     slabclass_t *p = &slabclass[id];
	197     int len = settings.slab_reassign ? settings.item_size_max
	198         : p->size * p->perslab;
	199     char *ptr;             
	200            
	201     if ((mem_limit && mem_malloced + len > mem_limit && p->slabs > 0) ||
	202         (grow_slab_list(id) == 0) ||    
	203         ((ptr = memory_allocate((size_t)len)) == 0)) {
	204            
	205         MEMCACHED_SLABS_SLABCLASS_ALLOCATE_FAILED(id);
	206         return 0;          
	207     }      
	208            
	209     memset(ptr, 0, (size_t)len);    
	210     split_slab_page_into_freelist(ptr, id);
	211            
	212     p->slab_list[p->slabs++] = ptr; 
	213     mem_malloced += len;   
	214     MEMCACHED_SLABS_SLABCLASS_ALLOCATE(id);
	215            
	216     return 1;              
	217 }
	218  
	219 /*@null@*/ 
	220 static void *do_slabs_alloc(const size_t size, unsigned int id) {
	221     slabclass_t *p;        
	222     void *ret = NULL;      
	223     item *it = NULL;       
	224  
	225     if (id < POWER_SMALLEST || id > power_largest) {
	226         MEMCACHED_SLABS_ALLOCATE_FAILED(size, 0);
	227         return NULL;
	228     }
	229  
	230     p = &slabclass[id];
	231     assert(p->sl_curr == 0 || ((item *)p->slots)->slabs_clsid == 0);
	232  
	233     /* fail unless we have space at the end of a recently allocated page,
	234        we have something on our freelist, or we could allocate a new page */
	235     if (! (p->sl_curr != 0 || do_slabs_newslab(id) != 0)) {
	236         /* We don't have more memory available */
	237         ret = NULL;
	238     } else if (p->sl_curr != 0) {
	239         /* return off our freelist */
	240         it = (item *)p->slots;
	241         p->slots = it->next;
	242         if (it->next) it->next->prev = 0;
	243         p->sl_curr--;
	244         ret = (void *)it;
	245     }
	246  
	247     if (ret) {
	248         p->requested += size;
	249         MEMCACHED_SLABS_ALLOCATE(size, id, p->size, ret);
	250     } else {
	251         MEMCACHED_SLABS_ALLOCATE_FAILED(size, id);
	252     }
	253  
	254     return ret;
	255 }

slab优先从slots（空闲chunk空间列表）中申请空间，如果没有则尝试申请一个Page的新空间（do_slab_newslab()），申请新slab是会先判断是否进行slab_reasgin（重新分配slab空间，默认不开启）。

##内存浪费

根据上述描述，Memcache使用Slab预分配的方式进行内存管理提升了性能（减少分配内存的消耗），但是带来了内存浪费，主要体现在：

1. Data Item Size <= Chunk Size，Chunk是存储数据项的最小单元，数据项的大小必须不大于其所在的Chunk大小。也就是说76B的数据对象存入96B的Chunk中，将带来96B-76B=20B的空间浪费。

2. Memcache是按照Page申请和使用内存的，当Page大小不是Chunk的整数倍时，余下的空间将被浪费。即如果PageSize=1M，ChunkSize=1000B,那么将有1024*1024%1000=576B的空间浪费。

3. Memcache默认是不开启slab reasign的，也就是说分配已经分配给一个slab的内存空间，即使该slab不用，默认也不会分配给其他slab的

##案例分析：定长问题导致逐出

memcache的chunk分布是均匀的，这是为了通用性考虑，但是现实中一些场景chunk的分布是不均运的，例如为了减小对数据库的压力，对数据进行了全量缓存，为标识数据库中不存在的记录，向缓存中放置了一个stupidObject。这个对象大小是固定的，且该数据的量很大，导致该数据类型所在的slab占用了大量缓存空间。再一次调整对象结构时，修改了这个StupidObject大小，使其分布在另一个slab中，但是这个原分配的slab空间不会回收，空闲空间不足，导致大量逐出。
