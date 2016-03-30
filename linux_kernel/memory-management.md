Memory Management
=================

# Page Frame Management

## Page Descriptors

State information of a page frame is kept in a page descriptor of type `page` defined in `include/linux/mm.h`, and page descriptors are stored in the mem_map array define in `mm/memory.h`.

```c
struct page {
    page_flags_t flags;   /* Array of flags, also encodes the zone number */
    atomic_t _count;   /* Page frame's reference counter */
    atomic_t _mapcount;   /* Number of Page Table entries that refer to the page frame */
    unsigned long private;
    struct address_space *mapping;
    pgoff_t index;   /* Our offset within mapping. */
    struct list_head lru;       /* Pointers to the least recently used doubly linked list of pages */
};
```

`flags` field includes up to 32 flags that describe the status of the page frame, which are defined in `include/linux/page-flags.h`:

- `PG_locked`: The page is locked; for instance, it is involved in a disk I/O operation.
- `PG_referenced`: The page has been recently accessed.
- `PG_dirty`: The page has been modified.
- `PG_lru`: The page is in the active or inactive page list.
- `PG_active`: The page is in the active page list.
- `PG_slab`: The page frame is included in a slab.
- `PG_reserved`: The page frame is reserved for kernel code or is unusable.
- `PG_writeback`: The page is being written to disk by means of the `writepage` method.
- ...and more

For each `PG_xyz` flag, the kernel defines some macros that manipulate its value. Usually, the `PageXyz` macro returns the value of the flag, while the `SetPageXyz` and `ClearPageXyz` macro set and clear the corresponding bit, respectively. For exmaple,

```c
#define PageDirty(page)     test_bit(PG_dirty, &(page)->flags)
#define SetPageDirty(page)  set_bit(PG_dirty, &(page)->flags)
#define TestSetPageDirty(page)  test_and_set_bit(PG_dirty, &(page)->flags)
#define ClearPageDirty(page)    clear_bit(PG_dirty, &(page)->flags)
#define TestClearPageDirty(page) test_and_clear_bit(PG_dirty, &(page)->flags)
```

## Non-Uniform Memory Access (NUMA)

Disregarding the role of the hardware caches, we expect the time required for a CPU to access a memory location to be essentially the same, regardless of the location’s physical address and the CPU. Unfortunately, this assumption is not true in some architectures.

Linux 2.6 supports the *Non-Uniform Memory Access (NUMA)* model, in which the access times for different memory locations from a given CPU may vary. The physical memory of the system is partitioned in several *nodes*. The time needed by a given CPU to access pages within a single node is the same. However, this time might not be the same for two different CPUs.

The physical memory inside each node can be split into several zones, and each node has a descriptor of type `pg_data_t` defined in `include/linux/mmzone.h`. All node descriptors are stored in a singly linked list, whose first element is pointed to by the `pgdat_list` variable, which is defined in `mm/page_alloc.c`.

```c
typedef struct pglist_data {
    struct zone node_zones[MAX_NR_ZONES];   /* Array of zone descriptors of the node */
    struct zonelist node_zonelists[GFP_ZONETYPES];
    int nr_zones;   /* Number of zones in the node */
    struct page *node_mem_map;   /* Array of page descriptors of the node */
    struct bootmem_data *bdata;
    unsigned long node_start_pfn;   /* Index of the first page frame in the node */
    unsigned long node_present_pages;   /* Size of the memory node, excluding holes (in page frames) */
    unsigned long node_spanned_pages;   /* Size of the node, including holes (in page frames) */
    int node_id;
    struct pglist_data *pgdat_next;   /* Next item in the memory node list */
    wait_queue_head_t kswapd_wait;
    struct task_struct *kswapd;
    int kswapd_max_order;
} pg_data_t;
```

Even if NUMA support is not compiled in the kernel, Linux makes use of a single node that includes all system physical memory. Thus, the `pgdat_list` variable points to a list consisting of a single element — the node 0 descriptor — stored in the `contig_page_data` variable. This approach makes the memory handling code more portable.

## Memory Zones

Real computer architectures have hardware constraints that may limit the way page frames can be used. In particular, the Linux kernel must deal with two hardware constraints of the 80×86 architecture:

- The Direct Memory Access (DMA) processors for old ISA buses have a strong limitation: they are able to address only the first 16 MB of RAM.
- In modern 32-bit computers with lots of RAM, the CPU cannot directly access all physical memory because the linear address space is too small.

To cope with these two limitations, Linux 2.6 partitions the physical memory of every memory node into **three *zones***. In the 80×86 UMA architecture the zones are:

