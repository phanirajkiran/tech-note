Process Address Space
=====================

# The Process’s Address Space

The *address space* of a process consists of all linear addresses that the process is allowed to use. Each process sees a different set of linear addresses; the address used by one process bears no relation to the address used by another.

The kernel represents intervals of linear addresses by means of resources called *memory regions*, which are characterized by an initial linear address, a length, and some access rights. For reasons of efficiency, both the initial address and the length of a memory region must be multiples of **4096**, so that the data identified by each memory region completely fills up the page frames allocated to it.

The following are system calls related to memory region creation and deletion:

- `brk()`: Changes the heap size of the process
- `execve()`: Loads a new executable file, thus changing the process address space
- `_exit()`: Terminates the current process and destroys its address space
- `fork()`: Creates a new process, and thus a new address space
- `mmap()`, `mmap2()`: Creates a memory mapping for a file, thus enlarging the process address space
- `munmap()`: Destroys a memory mapping for a file, thus contracting the process address space
- `shmat()`: Attaches a shared memory region
- `shmdt()`: Detaches a shared memory region

# The Memory Descriptor

All information related to the process address space is included in an object called the *memory descriptor* of type `mm_struct` defined in `include/linux/sched.h`. This object is referenced by the `mm` field of the process descriptor `task_struct`.

```c
struct mm_struct {
    /* Pointer to the head of the list of memory region objects */
    struct vm_area_struct * mmap;

    /* Pointer to the root of the red-black tree of memory region objects
 */
    struct rb_root mm_rb;

    /* Pointer to the last referenced memory region object */
    struct vm_area_struct * mmap_cache;

    unsigned long (*get_unmapped_area) (struct file *filp,
                unsigned long addr, unsigned long len,
                unsigned long pgoff, unsigned long flags);
    void (*unmap_area) (struct vm_area_struct *area);
    unsigned long mmap_base;
    unsigned long free_area_cache;
    pgd_t * pgd;
    atomic_t mm_users;   /* How many users with user space? */
    atomic_t mm_count;   /* How many references to "struct mm_struct" (users count as 1) */
    int map_count;   /* Number of memory regions */
    struct rw_semaphore mmap_sem;
    spinlock_t page_table_lock;   /* Memory regions’ and Page Tables’ spin lock */

    struct list_head mmlist;   /* Pointers to adjacent elements in the list of memory descriptors */

    unsigned long start_code, end_code, start_data, end_data;   /* code & data */
    unsigned long start_brk, brk, start_stack;   /* heap & stack */
    unsigned long arg_start, arg_end, env_start, env_end;   /* command-line arguments & environment variables */

    /* ... */

    /* Pointer to table for architecture-specific information (e.g. LDT of 80x86) */
    mm_context_t context;

    /* ... */
}
```

All memory descriptors are stored in a **doubly linked list**. Each descriptor stores the address of the adjacent list items in the `mmlist` field. The first element of the list is the `mmlist` field of `init_mm` which is defined in `arch/<platform>/kernel/init_task.c`, the memory descriptor used by **process 0** in the initialization phase.

The `mm_users` field stores the number of lightweight processes that share the `mm_ struct` data structure (either by `clone()`, `fork()`, or `vfork()`). The `mm_count` field is the main usage counter of the memory descriptor; **all "users" in `mm_users` count as one unit in `mm_count`**. Every time the `mm_count` field is decreased, the kernel checks whether it becomes zero; if so, the memory descriptor is deallocated because it is no longer in use.

## Memory Descriptor of Kernel Threads

Kernel threads run only in Kernel Mode, so they never access linear addresses below `TASK_SIZE`. Contrary to regular processes, kernel threads do not use memory regions.

To avoid useless TLB and cache flushes, a kernel thread uses the set of Page Tables of the **last previously running regular process**. To that end, two kinds of memory descriptor pointers are included in every process descriptor: `mm` and `active_mm`.

- `mm`: Points to the memory descriptor **owned** by the process.
- `active_mm`: Points to the memory descriptor used by the process when it is in execution.
  - For regular processes, `active_mm` stores the same pointer as `mm`
  - For kernel threads, `mm` is always `NULL`; `active_mm` points to `active_mm` of the previously running process

# Memory Regions

Linux implements a memory region by means of an object of type `vm_area_struct` which is defined in `include/linux/mm.h`:

```c
struct vm_area_struct {
    struct mm_struct * vm_mm;   /* Pointer to the memory descriptor that owns the region */
    unsigned long vm_start;   /* First linear address inside the region */
    unsigned long vm_end;   /* First linear address after the region */

    struct vm_area_struct *vm_next;   /* Next region in the process list */

    pgprot_t vm_page_prot;   /* Access permissions for the page frames of the region */
    unsigned long vm_flags;   /* VM_READ, VM_WRITE, VM_EXEC, VM_SHARED... */

    struct rb_node vm_rb;   /* Data for the red-black tree */

    /* For reverse mapping */
    union {
        struct {
            struct list_head list;
            void *parent;
            struct vm_area_struct *head;
        } vm_set;

        struct raw_prio_tree_node prio_tree_node;
    } shared;

    struct list_head anon_vma_node;
    struct anon_vma *anon_vma;

    struct vm_operations_struct * vm_ops;   /* Pointer to the methods of the memory region */

    /* Information about our backing store: */
    unsigned long vm_pgoff;   /* Offset in mapped file */
    struct file * vm_file;   /* Pointer to the file object of the mapped file, if any */
    void * vm_private_data;   /* Pointer to private data of the memory region */
    unsigned long vm_truncate_count;   /* Used when releasing a linear address interval in a non-linear file memory mapping */
}
```

