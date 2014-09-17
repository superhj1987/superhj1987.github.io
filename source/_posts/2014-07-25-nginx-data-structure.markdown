---
layout: post
title: "Nginx源码分析之基本数据结构"
date: 2014-07-25 17:08:02 +0800
comments: true
categories: 
---

### 引言

nginx实现中有很多结构体，一般命名为ngx_xxx_t。这些结构体分散在许多头文件中。src/core/ngx_core.h中把几乎所有的头文件都集合起来。也因此造成了nginx各部分源代码的耦合。但实际上nginx各个部分逻辑划分还是很明确的，整体上是一种松散的结构。

作者之所以重复造了这些轮子，无非是为了追求高效。查看这些数据结构的源码，的确是设计的比较精巧，也保证了对内存足够小的占用以及各种操作的高效。

<!--more-->

### 数据结构

nginx实现中有很多结构体，一般命名为ngx_XXX_t。这些结构体分散在许多头文件中。src/core/ngx_core.h中把几乎所有的头文件都集合起来。也因此造成了nginx各部分源代码的耦合。但实际上nginx各个部分逻辑划分还是很明确的，整体上是一种松散的结构。

* ngx_str_t

		typedef struct{
			size_t len;
			u_char *data;
		}ngx_str_t;
		
这是nginx对字符串的实现，源码在ngx_string.h中。len指的是字符串的长度（不包括\0），data指向字符串。这种设计一方面，在计算字符创长度时只需要读取len字段即可，另一方面可以重复引用一段字符串内存。

常用api:
	
		#define ngx_string(str) { sizof(str) - 1},(u_char *) str } //从一个普通字符串构造出一个nginx字符串，用sizeof计算长度，故参数必须是一个常量字符串。

		#define ngx_null_string {0,NULL}
		
		ngx_strncmp(s1,s2,n)
		
		ngx_strcm(s1,s2)
		 
* ngx_pool_t

		struct ngx_pool_s {
    		ngx_pool_data_t       d;
    		size_t                max;
    		ngx_pool_t           *current;
    		ngx_chain_t          *chain;
    		ngx_pool_large_t     *large;
    		ngx_pool_cleanup_t   *cleanup;
    		ngx_log_t            *log;
		};

这个数据结构在nginx中是一个非常重要的数据结构。用来管理一系列的资源（如内存、文件等
，使得对这些资源的使用和释放统一进行。这个是在c语言编程中值得借鉴的一个东西，代码中如果到处都是malloc和free的话，不仅会导致内存泄露，也会使代码难以阅读和维护。
		
* ngx_array_t

		struct ngx_array_s {
    		void        *elts; //指向实际的存储区域
    		ngx_uint_t   nelts; //数组实际元素个数
    		size_t       size; //数组单个元素的大小，单位是字节
    		ngx_uint_t   nalloc; //数组的容量
    		ngx_pool_t  *pool; //该数组用来分配内存的内存池
		};
	
* ngx_hash_t
	* ngx_hash_t不像其他的hash表的实现，可以插入删除元素，只能一次初始化。
	* 解决冲突使用的是开链法，但实际上是开了一段连续的存储空间，和数组差不多。   			
			ngx_int_t ngx_hash_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names,ngx_uint_t nelts);//ngx_hash_t的初始化。
			
	
			ngx_hash_init_t提供了初始化一个hash表所需要的一些基本信息
			typedef struct {
    			ngx_hash_t       *hash; //指向hash表
    			ngx_hash_key_pt   key; //指向从字符串生成hash值的hash函数。默认的实现为ngx_hash_key_lc
    			ngx_uint_t        max_size; //hash表中的桶的个数
    			ngx_uint_t        bucket_size; //每个桶的最大限制大小，单位是字节
    			char             *name; //hash表的名字
    			ngx_pool_t       *pool; //hash表分配内存使用的pool
    			ngx_pool_t       *temp_pool; //使用的临时pool,初始化完成后，可以释放和销毁
			} ngx_hash_init_t;
			
			typedef struct {    			ngx_str_t         key;    			ngx_uint_t        key_hash;    			void             *value;			} ngx_hash_key_t;
			void *ngx_hash_find(ngx_hash_t *hash, ngx_uint_t key, u_char *name, size_t len); //在hash里面查找key对应的value。			
			
* ngx_chain_t
			