- `ZONE_DMA`: Contains page frames of memory below **16 MB**
- `ZONE_NORMAL`: Contains page frames of memory at and above 16 MB and below 896 MB
- `ZONE_HIGHMEM`: Contains page frames of memory at and above 896 MB

Each memory zone has its own descriptor of type `zone`, which is defined in `include/linux/mmzone.h`.

```c
struct zone {
    unsigned long       free_pages;   /* Number of free pages in the zone */
    unsigned long       pages_min, pages_low, pages_high;
    unsigned long       lowmem_reserve[MAX_NR_ZONES];

    struct per_cpu_pageset  pageset[NR_CPUS];

    spinlock_t      lock;
    struct free_area    free_area[MAX_ORDER];   /* Identifies the blocks of free page frames in the zone  */

    spinlock_t      lru_lock;
    struct list_head    active_list;   /* List of active pages in the zone */
    struct list_head    inactive_list;   /* List of inactive pages in the zone */
    unsigned long       nr_scan_active;
    unsigned long       nr_scan_inactive;
    unsigned long       nr_active;   /* Number of pages in the zone’s active list */
    unsigned long       nr_inactive;   /* Number of pages in the zone’s inactive list */
    unsigned long       pages_scanned;
    int         all_unreclaimable;

    int temp_priority;
    int prev_priority;

    wait_queue_head_t   * wait_table;   /* Hash table of wait queues of processes waiting for one of the pages of the zone */
    unsigned long       wait_table_size;
    unsigned long       wait_table_bits;

    struct pglist_data  *zone_pgdat;
    struct page     *zone_mem_map;   /* Pointer to first page descriptor of the zone */
    unsigned long       zone_start_pfn;   /* Index of the first page frame of the zone */

    unsigned long       spanned_pages;   /* Total size of zone in pages, including holes */
    unsigned long       present_pages;   /* Total size of zone in pages, excluding holes */

    char            *name;
}
```

## The Zoned Page Frame Allocator

The kernel subsystem that handles the memory allocation requests for groups of **contiguous page frames** is called the *zoned page frame allocator*.

