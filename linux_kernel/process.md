Process
=======

# Lightweight process

Linux uses *lightweight processes (LWP)* to implement multi-threading support. Each entities that can be scheduled are allocated a process descriptor `task_struct`, which is defined in `include/linux/sched.h`.

```c
struct task_struct {
    volatile long state;  /* -1 unrunnable, 0 runnable, >0 stopped */
    struct thread_info *thread_info;  /* low-level information for the process */

  /* ... */

    struct list_head run_list;

  /* ... */

    struct list_head tasks;
    struct list_head ptrace_children;
    struct list_head ptrace_list;

    struct mm_struct *mm, *active_mm;

  /* ... */

    struct task_struct *real_parent;
    struct task_struct *parent;
    struct list_head children;
    struct list_head sibling;
    struct task_struct *group_leader;

    struct pid pids[PIDTYPE_MAX];

  /* ... */

    struct fs_struct *fs; /* current directory */
    struct files_struct *files; /* pointers to file descriptors */

  /* ... */

    struct signal_struct *signal; /* signal received */
    struct sighand_struct *sighand;

  /* ... */
};
```

Linux associates a different PID with each process or lightweight process in the system. However, POSIX standard states that all threads of a multithreaded application must have the same PID.

To comply with this standard, the identifier shared by the threads is the PID of the *thread group leader*, that is, the PID of the first lightweight process in the group; it is stored in the `tgid` field of the process descriptors.

The `getpid()` system call returns the value of `tgid` relative to the current process instead of the value of `pid`, so all the threads of a multithreaded application share the same identifier.

