# slab分配器与管理区页框分配器的接口
当slab分配器创建一个新的slab，它依赖管理区页帧分配器来获取一组空闲连续页帧。为了达到这个目的，它调用`kmem_getpages()`函数
`kmem_getpages`的函数实现在[mm/slab.c line889](https://elixir.bootlin.com/linux/v2.6.11/source/mm/slab.c#L889)中
```c
static void *kmem_getpages(kmem_cache_t *cachep, int flags, int nodeid)
/* 
三个参数的含义如下
kmem_cache_t *cachep：
	需要额外页帧的缓存的缓存描述符 请求的页框数由cachep->gfporder决定
int flags
	如何申请页框的标识，这组标识与放在高速缓存描述符的gfplags字段中的专用高速缓存分配标志相结合
int nodeid
	如果是NUMA则有效，标识从哪个节点申请页框
*/
```

1. 根据nodeid标识决定如何分配页框
	```c
	flags |= cachep->gfpflags;
	if (likely(nodeid == -1)) {
		page = alloc_pages(flags, cachep->gfporder);
	} else {
		page = alloc_pages_node(nodeid, flags, cachep->gfporder);
	}
	```
2. 如果已经创建了slab高速缓存并且SLAB_RECLAIM_ACCOUNT标志已经置位,那么当内核检查是否有足够的内存来满足一些用户态请求时，分配给slab的页框将被记录为可回收的页。
	```c
	if (!page)
			return NULL;
	if (cachep->flags & SLAB_RECLAIM_ACCOUNT)
			atomic_add(i, &slab_reclaim_pages);
	```

3. 函数还将所分配页框的页描述符中的PG_slab标志置位。
	```c
	while (i--) {
			SetPageSlab(page);
			page++;
		}
	```


在相反的操作中，通过调用`kmem_freepages`函数可以释放分配给salb的页框
`kmem_freepages`的函数实现在[mm/slab.c line919](https://elixir.bootlin.com/linux/v2.6.11/source/mm/slab.c#L919)中
```c
static void kmem_freepages(kmem_cache_t *cachep, void *addr)
{
	unsigned long i = (1<<cachep->gfporder);
	struct page *page = virt_to_page(addr);
	const unsigned long nr_freed = i;

	while (i--) {
		if (!TestClearPageSlab(page))
			BUG();
		page++;
	}
	sub_page_state(nr_slab, nr_freed);
	if (current->reclaim_state)
		current->reclaim_state->reclaimed_slab += nr_freed;
	free_pages((unsigned long)addr, cachep->gfporder);
	if (cachep->flags & SLAB_RECLAIM_ACCOUNT) 
		atomic_sub(1<<cachep->gfporder, &slab_reclaim_pages);
}
```
这个函数从线性地址addr开始释放页框，这些页框曾分配给由cachep标识的高速缓存中的slab。

如果当前进程正在执行内存回收（`current->reclaim_state`非NULL），`reclaim_state->reclaimed_slab`就会适当的增加，于是刚释放的页就能通过页框回收算法被记录下来。