Each memory region descriptor identifies a linear address interval; `vm_end-vm_start` denotes the length of the memory region. Memory regions owned by a process **never overlap**, and the kernel tries to merge regions when a new one is allocated right next to an existing one. Two adjacent regions can be merged if their access rights match.

![Adding or removing a linear address interval](http://i.imgur.com/Dsw9FzC.png)

The `vm_ops` field points to a `vm_operations_struct` data structure, which stores the methods of the memory region:

```c
struct vm_operations_struct {
    void (*open)(struct vm_area_struct * area);
    void (*close)(struct vm_area_struct * area);
    struct page * (*nopage)(struct vm_area_struct * area, unsigned long address, int *type);
    int (*populate)(struct vm_area_struct * area, unsigned long address, unsigned long len, pgprot_t prot, unsigned long pgoff, int nonblock);
#ifdef CONFIG_NUMA
    int (*set_policy)(struct vm_area_struct *vma, struct mempolicy *new);
    struct mempolicy *(*get_policy)(struct vm_area_struct *vma,
                    unsigned long addr);
#endif
};
```

- `open()`: Invoked when the memory region is added to the set of regions owned by a process.
- `close()`: Invoked when the memory region is removed from the set of regions owned by a process.
- `nopage()`: Invoked by the Page Fault exception handler when a process tries to access a page not present in RAM whose linear address belongs to the memory region.
- `populate()`: Invoked to set the page table entries corresponding to the linear addresses of the memory region (prefaulting). Mainly used for non-linear file memory mappings.

## Memory Region Data Structures

All the regions owned by a process are linked in a simple list. Regions appear in the list in ascending order by memory address. The `vm_next` field of each `vm_area_struct` element points to the next element in the list. The kernel finds the memory regions through the `mmap` field of the process memory descriptor, which points to the first memory region descriptor in the list.

![Descriptors related to the address space of a process](http://i.imgur.com/pEPvkkx.png)

To make the frequent operations including inserting, deleting and searching be more efficient, Linux 2.6 also stores memory descriptors in another data structures called *red-black trees*. In a red-black tree, each element (or *node*) usually has two children: a *left child* and a *right child*. The elements in the tree are sorted. For each node N, all elements of the subtree rooted at the left child of N precede N, while, conversely, all elements of the subtree rooted at the right child of N follow N; the key of the node is written inside the node itself. Moreover, a red-black tree must satisfy four additional rules:

1. Every node must be either red or black.
2. The root of the tree must be black.
3. The children of a red node must be black.
4. Every path from a node to a descendant leaf must contain the same number of black nodes. When counting the number of black nodes, null pointers are counted as black nodes.

These four rules ensure that every red-black tree with n internal nodes has a height of at most `2 × log(n + 1)`.

![Example of red-black trees](http://i.imgur.com/xaaLWyY.png)

The head of the red-black tree is referenced by the `mm_rb` field of the memory descriptor. Each memory region object stores the color of the node, as well as the pointers to the parent, the left child, and the right child, in the `vm_rb` field of type `rb_node` defined in `include/linux/rbtree.h`:

```c
struct rb_node
{
    struct rb_node *rb_parent;
    int rb_color;
#define RB_RED 0
#define RB_BLACK 1
    struct rb_node *rb_right;
    struct rb_node *rb_left;
};

struct rb_root
{
    struct rb_node *rb_node;
};
```

## Memory Region Handling

The following functions should be considered auxiliary functions that simplify the implementation of `do_mmap()` and `do_munmap()`.

### Finding the closest region to a given address: `find_vma()`

The `find_vma()` in `mm/mmap.c` locates the first memory region whose **`vm_end` field is greater than `addr`** and returns the address of its descriptor; if no such region exists, it returns a `NULL` pointer. Notice that the region selected by `find_vma()` does not necessarily include `addr` because `addr` may lie outside of any memory region.

Each memory descriptor includes an `mmap_cache` field that stores the descriptor address of the region that was last referenced by the process. This additional field is introduced to reduce the time spent in looking for the region that contains a given linear address.

```c
struct vm_area_struct * find_vma(struct mm_struct * mm, unsigned long addr)
{
    struct vm_area_struct *vma = NULL;

    if (mm) {
        vma = mm->mmap_cache;   /* Check the cache first */
        if (!(vma && vma->vm_end > addr && vma->vm_start <= addr)) {
            struct rb_node * rb_node;

            rb_node = mm->mm_rb.rb_node;
            vma = NULL;

            while (rb_node) {
                struct vm_area_struct * vma_tmp;

                vma_tmp = rb_entry(rb_node,
                        struct vm_area_struct, vm_rb);

                if (vma_tmp->vm_end > addr) {
                    vma = vma_tmp;
                    if (vma_tmp->vm_start <= addr)
                        break;
                    rb_node = rb_node->rb_left;
                } else
                    rb_node = rb_node->rb_right;
            }
            if (vma)
                mm->mmap_cache = vma;
        }
    }
    return vma;
}
```

The function uses the `rb_entry` macro, which derives from a pointer to a node of the red-black tree the address of the corresponding memory region descriptor.

```c
/* in rbtree.h */
#define rb_entry(ptr, type, member) container_of(ptr, type, member)

/* in kernel.h */
#define container_of(ptr, type, member) ({          \
        const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
        (type *)( (char *)__mptr - offsetof(type,member) );})
```

The `find_vma_prev()` function is similar to `find_vma()`, except that it writes in an additional `pprev` parameter a pointer to the descriptor of the memory region that precedes the one selected by the function.

```c
struct vm_area_struct *
find_vma_prev(struct mm_struct *mm, unsigned long addr,
            struct vm_area_struct **pprev)
{
    struct vm_area_struct *vma = NULL, *prev = NULL;
    struct rb_node * rb_node;
    if (!mm)
        goto out;

    /* Guard against addr being lower than the first VMA */
    vma = mm->mmap;

    /* Go through the RB tree quickly. */
    rb_node = mm->mm_rb.rb_node;

    while (rb_node) {
        struct vm_area_struct *vma_tmp;
        vma_tmp = rb_entry(rb_node, struct vm_area_struct, vm_rb);

        if (addr < vma_tmp->vm_end) {
            rb_node = rb_node->rb_left;
        } else {
            prev = vma_tmp;
            if (!prev->vm_next || (addr < prev->vm_next->vm_end))
                break;
            rb_node = rb_node->rb_right;
        }
    }

out:
    *pprev = prev;
    return prev ? prev->vm_next : vma;
}
```

Finally, the `find_vma_prepare()` function locates the position of the new leaf in the red-black tree that corresponds to a given linear address and returns the addresses of the preceding memory region and of the parent node of the leaf to be inserted.

```c
static struct vm_area_struct *
find_vma_prepare(struct mm_struct *mm, unsigned long addr,
        struct vm_area_struct **pprev, struct rb_node ***rb_link,
        struct rb_node ** rb_parent)
{
    struct vm_area_struct * vma;
    struct rb_node ** __rb_link, * __rb_parent, * rb_prev;

    __rb_link = &mm->mm_rb.rb_node;
    rb_prev = __rb_parent = NULL;
    vma = NULL;

    while (*__rb_link) {
        struct vm_area_struct *vma_tmp;

        __rb_parent = *__rb_link;
        vma_tmp = rb_entry(__rb_parent, struct vm_area_struct, vm_rb);

        if (vma_tmp->vm_end > addr) {
            vma = vma_tmp;
            if (vma_tmp->vm_start <= addr)
                return vma;
            __rb_link = &__rb_parent->rb_left;
        } else {
            rb_prev = __rb_parent;
            __rb_link = &__rb_parent->rb_right;
        }
    }

    *pprev = NULL;
    if (rb_prev)
        *pprev = rb_entry(rb_prev, struct vm_area_struct, vm_rb);
    *rb_link = __rb_link;
    *rb_parent = __rb_parent;
    return vma;
}
```

### Finding a region that overlaps a given interval: `find_vma_intersection()`

The `find_vma_intersection()` function finds the first memory region that overlaps a given linear address interval.

```c
static inline struct vm_area_struct * find_vma_intersection(struct mm_struct * mm, unsigned long start_addr, unsigned long end_addr)
{
    struct vm_area_struct * vma = find_vma(mm,start_addr);

    if (vma && end_addr <= vma->vm_start)   /* check interval */
        vma = NULL;
    return vma;
}
```

### Finding a free interval: `get_unmapped_area()`

The `get_unmapped_area()` function searches the process address space to find an available linear address interval.

```c
unsigned long
get_unmapped_area(struct file *file, unsigned long addr, unsigned long len,
        unsigned long pgoff, unsigned long flags)
{
    if (flags & MAP_FIXED) {
        unsigned long ret;

        if (addr > TASK_SIZE - len)
            return -ENOMEM;
        if (addr & ~PAGE_MASK)
            return -EINVAL;
        if (file && is_file_hugepages(file))  {
            /*
             * Check if the given range is hugepage aligned, and
             * can be made suitable for hugepages.
             */
            ret = prepare_hugepage_range(addr, len);
        } else {
            /*
             * Ensure that a normal request is not falling in a
             * reserved hugepage range.  For some archs like IA-64,
             * there is a separate region for hugepages.
             */
            ret = is_hugepage_only_range(addr, len);
        }
        if (ret)
            return -EINVAL;
        return addr;
    }

    if (file && file->f_op && file->f_op->get_unmapped_area)
        return file->f_op->get_unmapped_area(file, addr, len,
                        pgoff, flags);

    return current->mm->get_unmapped_area(file, addr, len, pgoff, flags);
}
```

If the `addr` parameter is not NULL, the function checks that the specified address is in the User Mode address space and that it is aligned to a page boundary. Next, the function invokes either one of two methods, depending on whether the linear address interval should be used for a **file memory mapping** or for an **anonymous memory mapping**.

In the latter case, the function executes the `get_unmapped_area` method of the memory descriptor. In turn, this method is implemented by either the `arch_get_unmapped_area()` function, or the `arch_get_unmapped_area_topdown()` function, according to the memory region layout of the process.

```c
unsigned long
arch_get_unmapped_area(struct file *filp, unsigned long addr,
        unsigned long len, unsigned long pgoff, unsigned long flags)
{
    struct mm_struct *mm = current->mm;
    struct vm_area_struct *vma;
    unsigned long start_addr;
    unsigned long begin, end;

    find_start_end(flags, &begin, &end);

    if (len > end)
        return -ENOMEM;

    if (addr) {
        addr = PAGE_ALIGN(addr);
        vma = find_vma(mm, addr);
        if (end - len >= addr &&
            (!vma || addr + len <= vma->vm_start))
            return addr;
    }
    addr = mm->free_area_cache;
    if (addr < begin)
        addr = begin;
    start_addr = addr;

full_search:
    for (vma = find_vma(mm, addr); ; vma = vma->vm_next) {
        /* At this point:  (!vma || addr < vma->vm_end). */
        if (end - len < addr) {
            /*
             * Start a new search - just in case we missed
             * some holes.
             */
            if (start_addr != begin) {
                start_addr = addr = begin;
                goto full_search;
            }
            return -ENOMEM;
        }
        if (!vma || addr + len <= vma->vm_start) {
            /*
             * Remember the place where we stopped the search:
             */
            mm->free_area_cache = addr + len;
            return addr;
        }
        addr = vma->vm_end;
    }
}
```

### Inserting a region in the memory descriptor list: `insert_vm_struct()`

`insert_vm_struct()` inserts a `vm_area_struct` structure in the memory region object list and red-black tree of a memory descriptor.

```c
int insert_vm_struct(struct mm_struct * mm, struct vm_area_struct * vma)
{
    struct vm_area_struct * __vma, * prev;
    struct rb_node ** rb_link, * rb_parent;

    if (!vma->vm_file) {
        BUG_ON(vma->anon_vma);
        vma->vm_pgoff = vma->vm_start >> PAGE_SHIFT;
    }
    __vma = find_vma_prepare(mm,vma->vm_start,&prev,&rb_link,&rb_parent);
    if (__vma && __vma->vm_start < vma->vm_end)
        return -ENOMEM;
    vma_link(mm, vma, prev, rb_link, rb_parent);
    return 0;
}
```

The function invokes the `find_vma_prepare()` function to look up the position in the red-black tree `mm->mm_rb` where vma should go. Then `insert_vm_struct()` invokes the `vma_link()` function, which in turn:

1. Inserts the memory region in the linked list referenced by `mm->mmap`.
2. Inserts the memory region in the red-black tree `mm->mm_rb`.
3. If the memory region is anonymous, inserts the region in the list headed at the corresponding `anon_vma` data structure.
4. Increases the `mm->map_count` counter.

## Allocating a Linear Address Interval

To allocate new linear address intervals, the `do_mmap()` function creates and initializes a new memory region for the current process. However, after a successful allocation, the memory region could be merged with other memory regions defined for the process.

```c
static inline unsigned long do_mmap(struct file *file, unsigned long addr,
    unsigned long len, unsigned long prot,
    unsigned long flag, unsigned long offset)
{
    unsigned long ret = -EINVAL;
    if ((offset + PAGE_ALIGN(len)) < offset)
        goto out;
    if (!(offset & ~PAGE_MASK))
        ret = do_mmap_pgoff(file, addr, len, prot, flag, offset >> PAGE_SHIFT);
out:
    return ret;
}
```

The `do_mmap()` function performs some preliminary checks on the value of offset and then executes the `do_mmap_pgoff()` function. Notice that at this time we will suppose that the new interval of linear address does not map a file on disk.

```c
unsigned long do_mmap_pgoff(struct file * file, unsigned long addr,
            unsigned long len, unsigned long prot,
            unsigned long flags, unsigned long pgoff)
{
    /* ... */

    /* Check overflow? */
    len = PAGE_ALIGN(len);
    if (!len || len > TASK_SIZE)
        return -EINVAL;

    /* Check offset overflow? */
    if ((pgoff + (len >> PAGE_SHIFT)) < pgoff)
        return -EINVAL;

    /* Too many mappings? */
    if (mm->map_count > sysctl_max_map_count)
        return -ENOMEM;

    /* Obtain a linear address interval for the new region */
    addr = get_unmapped_area(file, addr, len, pgoff, flags);
    if (addr & ~PAGE_MASK)
        return addr;

    /* ... */

munmap_back:
    /* Locate the object of the memory region that shall precede the new interval, as well as the position of the new region in the red-black tree */
    vma = find_vma_prepare(mm, addr, &prev, &rb_link, &rb_parent);
    if (vma && vma->vm_start < addr + len) {
        if (do_munmap(mm, addr, len))
            return -ENOMEM;
        goto munmap_back;
    }

    /* Check against address space limit */
    if ((mm->total_vm << PAGE_SHIFT) + len
        > current->signal->rlim[RLIMIT_AS].rlim_cur)
        return -ENOMEM;

    /* ... */

    /*
     * Can we just expand an old private anonymous mapping?
     * The VM_SHARED test is necessary because shmem_zero_setup
     * will create the file object for a shared anonymous map below.
     */
    if (!file && !(vm_flags & VM_SHARED) &&
        vma_merge(mm, prev, addr, addr + len, vm_flags,
                    NULL, NULL, pgoff, NULL))
        goto out;

    /*
     * Determine the object being mapped and call the appropriate
     * specific mapper. the address has already been validated, but
     * not unmapped, but the maps are removed from the list.
     */
    vma = kmem_cache_alloc(vm_area_cachep, SLAB_KERNEL);
    if (!vma) {
        error = -ENOMEM;
        goto unacct_error;
    }
    memset(vma, 0, sizeof(*vma));

    vma->vm_mm = mm;
    vma->vm_start = addr;
    vma->vm_end = addr + len;
    vma->vm_flags = vm_flags;
    vma->vm_page_prot = protection_map[vm_flags & 0x0f];
    vma->vm_pgoff = pgoff;

    /* ... */

    if (!file || !vma_merge(mm, prev, addr, vma->vm_end,
            vma->vm_flags, NULL, file, pgoff, vma_policy(vma))) {
        file = vma->vm_file;
        vma_link(mm, vma, prev, rb_link, rb_parent);   /* Insert the new region in the memory region list and red-black tree */
        if (correct_wcount)
            atomic_inc(&inode->i_writecount);
    } else {
        if (file) {
            if (correct_wcount)
                atomic_inc(&inode->i_writecount);
            fput(file);
        }
        mpol_free(vma_policy(vma));
        kmem_cache_free(vm_area_cachep, vma);
    }
out:
    mm->total_vm += len >> PAGE_SHIFT;

    /* ... */
}
```

## Releasing a Linear Address Interval

When the kernel must delete a linear address interval from the address space of the current process, it uses the `do_munmap()` function.

### The `do_munmap()` function

The function goes through two main phases:

- Phase 1: Scans the list of memory regions owned by the process and unlinks all regions included in the linear address interval from the process address space
- Phase 2: Updates the process Page Tables and removes the memory regions identified in the first phase

```c
int do_munmap(struct mm_struct *mm, unsigned long start, size_t len)
{
    /*** Phase 1 ***/
    unsigned long end;
    struct vm_area_struct *mpnt, *prev, *last;

    if ((start & ~PAGE_MASK) || start > TASK_SIZE || len > TASK_SIZE-start)
        return -EINVAL;

    if ((len = PAGE_ALIGN(len)) == 0)
        return -EINVAL;

    /* Locates the first memory region mpnt that ends after the linear address interval to be deleted */
    mpnt = find_vma_prev(mm, start, &prev);
    if (!mpnt)
        return 0;
    /* we have start < mpnt->vm_end  */

    /* No overlap, no such memory region */
    end = start + len;
    if (mpnt->vm_start >= end)
        return 0;

    /* Check if the linear address interval starts inside the mpnt memory region */
    if (start > mpnt->vm_start) {
        int error = split_vma(mm, mpnt, start, 0);
        if (error)
            return error;
        prev = mpnt;
    }

    /* Check if the linear address interval ends inside a memory region */
    last = find_vma(mm, end);
    if (last && end > last->vm_start) {
        int error = split_vma(mm, last, end, 1);
        if (error)
            return error;
    }
    mpnt = prev? prev->vm_next: mm->mmap;

    /*** Phase 2 ***/
    /* Remove the memory regions included in the linear address interval from the process’s linear address space */
    detach_vmas_to_be_unmapped(mm, mpnt, prev, end);
    spin_lock(&mm->page_table_lock);
    unmap_region(mm, mpnt, prev, start, end);
    spin_unlock(&mm->page_table_lock);

    /* Fix up all other VM information */
    unmap_vma_list(mm, mpnt);

    return 0;
}

static void
detach_vmas_to_be_unmapped(struct mm_struct *mm, struct vm_area_struct *vma,
    struct vm_area_struct *prev, unsigned long end)
{
    struct vm_area_struct **insertion_point;
    struct vm_area_struct *tail_vma = NULL;

    insertion_point = (prev ? &prev->vm_next : &mm->mmap);
    do {
        rb_erase(&vma->vm_rb, &mm->mm_rb);
        mm->map_count--;
        tail_vma = vma;
        vma = vma->vm_next;
    } while (vma && vma->vm_start < end);
    *insertion_point = vma;
    tail_vma->vm_next = NULL;
    mm->mmap_cache = NULL;      /* Kill the cache. */
}
```

### The `split_vma()` function

The purpose of the `split_vma()` function is to split a memory region that intersects a linear address interval into two smaller regions, one outside of the interval and the other inside.

```c
int split_vma(struct mm_struct * mm, struct vm_area_struct * vma,
          unsigned long addr, int new_below)
{
    struct mempolicy *pol;
    struct vm_area_struct *new;

    if (is_vm_hugetlb_page(vma) && (addr & ~HPAGE_MASK))
        return -EINVAL;

    if (mm->map_count >= sysctl_max_map_count)
        return -ENOMEM;

    new = kmem_cache_alloc(vm_area_cachep, SLAB_KERNEL);
    if (!new)
        return -ENOMEM;

    /* most fields are the same, copy all, and then fixup */
    *new = *vma;

    if (new_below)   /* the linear address interval ends inside the vma region */
        new->vm_end = addr;
    else {   /* the linear address interval starts inside the vma region */
        new->vm_start = addr;
        new->vm_pgoff += ((addr - vma->vm_start) >> PAGE_SHIFT);
    }

    pol = mpol_copy(vma_policy(vma));
    if (IS_ERR(pol)) {
        kmem_cache_free(vm_area_cachep, new);
        return PTR_ERR(pol);
    }
    vma_set_policy(new, pol);

    if (new->vm_file)
        get_file(new->vm_file);

    if (new->vm_ops && new->vm_ops->open)
        new->vm_ops->open(new);

    if (new_below)
        vma_adjust(vma, addr, vma->vm_end, vma->vm_pgoff +
            ((addr - new->vm_start) >> PAGE_SHIFT), new);
    else
        vma_adjust(vma, vma->vm_start, addr, vma->vm_pgoff, new);

    return 0;
}
```

### The `unmap_region()` function

The `unmap_region()` function walks through a list of memory regions and releases the page frames belonging to them.

```c
static void unmap_region(struct mm_struct *mm,
    struct vm_area_struct *vma,
    struct vm_area_struct *prev,
    unsigned long start,
    unsigned long end)
{
    struct mmu_gather *tlb;
    unsigned long nr_accounted = 0;

    lru_add_drain();
    tlb = tlb_gather_mmu(mm, 0);
    unmap_vmas(&tlb, mm, vma, start, end, &nr_accounted, NULL); /* scan all Page Table entries belonging to the linear address interval */
    vm_unacct_memory(nr_accounted);

    if (is_hugepage_only_range(start, end - start))
        hugetlb_free_pgtables(tlb, prev, start, end);
    else
        free_pgtables(tlb, prev, start, end);   /* try to reclaim the Page Tables of the process */
    tlb_finish_mmu(tlb, start, end);   /* flush the TLB */
}
```

# Page Fault Exception Handler

The `do_page_fault()` function, which is the Page Fault interrupt service routine for the 80x86 architecture, compares the linear address that caused the Page Fault against the memory regions of the current process; it can thus determine the proper way to handle the exception according to the scheme  illustrated below:

![Overall scheme for the Page Fault handler](http://i.imgur.com/oQTwtDN.png)

![The flow diagram of the Page Fault handler](http://i.imgur.com/0ToEnBl.png)

The `do_page_fault()` function accepts the following input parameters:

- The `regs` address of a `pt_regs` structure containing the values of the microprocessor registers when the exception occurred.
- A 3-bit `error_code`, which is pushed on the stack by the control unit when the exception occurred:
  - bit 0
    - clear: caused by an access to a page that is not present
    - set: caused by an invalid access right
  - bit 1
    - clear: caused by a **read or execute** access
    - set: caused by a **write** access
  - bit 2
    - clear: occurred in **Kernel Mode**
    - set: occurred in **User Mode**

```c
asmlinkage void do_page_fault(struct pt_regs *regs, unsigned long error_code)
{
    struct task_struct *tsk;
    struct mm_struct *mm;
    struct vm_area_struct * vma;
    unsigned long address;
    const struct exception_table_entry *fixup;
    int write;
    siginfo_t info;

    /* Get the linear address from cr2 control register */
    __asm__("movq %%cr2,%0":"=r" (address));
    if (notify_die(DIE_PAGE_FAULT, "page fault", regs, error_code, 14,
                    SIGSEGV) == NOTIFY_STOP)
        return;

    if (likely(regs->eflags & X86_EFLAGS_IF))
        local_irq_enable();

    if (unlikely(page_fault_trace))
        printk("pagefault rip:%lx rsp:%lx cs:%lu ss:%lu address %lx error %lx\n",
               regs->rip,regs->rsp,regs->cs,regs->ss,address,error_code);

    tsk = current;
    mm = tsk->mm;
    info.si_code = SEGV_MAPERR;


    /* Check whether the faulty linear address belongs to the 4GB address space */
    if (unlikely(address >= TASK_SIZE)) {
        if (!(error_code & 5)) {
            if (vmalloc_fault(address) < 0)   /* accessing a noncontiguous memory area in Kernel Mode */
                goto bad_area_nosemaphore;
            return;
        }
        goto bad_area_nosemaphore;
    }

    if (unlikely(error_code & (1 << 3)))
        pgtable_bad(address, regs, error_code);

    /* Check whether the exception occurred while the kernel was executing some critical routine (e.g. interrupt handler) or running a kernel thread */
    if (unlikely(in_atomic() || !mm))
        goto bad_area_nosemaphore;

 again:
    /* Try to acquire semaphore */
    if (!down_read_trylock(&mm->mmap_sem)) {
        if ((error_code & 4) == 0 &&
            !search_exception_tables(regs->rip))
            goto bad_area_nosemaphore;
        down_read(&mm->mmap_sem);
    }

    /* Look for a memory region containing the faulty linear address */
    vma = find_vma(mm, address);
    if (!vma)
        goto bad_area;
    if (likely(vma->vm_start <= address))
        goto good_area;
    if (!(vma->vm_flags & VM_GROWSDOWN))   /* User Mode stack */
        goto bad_area;
    if (error_code & 4) {
        if (address + 128 < regs->rsp)   /* check with stack pointer */
            goto bad_area;
    }
    if (expand_stack(vma, address))
        goto bad_area;

    /* ... */
}
```

## Handling a Faulty Address Outside the Address Space

If address does not belong to the process address space, `do_page_fault()` proceeds to execute the statements at the label `bad_area`. If the error occurred in User Mode, it sends a `SIGSEGV` signal to *current* and terminates:

```c
bad_area:
    up_read(&mm->mmap_sem);

bad_area_nosemaphore:
    if (error_code & 4) {   /* User Mode */
        if (is_prefetch(regs, address, error_code))
            return;

        /* ... */

        tsk->thread.cr2 = address;
        /* Kernel addresses are always protection faults */
        tsk->thread.error_code = error_code | (address >= TASK_SIZE);
        tsk->thread.trap_no = 14;
        info.si_signo = SIGSEGV;
        info.si_errno = 0;
        info.si_addr = (void __user *)address;
        force_sig_info(SIGSEGV, &info, tsk);
        return;
    }
```

If the exception occurred in Kernel Mode (bit 2 of `error_code` is clear), there are still two alternatives:

- The exception occurred while using some linear address that has been passed to the kernel as a parameter of a system call.
- The exception is due to a real kernel bug.

```c
no_context:
    fixup = search_exception_tables(regs->rip);
    if (fixup) {
        regs->rip = fixup->fixup;
        return;
    }

    /* Hall of shame of CPU/BIOS bugs */
    if (is_prefetch(regs, address, error_code))
        return;

    if (is_errata93(regs, address))
        return;

    /* Kernel oops */
    oops_begin();

    if (address < PAGE_SIZE)
        printk(KERN_ALERT "Unable to handle kernel NULL pointer dereference");
    else
        printk(KERN_ALERT "Unable to handle kernel paging request");
    printk(" at %016lx RIP: \n" KERN_ALERT,address);
    printk_address(regs->rip);
    printk("\n");
    dump_pagetable(address);
    __die("Oops", regs, error_code);
    /* Executive summary in case the body of the oops scrolled away */
    printk(KERN_EMERG "CR2: %016lx\n", address);
    oops_end();
    do_exit(SIGKILL);   /* kill current process */
```

## Handling a Faulty Address Inside the Address Space

If address belongs to the process address space, `do_page_fault()` proceeds to the statement labeled `good_area`:

```c
good_area:
    info.si_code = SEGV_ACCERR;
    write = 0;
    switch (error_code & 3) {
        default:   /* 3: write, present */
            /* fall through */
        case 2:   /* write, not present */
            if (!(vma->vm_flags & VM_WRITE))
                goto bad_area;
            write++;
            break;
        case 1:   /* read, present (maybe tried to access a privileged page frame) */
            goto bad_area;
        case 0:   /* read, not present */
            if (!(vma->vm_flags & (VM_READ | VM_EXEC)))   /* check whether memory region is readable or executable */
                goto bad_area;
    }

    /* Memory region access rights match the access type, handle it */
    switch (handle_mm_fault(mm, vma, address, write)) {
    case 1:
        tsk->min_flt++;
        break;
    case 2:
        tsk->maj_flt++;
        break;
    case 0:
        goto do_sigbus;   /* send SIGBUS signal */
    default:
        goto out_of_memory;   /* OOM, kill current process */
    }

    up_read(&mm->mmap_sem);
    return;
```

The `handle_mm_fault` function in `mm/memory.c` starts by checking whether the Page Middle Directory and the Page Table used to map address exist. Even if address belongs to the process address space, the corresponding Page Tables **might not have been allocated**, so the task of allocating them precedes everything else:

```c
int handle_mm_fault(struct mm_struct *mm, struct vm_area_struct * vma,
        unsigned long address, int write_access)
{
    pgd_t *pgd;
    pud_t *pud;
    pmd_t *pmd;
    pte_t *pte;

    __set_current_state(TASK_RUNNING);

    inc_page_state(pgfault);

    if (is_vm_hugetlb_page(vma))
        return VM_FAULT_SIGBUS;

    pgd = pgd_offset(mm, address);
    spin_lock(&mm->page_table_lock);

    pud = pud_alloc(mm, pgd, address);
    if (!pud)
        goto oom;

    pmd = pmd_alloc(mm, pud, address);
    if (!pmd)
        goto oom;

    pte = pte_alloc_map(mm, pmd, address);
    if (!pte)
        goto oom;

    return handle_pte_fault(mm, vma, address, write_access, pte, pmd);

 oom:
    spin_unlock(&mm->page_table_lock);
    return VM_FAULT_OOM;
}
```

The `handle_pte_fault()` function is then invoked to inspect the Page Table entry corresponding to address and to determine how to allocate a new page frame for the process:

```c
static inline int handle_pte_fault(struct mm_struct *mm,
    struct vm_area_struct * vma, unsigned long address,
    int write_access, pte_t *pte, pmd_t *pmd)
{
    pte_t entry;

    entry = *pte;
    if (!pte_present(entry)) {   /* not already stored in any page frame */
        if (pte_none(entry))   /* e.g. due to Damand Paging */
            return do_no_page(mm, vma, address, write_access, pte, pmd);
        if (pte_file(entry))
            return do_file_page(mm, vma, address, write_access, pte, pmd);
        return do_swap_page(mm, vma, address, pte, pmd, entry, write_access);
    }

    if (write_access) {
        if (!pte_write(entry))   /* marked read-only, Copy On Write */
            return do_wp_page(mm, vma, address, pte, pmd, entry);

        entry = pte_mkdirty(entry);
    }
    entry = pte_mkyoung(entry);
    ptep_set_access_flags(vma, address, pte, entry, write_access);
    update_mmu_cache(vma, address, entry);
    pte_unmap(pte);
    spin_unlock(&mm->page_table_lock);
    return VM_FAULT_MINOR;
}
```

## Demand Paging

The term *demand paging* denotes a dynamic memory allocation technique that consists of deferring page frame allocation until the last possible moment — until the process attempts to address a page that is not present in RAM, thus causing a Page Fault exception.

## Copy On Write

The idea of *Copy On Write (COW)* is quite simple: instead of duplicating page frames, they are shared between the parent and the child process. However, as long as they are shared, they cannot be modified. Whenever the parent or the child process attempts to write into a shared page frame, an exception occurs. At this point, the kernel duplicates the page into a new page frame that it marks as writable.

The core part of Copy On Write is implemented in `do_wp_page()` function:

```c
static int do_wp_page(struct mm_struct *mm, struct vm_area_struct * vma,
    unsigned long address, pte_t *page_table, pmd_t *pmd, pte_t pte)
{
    struct page *old_page, *new_page;
    unsigned long pfn = pte_pfn(pte);
    pte_t entry;

    /* ... */

    old_page = pfn_to_page(pfn);

    /* ... */

    if (unlikely(anon_vma_prepare(vma)))
        goto no_new_page;
    if (old_page == ZERO_PAGE(address)) {
        new_page = alloc_zeroed_user_highpage(vma, address);
        if (!new_page)
            goto no_new_page;
    } else {
        new_page = alloc_page_vma(GFP_HIGHUSER, vma, address);
        if (!new_page)
            goto no_new_page;
        copy_user_highpage(new_page, old_page, address);
    }

    /* ... */
}
```

# Creating and Deleting a Process Address Space

## Creating a Process Address Space

The kernel will invoke the `copy_mm()` function in `kernel/fork.c` while creating a new process. This function creates the process address space by setting up all Page Tables and memory descriptors of the new process.

```c
static int copy_mm(unsigned long clone_flags, struct task_struct * tsk)
{
    struct mm_struct * mm, *oldmm;
    int retval;

    tsk->min_flt = tsk->maj_flt = 0;
    tsk->nvcsw = tsk->nivcsw = 0;

    tsk->mm = NULL;
    tsk->active_mm = NULL;

    oldmm = current->mm;
    if (!oldmm)   /* kernel thread? */
        return 0;

    if (clone_flags & CLONE_VM) {
        atomic_inc(&oldmm->mm_users);
        mm = oldmm;   /* share address space */
        spin_unlock_wait(&oldmm->page_table_lock);
        goto good_mm;
    }

    retval = -ENOMEM;
    mm = allocate_mm();   /* create a new address space */
    if (!mm)
        goto fail_nomem;

    memcpy(mm, oldmm, sizeof(*mm));
    if (!mm_init(mm))
        goto fail_nomem;

    /* Make a copy of the Local Descriptor Table of current and adds it to the address space of tsk */
    if (init_new_context(tsk,mm))
        goto fail_nocontext;

    retval = dup_mmap(mm, oldmm);
    if (retval)
        goto free_pt;

    mm->hiwater_rss = mm->rss;
    mm->hiwater_vm = mm->total_vm;

good_mm:
    tsk->mm = mm;
    tsk->active_mm = mm;
    return 0;

free_pt:
    mmput(mm);
fail_nomem:
    return retval;

fail_nocontext:
    mm_free_pgd(mm);
    free_mm(mm);
    return retval;
}
```

## Deleting a Process Address Space

When a process terminates, the kernel invokes the `exit_mm()` function to release the address space owned by that process:

```c
void exit_mm(struct task_struct * tsk)
{
    struct mm_struct *mm = tsk->mm;

    mm_release(tsk, mm);   /* wakes up all processes sleeping in the tsk-> vfork_done completion */
    if (!mm)
        return;
    down_read(&mm->mmap_sem);
    if (mm->core_waiters) {
        up_read(&mm->mmap_sem);
        down_write(&mm->mmap_sem);
        if (!--mm->core_waiters)
            complete(mm->core_startup_done);
        up_write(&mm->mmap_sem);

        wait_for_completion(&mm->core_done);
        down_read(&mm->mmap_sem);
    }
    atomic_inc(&mm->mm_count);   /* increases main usage counter */
    if (mm != tsk->active_mm) BUG();

    task_lock(tsk);
    tsk->mm = NULL;
    up_read(&mm->mmap_sem);
    enter_lazy_tlb(mm, current);
    task_unlock(tsk);
    mmput(mm);   /* release the Local Descriptor Table, the memory region descriptors, and the Page Tables */
}
```

## Managing the Heap

Each Unix process owns a specific memory region called the *heap*, which is used to satisfy the process’s dynamic memory requests. The `start_brk` and `brk` fields of the memory descriptor delimit the starting and ending addresses, respectively, of that region.

The following APIs can be used by the process to request and release dynamic memory:

- `malloc(size)`: Requests size bytes of dynamic memory; if the allocation succeeds, it returns the linear address of the first memory location.
- `calloc(n, size)`: Requests an array consisting of n elements of size size; if the allocation suc- ceeds, it initializes the array components to 0 and returns the linear address of the first element.
- `realloc(ptr, size)`: Changes the size of a memory area previously allocated by `malloc()` or `calloc()`.
- `free(addr)`: Releases the memory region allocated by `malloc()` or `calloc()` that has an initial address of addr.
- `brk(addr)`: Modifies the size of the heap directly; the addr parameter specifies the new value of `current->mm->brk`, and the return value is the new ending address of the memory region.
- `sbrk(incr)`: Is similar to `brk()`, except that the `incr` parameter specifies the increment or decrement of the heap size in bytes.

Notice that `brk()` function is implemented as a system call. The implementation can be found in `sys_brk()` function in `mm/mmap.c`:

```c
asmlinkage unsigned long sys_brk(unsigned long brk)
{
    unsigned long rlim, retval;
    unsigned long newbrk, oldbrk;
    struct mm_struct *mm = current->mm;

    down_write(&mm->mmap_sem);

    /* Check whether the addr parameter falls inside the memory region that contains the process code */
    if (brk < mm->end_code)
        goto out;
    newbrk = PAGE_ALIGN(brk);   /* align to a multiple of PAGE_SIZE */
    oldbrk = PAGE_ALIGN(mm->brk);
    if (oldbrk == newbrk)
        goto set_brk;

    /* Allow to shrink the heap */
    if (brk <= mm->brk) {
        if (!do_munmap(mm, newbrk, oldbrk-newbrk))
            goto set_brk;
        goto out;
    }

    /* Check against rlimit */
    rlim = current->signal->rlim[RLIMIT_DATA].rlim_cur;
    if (rlim < RLIM_INFINITY && brk - mm->start_data > rlim)
        goto out;

    /* Check against existing mmap mappings */
    if (find_vma_intersection(mm, oldbrk, newbrk+PAGE_SIZE))
        goto out;

    /* Everything is OK */
    if (do_brk(oldbrk, newbrk-oldbrk) != oldbrk)
        goto out;
set_brk:
    mm->brk = brk;
out:
    retval = mm->brk;
    up_write(&mm->mmap_sem);
    return retval;
}
```