![Components of the zoned page frame allocator](http://i.imgur.com/KcdFwim.png)

In the case of allocation requests, the component searches a memory zone that includes a group of contiguous page frames that can satisfy the request. Inside each zone, page frames are handled by a component named *buddy system*. To get better system performance, a small number of page frames are kept in cache to quickly satisfy the allocation requests for single page frames.

### Requesting and releasing page frames

Page frames can be requested by using six slightly different functions and macros, and most of them are defined in `include/linux/gfp.h`:

- `alloc_pages(gfp_mask, order)`
  - Macro used to request **2^order** contiguous page frames.
  - Returns the address of the **descriptor** of the first allocated page frame.
- `alloc_page(gfp_mask)`
  - Macro used to get a single page frame
  - Expands to `alloc_pages(gfp_mask, 0)`
  - Returns the address of the **descriptor** of the allocated page frame.
- `__get_free_pages(gfp_mask, order)`
  - similar to `alloc_pages()`
  - Returns the **linear address** of the first allocated page
- `__get_free_page(gfp_mask)`
  - Macro used to get a single page frame
  - Expands to `__get_free_pages(gfp_mask, 0)`
- `get_zeroed_page(gfp_mask)` in `mm/page_alloc.c`
  - Function used to obtain a page frame filled with zeros.
  - Invokes `alloc_pages(gfp_mask | __GFP_ZERO, 0)`
- `__get_dma_pages(gfp_mask, order)`
  - Macro used to get page frames suitable for DMA
  - Expands to `__get_free_pages(gfp_mask | __GFP_DMA, order)`

The parameter `gfp_mask` is a group of flags that specify how to look for free page frames. Among them, the `__GFP_DMA` and `__GFP_HIGHMEM` flags are called *zone modifiers*; they specify the zones searched by the kernel while looking for free page frames.

## The Buddy System Algorithm

The technique adopted by Linux to solve the **external fragmentation** problem is based on the well-known *buddy system* algorithm. All free page frames are grouped into 11 lists of blocks that contain groups of 1, 2, 4, 8, 16, 32, 64, 128, 256, 512, and 1024 contiguous page frames, respectively.

The reverse operation, releasing blocks of page frames, gives rise to the name of this algorithm. The kernel attempts to merge pairs of free buddy blocks of size b together into a single block of size 2b. Two blocks are considered buddies if:

- Both blocks have the same size, say b.
- They are located in contiguous physical addresses.
- The physical address of the first page frame of the first block is a multiple of 2 × b × 2^12.

### Data structures

Linux 2.6 uses a different buddy system for each zone. Thus, in the 80 × 86 architecture, there are 3 buddy systems. Each buddy system relies on the following main data structures:

- Each zone is concerned with a subset of the `mem_map` elements. The first element in the subset and its number of elements are specified, respectively, by the `zone_mem_map` and `size` fields of the zone descriptor.
- An array consisting of eleven elements of type `free_area`, one element for each group size. The array is stored in the `free_area` field of the zone descriptor.

### Allocating a block

The `__rmqueue()` function in `mm/page_alloc.c` is used to find a free block in a zone. It performs a cyclic search through each list for an available block, starting with the list for the requested order and continuing if necessary to larger orders:

```c
static struct page *__rmqueue(struct zone *zone, unsigned int order)
{
    struct free_area * area;
    unsigned int current_order;
    struct page *page;

    for (current_order = order; current_order < MAX_ORDER; ++current_order) {
        area = zone->free_area + current_order;
        if (list_empty(&area->free_list))
            continue;

        page = list_entry(area->free_list.next, struct page, lru);
        list_del(&page->lru);
        rmv_page_order(page);
        area->nr_free--;
        zone->free_pages -= 1UL << order;
        return expand(zone, page, order, current_order, area);
    }

    return NULL;
}
```

In `expand()` function, when it becomes necessary to use a block of 2^k page frames to satisfy a request for 2^h page frames (h < k), the program allocates the first 2^h page frames and iteratively reassigns the last 2^k – 2^h page frames to the `free_area` lists that have indexes between h and k:

```c
static inline struct page *
expand(struct zone *zone, struct page *page,
    int low, int high, struct free_area *area)
{
    unsigned long size = 1 << high;

    while (high > low) {
        area--;
        high--;
        size >>= 1;
        BUG_ON(bad_range(zone, &page[size]));
        list_add(&page[size].lru, &area->free_list);
        area->nr_free++;
        set_page_order(&page[size], high);
    }
    return page;
}
```

### Freeing a block

The `__free_pages_bulk()` function in `mm/page_alloc.c` implements the buddy system strategy for freeing page frames.

The function performs a cycle executed at most 10–order times, once for each possibility for **merging** a block with its buddy. The function starts with the smallest-sized block and moves up to the top size:

```c
static inline void __free_pages_bulk (struct page *page, struct page *base,
        struct zone *zone, unsigned int order)
{
    unsigned long page_idx;
    struct page *coalesced;
    int order_size = 1 << order;

    if (unlikely(order))
        destroy_compound_page(page, order);

    page_idx = page - base;

    BUG_ON(page_idx & (order_size - 1));
    BUG_ON(bad_range(zone, page));

    zone->free_pages += order_size;
    while (order < MAX_ORDER-1) {
        struct free_area *area;
        struct page *buddy;
        int buddy_idx;

        buddy_idx = (page_idx ^ (1 << order));
        buddy = base + buddy_idx;
        if (bad_range(zone, buddy))
            break;
        if (!page_is_buddy(buddy, order))
            break;
        /* Move the buddy up one level. */
        list_del(&buddy->lru);
        area = zone->free_area + order;
        area->nr_free--;
        rmv_page_order(buddy);
        page_idx &= buddy_idx;
        order++;
    }
    coalesced = base + page_idx;
    set_page_order(coalesced, order);
    list_add(&coalesced->lru, &zone->free_area[order].free_list);
    zone->free_area[order].nr_free++;
}

static inline int page_is_buddy(struct page *page, int order)
{
       if (PagePrivate(page)           &&
           (page_order(page) == order) &&
           !PageReserved(page)         &&
            page_count(page) == 0)
               return 1;
       return 0;
}
```

## The Per-CPU Page Frame Cache

The kernel often requests and releases **single** page frames. To boost system performance, each memory zone defines a *per-CPU page frame cache*. Each per-CPU cache includes some **pre-allocated** page frames to be used for single memory requests issued by the local CPU.

There are two caches for each memory zone and for each CPU: a *hot cache*, which stores page frames whose contents are likely to be included in the CPU’s hard- ware cache, and a *cold cache*.

The main data structure implementing the per-CPU page frame cache is an array of `per_cpu_pageset` data structures stored in the `pageset` field of the memory zone descriptor.

```c
struct per_cpu_pages {
    int count;   /* number of pages in the list */
    int low;   /* low watermark, refill needed */
    int high;   /* high watermark, emptying needed */
    int batch;   /* chunk size for buddy add/remove */
    struct list_head list;   /* the list of pages */
};

struct per_cpu_pageset {
    struct per_cpu_pages pcp[2];   /* 0: hot.  1: cold */
#ifdef CONFIG_NUMA
    unsigned long numa_hit;   /* allocated in intended node */
    unsigned long numa_miss;   /* allocated in non intended node */
    unsigned long numa_foreign;   /* was intended here, hit elsewhere */
    unsigned long interleave_hit;   /* interleaver prefered this zone */
    unsigned long local_node;   /* allocation from local node */
    unsigned long other_node;   /* allocation from other node */
#endif
} ____cacheline_aligned_in_smp;
```

The kernel monitors the size of the both the hot and cold caches by using two watermarks: if the number of page frames falls below the `low` watermark, the kernel replenishes the proper cache by allocating batch single page frames from the buddy system; otherwise, if the number of page frames rises above the `high` watermark, the kernel releases to the buddy system `batch` page frames in the cache.

### Allocating page frames through the per-CPU page frame caches

The `buffered_rmqueue()` function in `mm/page_alloc.c` allocates page frames in a given memory zone. It makes use of the per-CPU page frame caches to handle single page frame requests.

```c
static struct page *
buffered_rmqueue(struct zone *zone, int order, int gfp_flags)
{
    unsigned long flags;
    struct page *page = NULL;
    int cold = !!(gfp_flags & __GFP_COLD);

    if (order == 0) {   /* single page frame */
        struct per_cpu_pages *pcp;

        pcp = &zone->pageset[get_cpu()].pcp[cold];
        local_irq_save(flags);
        if (pcp->count <= pcp->low)   /* replenished */
            pcp->count += rmqueue_bulk(zone, 0,
                        pcp->batch, &pcp->list);   /* allocate from buddy system */
        if (pcp->count) {
            page = list_entry(pcp->list.next, struct page, lru);   /* get a page from cache's list */
            list_del(&page->lru);
            pcp->count--;
        }
        local_irq_restore(flags);
        put_cpu();
    }

    if (page == NULL) {   /* request several contiguous frames, or frame cache is empty */
        spin_lock_irqsave(&zone->lock, flags);
        page = __rmqueue(zone, order);   /* allocate from buddy system */
        spin_unlock_irqrestore(&zone->lock, flags);
    }

    if (page != NULL) {
        BUG_ON(bad_range(zone, page));
        mod_page_state_zone(zone, pgalloc, 1 << order);
        prep_new_page(page, order);

        if (gfp_flags & __GFP_ZERO)
            prep_zero_page(page, order, gfp_flags);

        if (order && (gfp_flags & __GFP_COMP))
            prep_compound_page(page, order);
    }
    return page;
}
```

### Releasing page frames to the per-CPU page frame caches

In order to release a single page frame to a per-CPU page frame cache, the kernel makes use of the `free_hot_page()` and `free_cold_page()` functions. Both of them are simple wrappers for the `free_hot_cold_page()` function:

```c
static void fastcall free_hot_cold_page(struct page *page, int cold)
{
    struct zone *zone = page_zone(page);
    struct per_cpu_pages *pcp;
    unsigned long flags;

    arch_free_page(page, 0);

    kernel_map_pages(page, 1, 0);
    inc_page_state(pgfree);
    if (PageAnon(page))
        page->mapping = NULL;
    free_pages_check(__FUNCTION__, page);
    pcp = &zone->pageset[get_cpu()].pcp[cold];
    local_irq_save(flags);
    if (pcp->count >= pcp->high)
        pcp->count -= free_pages_bulk(zone, pcp->batch, &pcp->list, 0);   /* release page frames to buddy system */
    list_add(&page->lru, &pcp->list);   /* add remaining frames into cache's list */
    pcp->count++;
    local_irq_restore(flags);
    put_cpu();
}
```

Noted that in the current version of the Linux 2.6 kernel, no page frame is ever released to the cold cache: the kernel always assumes the freed page frame is hot with respect to the hardware cache.

## The Zone Allocator

The *zone allocator* is the frontend of the kernel page frame allocator. This component must locate a memory zone that includes a number of free page frames large enough to satisfy the memory request.

Every request for a group of contiguous page frames is eventually handled by executing the `alloc_pages` macro. This macro, in turn, ends up invoking the `__alloc_pages()` function in `mm/page_alloc.c`, which is the core of the zone allocator. This function scans every memory zone included in the zonelist data structure:

```c
struct page * fastcall
__alloc_pages(unsigned int gfp_mask, unsigned int order,
        struct zonelist *zonelist)
{
    /* ... */

    zones = zonelist->zones;   /* the list of zones suitable for gfp_mask */

    /* ... */

    classzone_idx = zone_idx(zones[0]);

 restart:
    /* Go through the zonelist once, looking for a zone with enough free */
    for (i = 0; (z = zones[i]) != NULL; i++) {

        if (!zone_watermark_ok(z, order, z->pages_low,
                       classzone_idx, 0, 0))
            continue;

        page = buffered_rmqueue(z, order, gfp_mask);
        if (page)
            goto got_pg;
    }
}
```

# Memory Area Management

This section deals with *memory areas* — that is, with sequences of memory cells having contiguous physical addresses and an arbitrary length.

The buddy system algorithm adopts the page frame as the basic memory area. This is fine for dealing with relatively **large** memory requests. However, for small memory areas request , there is a new problem called *internal fragmentation*. It is caused by a mismatch between the size of the memory request and the size of the memory area allocated to satisfy the request.

## The Slab Allocator

The slab allocator groups objects into *caches*. Each cache is a "store" of objects of the **same type**. For instance, when a file is opened, the memory area needed to store the corresponding "open file" object is taken from a slab allocator cache named *filp* (for "file pointer").

The area of main memory that contains a cache is divided into *slabs*; each slab consists of one or more contiguous page frames that contain both allocated and free objects.

![The slab allocator components](http://i.imgur.com/zS01IBX.png)

## Cache Descriptor

Each cache is described by a structure of type `kmem_cache_t` (which is equivalent to the type `struct kmem_cache_s` defined in `mm/slab.c`).

```c
struct kmem_cache_s {
/* 1) per-cpu data, touched during every alloc/free */
    struct array_cache  *array[NR_CPUS];   /* Per-CPU array of pointers to local caches of free objects */
    unsigned int        batchcount;
    unsigned int        limit;
/* 2) touched by every alloc & free from the backend */
    struct kmem_list3   lists;
    unsigned int        objsize;
    unsigned int        flags;   /* constant flags */
    unsigned int        num;   /* # of objs per slab */
    unsigned int        free_limit;   /* upper limit of objects in the lists */
    spinlock_t      spinlock;

/* 3) cache_grow/shrink */
    /* order of pgs per slab (2^n) */
    unsigned int        gfporder;

    /* force GFP flags, e.g. GFP_DMA */
    unsigned int        gfpflags;

    size_t          colour;   /* cache colouring range */
    unsigned int        colour_off;   /* colour offset */
    unsigned int        colour_next;   /* cache colouring */
    kmem_cache_t        *slabp_cache;
    unsigned int        slab_size;
    unsigned int        dflags;   /* dynamic flags */

    /* constructor func */
    void (*ctor)(void *, kmem_cache_t *, unsigned long);

    /* de-constructor func */
    void (*dtor)(void *, kmem_cache_t *, unsigned long);

/* 4) cache creation/removal */
    const char      *name;
    struct list_head    next;

/* 5) statistics */
  /* ... */
};
```

For the `lists` field of the `kmem_cache_t` descriptor:

```c
struct kmem_list3 {
    struct list_head    slabs_partial;   /* Doubly linked circular list of slab descriptors with both free and non-free objects */
    struct list_head    slabs_full;   /* Doubly linked circular list of slab descriptors with no free objects */
    struct list_head    slabs_free;   /* Doubly linked circular list of slab descriptors with free objects only */
    unsigned long   free_objects;
    int     free_touched;
    unsigned long   next_reap;
    struct array_cache  *shared;   /* Pointer to a local cache shared by all CPUs */
};
```

## Slab Descriptor

Each slab of a cache has its own descriptor of type `slab`:

```c
struct slab {
    struct list_head    list;   /* Pointers for one of the three doubly linked list of slab descriptors */
    unsigned long       colouroff;   /* Offset of the first object in the slab  */
    void            *s_mem;     /* Address of first object (either allocated or free) in the slab */
    unsigned int        inuse;      /* Number of objects in the slab that are currently used (not free) */
    kmem_bufctl_t       free;   /* Index of next free object in the slab, or BUFCTL_END if there are no free objects left */
};
```

Slab descriptors can be stored in two possible places:

- External slab descriptor
  - Stored outside the slab, in one of the **general caches** not suitable for ISA DMA pointed to by `cache_sizes`.
- Internal slab descriptor
  - Stored inside the slab, at the beginning of the first page frame assigned to the slab.
  - The slab allocator chooses this solution when the size of the objects is smaller than 512MB or when internal fragmentation leaves enough space for the slab descriptor and the object descriptors.

![Relationship between cache and slab descriptors](http://i.imgur.com/70SAPFH.png)

## General and Specific Caches

Caches are divided into two types:

- *General caches*
  - Used only by the slab allocator for its own purposes.
  - A first cache called `kmem_cache` whose objects are the cache descriptors of the remaining caches used by the kernel. The `cache_cache` variable contains the descriptor of this special cache.
  - Several additional caches contain general purpose memory areas. The range of the memory area sizes typically includes 13 geometrically distributed sizes. A table called `malloc_sizes` (whose elements are of type `cache_sizes`) points to 26 cache descriptors associated with memory areas of size 32, 64, 128, 256, 512, 1,024, 2,048, 4,096, 8,192, 16,384, 32,768, 65,536, and 131,072 bytes.
  - The `kmem_cache_init()` function is invoked during system initialization to set up the general caches.
- *Specific caches*
  - Used by the remaining parts of the kernel.
  - Created by the `kmem_cache_create()` function

The names of all general and specific caches can be obtained at runtime by reading `/proc/slabinfo`; this file also specifies the number of free objects and the number of allocated objects in each cache.

## Interfacing the Slab Allocator with the Zoned Page Frame Allocator

When the slab allocator creates a new slab, it relies on the zoned page frame allocator to obtain a group of free contiguous page frames. For this purpose, it invokes the `kmem_getpages()` function in `mm/slab.c`:

```c
static void *kmem_getpages(kmem_cache_t *cachep, int flags, int nodeid)
{
    struct page *page;
    void *addr;
    int i;

    flags |= cachep->gfpflags;
    if (likely(nodeid == -1)) {
        page = alloc_pages(flags, cachep->gfporder);
    } else {
        page = alloc_pages_node(nodeid, flags, cachep->gfporder);
    }
    if (!page)
        return NULL;
    addr = page_address(page);

    i = (1 << cachep->gfporder);
    if (cachep->flags & SLAB_RECLAIM_ACCOUNT)
        atomic_add(i, &slab_reclaim_pages);
    add_page_state(nr_slab, i);
    while (i--) {
        SetPageSlab(page);
        page++;
    }
    return addr;
}
```

In the reverse operation, page frames assigned to a slab can be released by invoking the `kmem_ freepages()` function:

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

## Allocating a Slab to a Cache

A newly created cache does not contain a slab and therefore does not contain any free objects. New slabs are assigned to a cache only when both of the following are true:

- A request has been issued to allocate a new object.
- The cache does not include a free object.

Under these circumstances, the slab allocator assigns a new slab to the cache by invoking `cache_grow()`:

```c
static int cache_grow (kmem_cache_t * cachep, int flags, int nodeid)
{
    /* ... */

    /* Get mem for the objs. */
    if (!(objp = kmem_getpages(cachep, flags, nodeid)))
        goto failed;

    /* Get slab management. */
    if (!(slabp = alloc_slabmgmt(cachep, objp, offset, local_flags)))
        goto opps1;

    set_slab_attr(cachep, slabp, objp);

    cache_init_objs(cachep, slabp, ctor_flags);   /* apply construcutor method */

    if (local_flags & __GFP_WAIT)
        local_irq_disable();
    check_irq_off();
    spin_lock(&cachep->spinlock);

    /* Make slab active. */
    list_add_tail(&slabp->list, &(list3_data(cachep)->slabs_free));   /* add to the end of the fully free slab list */
    STATS_INC_GROWN(cachep);
    list3_data(cachep)->free_objects += cachep->num;
    spin_unlock(&cachep->spinlock);
    return 1;
opps1:
    kmem_freepages(cachep, objp);
failed:
    if (local_flags & __GFP_WAIT)
        local_irq_disable();
    return 0;
}
```

## Releasing a Slab from a Cache

Slabs can be destroyed in two cases:

- There are too many free objects in the slab cache.
- A timer function invoked periodically determines that there are fully unused slabs that can be released.

In both cases, the `slab_destroy()` function is invoked to destroy a slab and release the corresponding page frames to the zoned page frame allocator:

```c
static void slab_destroy (kmem_cache_t *cachep, struct slab *slabp)
{
    void *addr = slabp->s_mem - slabp->colouroff;

  /* ... */

    if (cachep->dtor) {
        int i;
        for (i = 0; i < cachep->num; i++) {
            void* objp = slabp->s_mem+cachep->objsize*i;
            (cachep->dtor)(objp, cachep, 0);   /* apply destructor method */
        }
    }

    if (unlikely(cachep->flags & SLAB_DESTROY_BY_RCU)) {
        struct slab_rcu *slab_rcu;

        slab_rcu = (struct slab_rcu *) slabp;
        slab_rcu->cachep = cachep;
        slab_rcu->addr = addr;
        call_rcu(&slab_rcu->head, kmem_rcu_free);
    } else {
        kmem_freepages(cachep, addr);   /* return page frames to buddy system */
        if (OFF_SLAB(cachep))
            kmem_cache_free(cachep->slabp_cache, slabp);
    }
}
```

## Object Descriptor

Each object has a short descriptor of type `kmem_bufctl_t`. Object descriptors are stored in an array placed right after the corresponding slab descriptor. They can be stored in two possible ways:

- External object descriptors
  - Stored outside the slab, in the general cache pointed to by the `slabp_cache` field of the cache descriptor. The size of the memory area, and thus the particular general cache used to store object descriptors, depends on the number of objects stored in the slab (`num` field of the cache descriptor).
- Internal object descriptors
  - Stored inside the slab, right before the objects they describe.

![Relationships between slab and object descriptors](http://i.imgur.com/QLyIO0t.png)

According to `include/asm-x86_64/types.h`, an object descriptor is simply an `unsigned short` integer, which is meaningful only when the object is free. It contains the index of the next free object in the slab, thus implementing a simple list of free objects inside the slab.

```c
typedef unsigned short kmem_bufctl_t;
```

## Allocating a Slab Object

New objects may be obtained by invoking the `kmem_cache_alloc()` function in `mm/slab.c`:

```c
void * kmem_cache_alloc (kmem_cache_t *cachep, int flags)
{
    return __cache_alloc(cachep, flags);
}

static inline void * __cache_alloc (kmem_cache_t *cachep, int flags)
{
    unsigned long save_flags;
    void* objp;
    struct array_cache *ac;

    cache_alloc_debugcheck_before(cachep, flags);

    local_irq_save(save_flags);
    ac = ac_data(cachep);
    if (likely(ac->avail)) {
        STATS_INC_ALLOCHIT(cachep);
        ac->touched = 1;
        objp = ac_entry(ac)[--ac->avail];   /* address of that free object */
    } else {
        STATS_INC_ALLOCMISS(cachep);
        objp = cache_alloc_refill(cachep, flags);   /* no free objects in the local cache */
    }
    local_irq_restore(save_flags);
    objp = cache_alloc_debugcheck_after(cachep, flags, objp, __builtin_return_address(0));
    return objp;
}

static inline void ** ac_entry(struct array_cache *ac)
{
  /* The local cache array is stored right after the ac descriptor */
    return (void**)(ac+1);
}
```

## Freeing a Slab Object

The `kmem_cache_free()` function releases an object previously allocated by the slab allocator to some kernel function.

```c
void kmem_cache_free (kmem_cache_t *cachep, void *objp)
{
    unsigned long flags;

    local_irq_save(flags);
    __cache_free(cachep, objp);
    local_irq_restore(flags);
}

static inline void __cache_free (kmem_cache_t *cachep, void* objp)
{
    struct array_cache *ac = ac_data(cachep);

    check_irq_off();
    objp = cache_free_debugcheck(cachep, objp, __builtin_return_address(0));

    if (likely(ac->avail < ac->limit)) {
        STATS_INC_FREEHIT(cachep);
        ac_entry(ac)[ac->avail++] = objp;   /* add to local cache */
        return;
    } else {
        STATS_INC_FREEMISS(cachep);
        cache_flusharray(cachep, ac);   /* deplete the local cache */
        ac_entry(ac)[ac->avail++] = objp;
    }
}
```

## General Purpose Objects

Infrequent requests for memory areas are handled through a group of general caches whose objects have geometrically distributed sizes ranging from a minimum of 32 to a maximum of 131,072 bytes. Objects of this type are obtained by invoking the `kmalloc()`, then calling `__kmalloc()` function:

```c
void * __kmalloc (size_t size, int flags)
{
    struct cache_sizes *csizep = malloc_sizes;

    for (; csizep->cs_size; csizep++) {   /* locate the nearest power-of-2 size */
        if (size > csizep->cs_size)
            continue;
        return __cache_alloc(flags & GFP_DMA ?
             csizep->cs_dmacachep : csizep->cs_cachep, flags);
    }
    return NULL;
}
```

Objects obtained by invoking `kmalloc()` can be released by calling `kfree()`:

```c
void kfree (const void *objp)
{
    kmem_cache_t *c;
    unsigned long flags;

    if (!objp)
        return;
    local_irq_save(flags);
    kfree_debugcheck(objp);
    c = GET_PAGE_CACHE(virt_to_page(objp));
    __cache_free(c, (void*)objp);
    local_irq_restore(flags);
}
```

# Noncontiguous Memory Area Management

If the requests for memory areas are infrequent, it makes sense to consider an allocation scheme based on **noncontiguous page frames accessed through contiguous linear addresses**. The main advantage of this schema is to avoid external fragmentation, while the disadvantage is that it is necessary to fiddle with the kernel Page Tables.

![The linear address interval starting from PAGE_OFFSET](http://i.imgur.com/iPZTEhh.png)

The `VMALLOC_START` macro defines the starting address of the linear space reserved for noncontiguous memory areas, while `VMALLOC_END` defines its ending address.

## Descriptors of Noncontiguous Memory Areas

Each noncontiguous memory area is associated with a descriptor of type `vm_struct` defined in `include/linux/vmalloc.h`:

```c
struct vm_struct {
    void            *addr;   /* Linear address of the first memory cell of the area */
    unsigned long       size;   /* Size of the area plus 4,096 (inter-area safety interval) */
    unsigned long       flags;
    struct page     **pages;   /* Pointer to array of nr_pages pointers to page descriptors */
    unsigned int        nr_pages;   /* Number of pages filled by the area */
    unsigned long       phys_addr;
    struct vm_struct    *next;
};
```

## Allocating a Noncontiguous Memory Area

The `vmalloc()` function in `mm/vmalloc.c` allocates a noncontiguous memory area to the kernel:

```c
void *vmalloc(unsigned long size)
{
       return __vmalloc(size, GFP_KERNEL | __GFP_HIGHMEM, PAGE_KERNEL);
}

void *__vmalloc(unsigned long size, int gfp_mask, pgprot_t prot)
{
    struct vm_struct *area;
    struct page **pages;
    unsigned int nr_pages, array_size, i;

    size = PAGE_ALIGN(size);
    if (!size || (size >> PAGE_SHIFT) > num_physpages)
        return NULL;

  /* Looks for a free range of linear addresses between VMALLOC_START and VMALLOC_END */
    area = get_vm_area(size, VM_ALLOC);
    if (!area)
        return NULL;

    nr_pages = size >> PAGE_SHIFT;
    array_size = (nr_pages * sizeof(struct page *));

    area->nr_pages = nr_pages;
    if (array_size > PAGE_SIZE)
        pages = __vmalloc(array_size, gfp_mask, PAGE_KERNEL);
    else
        pages = kmalloc(array_size, (gfp_mask & ~__GFP_HIGHMEM));   /* request a group of contiguous page frames large enough to contain an array of page descriptor pointers */
    area->pages = pages;
    if (!area->pages) {
        remove_vm_area(area->addr);
        kfree(area);
        return NULL;
    }
    memset(area->pages, 0, array_size);

    for (i = 0; i < area->nr_pages; i++) {
        area->pages[i] = alloc_page(gfp_mask);   /* allocate a page frame */
        if (unlikely(!area->pages[i])) {
            area->nr_pages = i;
            goto fail;
        }
    }

    /* Associated with a linear address included in the interval of contiguous linear addresses */
    if (map_vm_area(area, prot, &pages))
        goto fail;
    return area->addr;

fail:
    vfree(area->addr);
    return NULL;
}
```

Linux 2.6 also features a `vmap()` function, which maps page frames already allocated in a noncontiguous memory area: essentially, this function receives as its parameter an array of pointers to page descriptors, invokes `get_vm_area()` to get a new `vm_struct` descriptor, and then invokes `map_vm_area()` to map the page frames. The function is thus similar to `vmalloc()`, but it **does not allocate page frames**.

## Releasing a Noncontiguous Memory Area

The `vfree()` function releases noncontiguous memory areas created by `vmalloc()` or `vmalloc_32()`, while the `vunmap()` function releases memory areas created by `vmap()`. Both functions have one parameter — the address of the initial linear address of the area to be released; they both rely on the `__vunmap()` function to do the real work:

```c
void __vunmap(void *addr, int deallocate_pages)
{
    struct vm_struct *area;

    if (!addr)
        return;

    if ((PAGE_SIZE-1) & (unsigned long)addr) {
        printk(KERN_ERR "Trying to vfree() bad address (%p)\n", addr);
        WARN_ON(1);
        return;
    }

    area = remove_vm_area(addr);   /* clear the kernel’s page table entries */
    if (unlikely(!area)) {
        printk(KERN_ERR "Trying to vfree() nonexistent vm area (%p)\n",
                addr);
        WARN_ON(1);
        return;
    }

    if (deallocate_pages) {
        int i;

        for (i = 0; i < area->nr_pages; i++) {
            if (unlikely(!area->pages[i]))
                BUG();
            __free_page(area->pages[i]);   /* release the page frame to the zoned page frame allocator */
        }

        if (area->nr_pages > PAGE_SIZE/sizeof(struct page *))
            vfree(area->pages);
        else
            kfree(area->pages);
    }

    kfree(area);   /* release the vm_struct descriptor */
    return;
}
```