![Storing the thread_info structure and the process kernel stack in two page frames](http://i.imgur.com/U3FJkMA.png)

# Doubly linked list

## Circular list: `list_head`

Linux kernel defines the `list_head` data structure, whose only fields `next` and `prev` represent the forward and back pointers of a generic doubly linked list element, respectively. It is important to note, however, that the pointers in a `list_head` field store the addresses of other `list_head` fields rather than the addresses of the whole data structures in which the `list_head` structure is included.

```c
struct list_head {
    struct list_head *next, *prev;
};
```

![Doubly linked lists built with list_head data structures](http://i.imgur.com/AhPK8qG.png)

A new list is created by using the `LIST_HEAD(list_name)` macro defined in `include/linux/list.h`.

```c
#define LIST_HEAD_INIT(name) { &(name), &(name) }

#define LIST_HEAD(name) \
    struct list_head name = LIST_HEAD_INIT(name)
```

There are some list handling functions supported, including `list_add(n, p)`, `list_add_tail(n, p`, `list_del(p)`, `list_for_each_entry(p, h, m)`, etc.

The macro `list_for_each_entry(p, h, m)` actually uses another macro `list_entry`, and finally the macro `container_of` defined in `include/linux/kernel.h` to cast a member of a structure out to the containing structure.

```c
#define container_of(ptr, type, member) ({          \
        const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
        (type *)( (char *)__mptr - offsetof(type,member) );})
```

## Non-circular list: `hlist_head`

Linux kernel also provides another kind of doubly linked list, which mainly differs from a `list_head` list because it is **not circular**.

It is mainly used for **hash tables**, where space is important, and finding the the last element in constant time is not. The list head is stored in an `hlist_head` data structure, which is simply a pointer to the first element in the list (`NULL` if the list is empty).

An example of a doubly linked list is the *process list*. There are some useful macros:

```c
#define REMOVE_LINKS(p) do {                    \
    if (thread_group_leader(p))             \
        list_del_init(&(p)->tasks);         \
    remove_parent(p);                   \
    } while (0)

#define SET_LINKS(p) do {                   \
    if (thread_group_leader(p))             \
        list_add_tail(&(p)->tasks,&init_task.tasks);    \
    add_parent(p, (p)->parent);             \
    } while (0)

#define next_task(p)    list_entry((p)->tasks.next, struct task_struct, tasks)
#define prev_task(p)    list_entry((p)->tasks.prev, struct task_struct, tasks)

#define for_each_process(p) \
    for (p = &init_task ; (p = next_task(p)) != &init_task ; )
```

# Runqueue

When looking for a new process to run on a CPU, the kernel has to consider only the **runnable** processes (that is, the processes in the `TASK_RUNNING` state).

Linux 2.6 implements the runqueue without putting all runnable processes in the same list. The aim is to allow the scheduler to select the best runnable process in constant time.

The trick is splitting the runqueue in many lists of runnable processes, one list per process priority. Each `task_struct` descriptor includes a `run_list` field which points to the corresponding runqueue.

All these are implemented by a single data structure `prio_array_t`, which is defined in `include/linux/sched.c`. Noted that each CPU has its own runqueue.

```c
struct prio_array {
    unsigned int nr_active; /* The number of process descriptors linked into the lists */
    unsigned long bitmap[BITMAP_SIZE];  /* A priority bitmap: each flag is set if and only if the corre- sponding priority list is not empty */
    struct list_head queue[MAX_PRIO]; /* The 140 heads of the priority lists */
};
```

## Enqueue/dequeue tasks

The scheduler calls `enqueue_task(p, array)` and `dequeue_task(p, array)` to insert/remove process descriptor into/from a runqueue list.

```c
static void dequeue_task(struct task_struct *p, prio_array_t *array)
{
    array->nr_active--;
    list_del(&p->run_list);
    if (list_empty(array->queue + p->prio))
        __clear_bit(p->prio, array->bitmap);
}

static void enqueue_task(struct task_struct *p, prio_array_t *array)
{
    sched_info_queued(p);
    list_add_tail(&p->run_list, array->queue + p->prio);
    __set_bit(p->prio, array->bitmap);
    array->nr_active++;
    p->array = array;
}
```

# Real parent vs. parent

The field `parent` in `task_struct` usually matches the process descriptor pointed by `real_parent`.

- `real_parent`: Points to the process descriptor of the process that **created** P or to the descriptor of process 1 (**init**) if the parent process no longer exists.
- `parent`: Points to the **current** parent of P (this is the process that must **be signaled** when the child process terminates, i.e. `SIGCHLD`). It may occasionally differ from `real_parent` in some cases, such as when another process issues a `ptrace()` system call requesting that it be allowed to monitor P.

# PID hash table

To speed up process list scanning, Linux kernel introduced four hash tables for different types of PID (defined in `include/kernel/pid.h`).

```c
enum pid_type
{
    PIDTYPE_PID,  /* PID of the process */
    PIDTYPE_TGID, /* PID of thread group leader process */
    PIDTYPE_PGID, /* PID of the group leader process */
    PIDTYPE_SID,  /* PID of the session leader process */
    PIDTYPE_MAX
};
```

The four hash tables are dynamically allocated during the kernel initialization phase, and their addresses are stored in the `pid_hash` array of `kernel/pid.c`.

```c
#define pid_hashfn(nr) hash_long((unsigned long)nr, pidhash_shift)
static struct hlist_head *pid_hash[PIDTYPE_MAX];
```

![The PID hash tables](http://i.imgur.com/2Kc7nlZ.png)

## PID data structure

The `pid` data structure is defined as:

```c
struct pid
{
    int nr; /* The PID number */
    struct hlist_node pid_chain;  /* he links to the next and previous elements in the hash chain list */
    struct list_head pid_list;  /* The head of the per-PID list */
};
```

There are sets of function used to handle PID hash table, including `find_task_by_pid(nr)`, `attach_pid(task, type, nr)`, `detach_pid(task, type)`, etc.

```c
struct pid * fastcall find_pid(enum pid_type type, int nr)
{
    struct hlist_node *elem;
    struct pid *pid;

    hlist_for_each_entry(pid, elem,
            &pid_hash[type][pid_hashfn(nr)], pid_chain) {
        if (pid->nr == nr)
            return pid;
    }
    return NULL;
}
```

## PID hash table linkage

In `task_struct`, there is a field `struct pid pids[PIDTYPE_MAX]` used for hash table linkage. The following functions use these linkages:

```c
static inline int pid_alive(struct task_struct *p)
{
    return p->pids[PIDTYPE_PID].nr != 0;  /* check that a task structure is not stale */
}
```

```c
static inline int thread_group_empty(task_t *p)
{
    return list_empty(&p->pids[PIDTYPE_TGID].pid_list);
}
```

# Wait queues

Wait queues have several uses in the kernel, particularly for interrupt handling, process synchronization, and timing. They implement conditional waits on events: a process wishing to wait for a specific event places itself in the proper wait queue and relinquishes control.

Each wait queue is identified by a *wait queue head*, which is defined in `include/kernel/wait.h`:

```c
struct __wait_queue_head {
    spinlock_t lock;
    struct list_head task_list;
};
typedef struct __wait_queue_head wait_queue_head_t;
```

Because wait queues are modified by interrupt handlers as well as by major kernel functions, the doubly linked lists are protected from concurrent accesses by the `lock` spin lock.

Elements of a wait queue list are of type `wait_queue_t`:

```c
typedef struct __wait_queue wait_queue_t;
struct __wait_queue {
    unsigned int flags;
#define WQ_FLAG_EXCLUSIVE   0x01
    struct task_struct * task;
    wait_queue_func_t func;
    struct list_head task_list;
};
```

## exclusive vs. nonexclusive

- exclusive process
  - `flags = 0`
  - selectively woken up by the kernel
  - inserted to head of wait queue
- nonexclusive process
  - `flags = 1`
  - **always** woken up by the kernel when the event occurs
  - inserted to tail of wait queue

## Handling wait queues

To initialize wait queue head and entires:

```c
static inline void init_waitqueue_head(wait_queue_head_t *q)
{
    q->lock = SPIN_LOCK_UNLOCKED;
    INIT_LIST_HEAD(&q->task_list);
}

static inline void init_waitqueue_entry(wait_queue_t *q, struct task_struct *p)
{
    q->flags = 0;
    q->task = p;
    q->func = default_wake_function;
}
```

The `add_wait_queue()` function inserts a nonexclusive process in the **first position** of a wait queue list. The `add_wait_queue_exclusive()` function inserts an exclusive process in the **last position** of a wait queue list.

A process wishing to wait for a specific condition can invoke `slepp_on()`-like functions:

```c
#define SLEEP_ON_VAR                    \
    unsigned long flags;                \
    wait_queue_t wait;              \
    init_waitqueue_entry(&wait, current);

#define SLEEP_ON_HEAD                   \
    spin_lock_irqsave(&q->lock,flags);      \
    __add_wait_queue(q, &wait);         \
    spin_unlock(&q->lock);

#define SLEEP_ON_TAIL                   \
    spin_lock_irq(&q->lock);            \
    __remove_wait_queue(q, &wait);          \
    spin_unlock_irqrestore(&q->lock, flags);

void fastcall __sched sleep_on(wait_queue_head_t *q)
{
    SLEEP_ON_VAR

    current->state = TASK_UNINTERRUPTIBLE;  /* set to TASK_INTERRUPTIBLE in interruptible_sleep_on() */

    SLEEP_ON_HEAD
    schedule();
    SLEEP_ON_TAIL
}
```

Since `sleep-on()`-like functions are a well-known source of race conditions, it is encouraged to make use of `prepare_to_wait()` and `prepare_to_wait_exclusive()`:

```c
void fastcall
prepare_to_wait(wait_queue_head_t *q, wait_queue_t *wait, int state)
{
    unsigned long flags;

    wait->flags &= ~WQ_FLAG_EXCLUSIVE;
    spin_lock_irqsave(&q->lock, flags);
    if (list_empty(&wait->task_list))
        __add_wait_queue(q, wait);  /* call __add_wait_queue_tail() instead if exclusive */
    /*
     * don't alter the task state if this is just going to
     * queue an async wait queue callback
     */
    if (is_sync_wait(wait))
        set_current_state(state);
    spin_unlock_irqrestore(&q->lock, flags);
}
```

As soon as the process is awakened, it executes the `finish_wait()` function, which sets again the process state to `TASK_RUNNING`, and removes the wait queue element from the wait queue list.

## Wake up processes

The call path to wake up a process is `wake_up` macro in `include/linux/wait.h` -> `__wake_up()` in `kernel/sched.c` -> `__wake_up_common()`.

`__wake_up_common()` is the core wakeup function. For non-exclusive wakeups (`nr_exclusive == 0`), it just wake everything up. For an exclusive wakeup, we wake **all** the non-exclusive tasks and **one** exclusive task:

```c
static void __wake_up_common(wait_queue_head_t *q, unsigned int mode,
                 int nr_exclusive, int sync, void *key)
{
    struct list_head *tmp, *next;

    list_for_each_safe(tmp, next, &q->task_list) {
        wait_queue_t *curr;
        unsigned flags;
        curr = list_entry(tmp, wait_queue_t, task_list);
        flags = curr->flags;
        if (curr->func(curr, mode, sync, key) &&
            (flags & WQ_FLAG_EXCLUSIVE) &&
            !--nr_exclusive)
            break;
    }
}
```

# Process Resource Limits

Each process has an associated set of *resource limits*, which specify the amount of system resources it can use.

The resource limits are stored in an array (`task_struct->signal`) of elements of type `struct rlimit`, which is defined in `include/linux/resource.h`:

```c
struct rlimit {
    unsigned long   rlim_cur;
    unsigned long   rlim_max;
};
```

For example, `current->signal->rlim[RLIMIT_CPU].rlim_cur` represents the current limit on the CPU time of the running process.

By using the `getrlimit()` and `setrlimit()` system calls, a user can always increase the `rlim_cur` limit of some resource up to `rlim_max`. Only the superuser (or user with `CAP_SYS_RESOURCE` capability) can increase the `rlim_max` field.

# Process Switch

## Hardware Context

The set of data that must be loaded into the registers before the process resumes its execution on the CPU is called the *hardware context*. The hardware context is a subset of the process execution context, which includes all information needed for the process execution. In Linux, a part of the hardware context of a process is stored in the process descriptor, while the remaining part is saved in the Kernel Mode stack.

Process switching occurs only in **Kernel Mode**. The contents of all registers used by a process in User Mode have already been saved on the Kernel Mode stack before performing process switching.

The 80x86 architecture includes a specific segment type called the *Task State Segment* (TSS), to store hardware contexts. While in Linux, there is only single TSS for each processor, each process descriptor includes additional field `struct thread_struct thread`, in which the kernel saves the hardware context whenever the process is being switched out. The data structutr includes fields for most of the CPU registers, except the general-purpose registers such as *eax*, *ebx*, etc., which are stored in the Kernel Mode stack.

## Performing the Process Switch

A process switch may occur at just one well-defined point: the `schedule()` function.

The `switch_to` macro is used to switch the Kernel Mode stack and the hardware context. The macro is a hardware-dependent routine (*extended inline assembly language*) that has three parameters, called **prev**, **next**, and **last**.

The `last` parameter of the `switch_to` macro is designed to preserve the reference to `prev` descriptor after process switching.

```c
static inline
task_t * context_switch(runqueue_t *rq, task_t *prev, task_t *next)
{
    struct mm_struct *mm = next->mm;
    struct mm_struct *oldmm = prev->active_mm;

    if (unlikely(!mm)) {
        next->active_mm = oldmm;
        atomic_inc(&oldmm->mm_count);
        enter_lazy_tlb(oldmm, next);
    } else
        switch_mm(oldmm, mm, next);

    if (unlikely(!prev->mm)) {
        prev->active_mm = NULL;
        WARN_ON(rq->prev_mm);
        rq->prev_mm = oldmm;
    }

    /* Here we just switch the register state and the stack. */
    switch_to(prev, next, prev);

    return prev;
}
```

# Creating Processes

## Modern Unix design

- *Copy On Write*: Allow both the parent and the child to read the same physical pages. Whenever either one tries to write on a physical page, the kernel copies its contents into a new physical page that is assigned to the writing process.
- Lightweight process: Allow both the parent and the child to share many per-process kernel data structures, such as the paging tables (and therefore the entire User Mode address space), the open file tables, and the signal dispositions.
- `vfork()` system call: Create a process that shares the memory address space of its parent. To prevent the parent from overwriting data needed by the child, the parent’s execution is blocked until the child exits or executes a new program.

## The `clone()`, `fork()`, and `vfork()` System Calls

Lightweight processes are created in Linux by using a function named `clone()`.

The traditional `fork()` system call is implemented by Linux as a `clone()` system call whose `flags` parameter specifies both a `SIGCHLD` signal and all the clone flags cleared, and whose `child_stack` parameter is the current parent stack pointer. Therefore, the parent and child temporarily share the same User Mode stack. (Copy On Write)

The `vfork()` system call, introduced in the previous section, is implemented by Linux as a `clone()` system call whose `flags` parameter specifies both a `SIGCHLD` signal and the flags `CLONE_VM` and `CLONE_VFORK`, and whose `child_stack` parameter is equal to the current parent stack pointer.

## The `do_fork()` function

The `do_fork()` function handles the `clone()`, `fork()`, and `vfork()` system calls.

```c
long pid = alloc_pidmap();
```

```c
p = copy_process(clone_flags, stack_start, regs, stack_size, parent_tidptr, child_tidptr, pid); /* new process descriptor */
```

```c
wake_up_new_task(p, clone_flags);
```

# Kernel Threads

Kernel threads run only in Kernel Mode, they use only linear addresses greater than **PAGE_OFFSET**. The `kernel_thread()` function creates a new kernel thread, and it essentially invokes `do_fork()`:

```c
do_fork(flags|CLONE_VM|CLONE_UNTRACED, 0, pregs, 0, NULL, NULL);
```

## Process 0

The ancestor of all processes, called **process 0**, the *idle process*, or, for historical reasons, the *swapper process*, is a kernel thread created from scratch during the initialization phase of Linux.

A process descriptor stored in the `init_task` variable, which is initialized by the `INIT_TASK` macro. A `thread_info` descriptor and a Kernel Mode stack stored in the `init_thread_union` variable and initialized by the `INIT_THREAD_INFO` macro. These variables and macros are defined in `init_task.c`:

```c
union thread_union init_thread_union
    __attribute__((__section__(".data.init_task"))) =
        { INIT_THREAD_INFO(init_task) };

struct task_struct init_task = INIT_TASK(init_task);  /* All other task structs will be allocated on slabs in fork.c */
```

The `start_kernel()` function initializes all the data structures needed by the kernel, enables interrupts, and creates another kernel thread, named **process 1** (i.e. *init* process):

```c
kernel_thread(init, NULL, CLONE_FS | CLONE_SIGHAND);
```

After having created the init process, process 0 executes the `cpu_idle()` function in `process.c`, which essentially consists of repeatedly executing the `hlt` assembly language instruction with the interrupts enabled. Process 0 is selected by the scheduler only when there are no other processes in the `TASK_RUNNING` state.

```c
/* For x86_64 */
void cpu_idle (void)
{
    int cpu = smp_processor_id();

    /* endless idle loop with no priority at all */
    while (1) {
        while (!need_resched()) {
            void (*idle)(void);

            if (cpu_isset(cpu, cpu_idle_map))
                cpu_clear(cpu, cpu_idle_map);
            rmb();
            idle = pm_idle;
            if (!idle)
                idle = default_idle;
            idle();
        }
        schedule();
    }
}
```

## Process 1

The kernel thread created by process 0 executes the `init()` function, which in turn completes the initialization of the kernel. Then `init()` invokes the `execve()` system call to load the executable program *init*.

```c
static void run_init_process(char *init_filename)
{
    argv_init[0] = init_filename;
    execve(init_filename, argv_init, envp_init);
}

static int init(void * unused)
{
  /* ... */

    if (sys_open((const char __user *) "/dev/console", O_RDWR, 0) < 0)
        printk("Warning: unable to open an initial console.\n");

    (void) sys_dup(0);
    (void) sys_dup(0);

    /*
     * We try each of these until one succeeds.
     *
     * The Bourne shell can be used instead of init if we are
     * trying to recover a really broken machine.
     */

    if (execute_command)
        run_init_process(execute_command);

    run_init_process("/sbin/init");
    run_init_process("/etc/init");
    run_init_process("/bin/init");
    run_init_process("/bin/sh");

    panic("No init found.  Try passing init= option to kernel.");
}
```