nginx的filter模块在处理从别的filter模块或者是handler模块传递过来的数据，数据一个链表的形式（ngx_chain_t）进行传递。
			
			struct ngx_chain_s {
    			ngx_buf_t    *buf;
    			ngx_chain_t  *next;
			};
			
创建ngx_chain_t对象

			ngx_chain_t *ngx_alloc_chain_link(ngx_pool_t *pool);
			
释放一个ngx_chain_t类型的对象。如果要释放整个chain，则迭代此链表，对每个节点使用此宏即可。

			释放一个ngx_chain_t类型的对象。如果要释放整个chain，则迭代此链表，对每个节点使用此宏即可。
			
* ngx_buf_t

ngx_buf_t是ngx_chain_t的数据结点

		struct ngx_buf_s {
    		u_char          *pos;
    		u_char          *last;
    		off_t            file_pos;
    		off_t            file_last;

    		u_char          *start;         /* start of buffer */
    		u_char          *end;           /* end of buffer */
    		ngx_buf_tag_t    tag;
    		ngx_file_t      *file;
    		ngx_buf_t       *shadow;


    		/* the buf's content could be changed */
    		unsigned         temporary:1;

    		/*
     		* the buf's content is in a memory cache or in a read only memory
     		* and must not be changed
     		*/
    		unsigned         memory:1;

    		/* the buf's content is mmap()ed and must not be changed */
    		unsigned         mmap:1;

    		unsigned         recycled:1;
    		unsigned         in_file:1;
   			unsigned         flush:1;
    		unsigned         sync:1;
    		unsigned         last_buf:1;
    		unsigned         last_in_chain:1;

    		unsigned         last_shadow:1;
    		unsigned         temp_file:1;

    		/* STUB */ int   num;
		};
		
* ngx_list_t

和普通的链表实现相比，它的节点是一个固定大小的数组。在初始化的时候，我们需要设定元素需要占用的空间大小，每个节点数组的容量大小。在添加元素到这个list里面的时候，会在最尾部的节点里的数组上添加元素，如果这个节点的数组存满了，就再增加一个新的节点到这个list里面去。

		typedef struct {
    		ngx_list_part_t  *last; //指向该链表的最后一个节点
    		ngx_list_part_t   part; //指向该链表首个存放具体元素的节点
    		size_t            size; //链表中存放的具体元素所需内存大小
    		ngx_uint_t        nalloc; //每个节点所含的固定大小的数组的容量
    		ngx_pool_t       *pool; //该list使用的分配内存的pool
		} ngx_list_t;
		
		struct ngx_list_part_s {
    		void             *elts; //节点中存放具体元素的内存的开始地址   
    		ngx_uint_t        nelts; //节点中已有元素个数，不能大于 nalloc
    		ngx_list_part_t  *next; //指向下一个节点
		};
		
		ngx_list_t *ngx_list_create(ngx_pool_t *pool, ngx_uint_t n, size_t size); //创建一个ngx_list_t类型的对象,并对该list的第一个节点分配存放元素的内存空间。

		pool:	分配内存使用的pool。
		n:	每个节点固定长度的数组的长度。
		size:	存放的具体元素的个数。
		
* ngx_queue_t 
		
		struct ngx_queue_s {
    		ngx_queue_t  *prev;
    		ngx_queue_t  *next;
		};
		
链表节点的数据成员并没有生命在链表节点的结构体中，只是声明了前向和后向指针。使用的时候需要定义一个哨兵节点。具体存放数据的节点称之为数据节点。对于数据节点，需要在数据结构体中加入一个类型为ngx_queue_s的域。使用下面的函数进行数据插入，其中x为数据节点的queue_t域。
		
		#define ngx_queue_insert_head(h, x)                         \
    		(x)->next = (h)->next;                                  \
    		(x)->next->prev = x;                                    \
    		(x)->prev = h;                                          \
    		(h)->next = x

		#define ngx_queue_insert_after   ngx_queue_insert_head

		#define ngx_queue_insert_tail(h, x)                          \
    		(x)->prev = (h)->prev;                                   \
    		(x)->prev->next = x;                                     \
    		(x)->next = h;                                           \
    		(h)->prev = x
    	获得数据时，使用ngx_queue_data()宏。
    	#define ngx_queue_data(q, type, link)                        \
    		(type *) ((u_char *) q - offsetof(type, link))