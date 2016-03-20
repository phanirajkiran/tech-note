Process Scheduling
==================

# Time Sharing

Linux scheduling is based on the *time sharing* technique: several processes run in **"time multiplexing"** because the CPU time is divided into slices, one for each runnable process. If a currently running process is not terminated when its time slice or *quantum* expires, a process switch may take place. Time sharing relies on **timer interrupts** and is thus transparent to processes. No additional code needs to be inserted in the programs to ensure CPU time sharing.

# Process Preemption

Linux processes are *preemptable*. When a process enters the `TASK_RUNNING` state, the kernel checks whether its dynamic priority is greater than the priority of the currently running process. If it is, the execution of current is interrupted and the scheduler is invoked to select another process to run (usually the process that just became runnable).

A process also may be preempted when its time quantum expires. When this occurs, the `TIF_NEED_RESCHED` flag in the `thread_info` structure of the current process is set, so the scheduler is invoked when the timer interrupt handler terminates.

```c
static inline void set_ti_thread_flag(struct thread_info *ti, int flag)
{
    set_bit(flag,&ti->flags);
}

static inline void set_tsk_thread_flag(struct task_struct *tsk, int flag)
{
    set_ti_thread_flag(tsk->thread_info,flag);
}


static inline void set_tsk_need_resched(struct task_struct *tsk)
{
    set_tsk_thread_flag(tsk,TIF_NEED_RESCHED);
}
```

# The Scheduling Algorithm

The scheduling algorithm of Linux 2.6 is much more sophisticated than the earlier versions. By design, it scales well with the number of runnable processes, because it selects the process to run in **constant time**.

The scheduler always succeeds in finding a process to be executed; in fact, there is always at least one runnable process: the *swapper process*, which has **PID 0** and executes only when the CPU cannot execute other processes.

Every Linux process is always scheduled according to one of the following *scheduling classes*:

- `SCHED_FIFO`
  - A **First-In, First-Out** real-time process. When the scheduler assigns the CPU to the process, it leaves the process descriptor in its current position in the runqueue list. If no other higher-priority real-time process is runnable, the process continues to use the CPU **as long as it wishes**, even if other real-time processes that have the same priority are runnable.
- `SCHED_RR`
  - A **Round Robin** real-time process. When the scheduler assigns the CPU to the process, it puts the process descriptor at the end of the runqueue list. This policy ensures a fair assignment of CPU time to all `SCHED_RR` real-time processes that have the same priority.
- `SCHED_NORMAL`
  - A conventional, time-shared process.

# Scheduling of Conventional Processes

Every conventional process has its own *static priority*, which is a number ranging from 100 (highest priority) to 139 (lowest priority).

## Base time quantum

The static priority essentially determines the *base time quantum* of a process, that is, the time quantum duration assigned to the process when it has exhausted its previous time quantum.

```
base time quantum (ms) =
  (140 – static priority) × 20, if static priority < 120
  (140 – static priority) × 5, if static priority ≥ 120
```

As a consequence, higher priority processes usually get longer slices of CPU time with respect to lower priority processes.

## Dynamic priority and average sleep time

Besides a static priority, a conventional process also has a dynamic priority, which is a value ranging from 100 (highest priority) to 139 (lowest priority). The dynamic priority is the number actually looked up by the scheduler when selecting the new process to run.

```
dynamic priority = max (100, min (static priority − bonus + 5, 139))
```

The *bonus* is a value ranging from 0 to 10; a value less than 5 represents a penalty that lowers the dynamic priority, while a value greater than 5 is a premium that raises the dynamic priority. The value of the bonus is related to the *average sleep time* of the process.

A process is considered "interactive" if it satisfies the following formula:

```
dynamic priority ≤ 3 × static priority / 4 + 28
```

which is equivalent to the following:

```
bonus - 5 ≥ static priority / 4 − 28
```

The expression `static priority / 4 − 28` is called the *interactive delta*. It should be noted that it is far easier for high priority than for low priority processes to become interactive.

## Active and expired processes

To avoid process starvation, when a process finishes its time quantum, it can be replaced by a lower priority process whose time quantum has not yet been exhausted.

To implement it, the scheduler keeps two disjoint sets of runnable processes:

- *Active processes*
  - These runnable processes have not yet exhausted their time quantum and are thus allowed to run.
- *Expired processes*
  - These runnable processes have exhausted their time quantum and are thus for- bidden to run until all active processes expire.

The general schema is slightly more complicated than this, because the scheduler tries to boost the performance of interactive processes.

- An active **batch** process that finishes its time quantum always becomes expired.
- An active **interactive** process that finishes its time quantum usually remains active. The scheduler refills its time quantum and leaves it in the set of active processes.

However, the scheduler moves an interactive process that finished its time quantum into the set of expired processes if the **eldest expired process** has already waited for a long time, or if an expired process has higher static priority (lower value) than the interactive process. As a consequence, the set of active processes will eventually become empty and the expired processes will have a chance to run.

# Data Structures Used by the Scheduler

## runqueue

Each CPU in the system has its own runqueue; all `runqueue` structures are stored in the runqueues per-CPU variable.

`runqueue` structure is defined in `kernel/sched.c`:

```c
struct runqueue {
    spinlock_t lock;

    unsigned long nr_running;   /* number of runnable processes in the runqueue list */
#ifdef CONFIG_SMP
    unsigned long cpu_load;
#endif
    unsigned long long nr_switches;
    unsigned long nr_uninterruptible;
    unsigned long expired_timestamp;
    unsigned long long timestamp_last_tick;
    task_t *curr, *idle;   /* pointer to currently running process and swappeer */
    struct mm_struct *prev_mm;
    prio_array_t *active, *expired, arrays[2];   /* active processes and expired processes */
    int best_expired_prio;
    atomic_t nr_iowait;

#ifdef CONFIG_SMP
    struct sched_domain *sd;

    int active_balance;
    int push_cpu;

    task_t *migration_thread;
    struct list_head migration_queue;
#endif

#ifdef CONFIG_SCHEDSTATS
  /* ... */
#endif
};
```

The `arrays` field of the runqueue is an array consisting of two `prio_array_t` structures. Each data structure represents a set of runnable processes, and includes 140 doubly linked list heads (one list for each possible process priority), a priority bitmap, and a counter of the processes included in the set.


Periodically, the role of the two data structures in arrays changes: the active processes suddenly become the expired processes, and the expired processes become the active ones. To achieve this change, the scheduler simply exchanges the contents of the `active` and `expired` fields of the runqueue.

## task_struct

Each process descriptor includes several fields related to scheduling:

```c
struct task_struct {
    volatile long state;
    struct thread_info *thread_info;   /* flags stores the TIF_NEED_RESCHED flag */

    /* ... */

    int prio;   /* dynamic priority */
    int static_prio;   /* static priority */
    struct list_head run_list;   /* pointers to the next and previous elements in the runqueue */
    prio_array_t *array;   /* pointer to the runqueue's prio_array_t */

    unsigned long sleep_avg;
    unsigned long long timestamp;   /* time of last insertion of the process in the runqueue, or time of last process switch involving the process */
    unsigned long long last_ran;   /* time of last process switch that replaced the process */
    int activated;

    unsigned long policy;   /* SCHED_NORMAL, SCHED_RR, or SCHED_FIFO */
    cpumask_t cpus_allowed;
    unsigned int time_slice;   /* ticks left in the time quantum of the process */
    unsigned int first_time_slice;   /* flag set to 1 if the process never exhausted its time quantum */

    unsigned long rt_priority;   /* real-time priority */

    /* ... */
};
```

When a new process is created, `sched_fork()`, invoked by `copy_process()`, sets the `time_slice` field of both `current` (the parent) and `p` (the child) processes in the following way:

```c
/* split the number of ticks left in two halves */
p->time_slice = (current->time_slice + 1) >> 1;
current->time_slice >>= 1;
```

The `copy_process()` function also initializes a few other fields of the child’s process descriptor related to scheduling:

```c
p->first_time_slice = 1;   /* never exhausted its time quantum */
p->timestamp = sched_clock();
```

# Functions Used by the Scheduler

## `scheduler_tick()`

`scheduler_tick()` is invoked once every tick to perform some operations related to scheduling.

```c
void scheduler_tick(void)
{
    int cpu = smp_processor_id();
    runqueue_t *rq = this_rq();
    task_t *p = current;

    rq->timestamp_last_tick = sched_clock();

    if (p == rq->idle) {   /* if swapper */
        if (wake_priority_sleeper(rq))   /* find runnable process */
            goto out;
        rebalance_tick(cpu, rq, SCHED_IDLE);
        return;
    }

    /* Task might have expired already, but not scheduled off yet */
    if (p->array != rq->active) {
        set_tsk_need_resched(p);
        goto out;
    }
    spin_lock(&rq->lock);

    if (rt_task(p)) {   /* real-time process */
      /* FIFO tasks have no timeslices */
        if ((p->policy == SCHED_RR) && !--p->time_slice) {
            p->time_slice = task_timeslice(p);
            p->first_time_slice = 0;
            set_tsk_need_resched(p);

            /* put it at the end of the queue: */
            requeue_task(p, rq->active);
        }
        goto out_unlock;
    }
    if (!--p->time_slice) {
        dequeue_task(p, rq->active);
        set_tsk_need_resched(p);   /* set TIF_NEED_RESCHED flag */
        p->prio = effective_prio(p);
        p->time_slice = task_timeslice(p);
        p->first_time_slice = 0;

        if (!rq->expired_timestamp)
            rq->expired_timestamp = jiffies;
        /* EXPIRED_STARVING macro checks whether the first expired process in the runqueue had to wait for more than 1000 ticks times the number of runnable processes in the runqueue plus one */
        if (!TASK_INTERACTIVE(p) || EXPIRED_STARVING(rq)) {
            enqueue_task(p, rq->expired);
            if (p->static_prio < rq->best_expired_prio)
                rq->best_expired_prio = p->static_prio;
        } else
            enqueue_task(p, rq->active);
    } else {
      /* Prevent a too long timeslice */
        if (TASK_INTERACTIVE(p) && !((task_timeslice(p) -
            p->time_slice) % TIMESLICE_GRANULARITY(p)) &&
            (p->time_slice >= TIMESLICE_GRANULARITY(p)) &&
            (p->array == rq->active)) {

            requeue_task(p, rq->active);
            set_tsk_need_resched(p);
        }
    }
out_unlock:
    spin_unlock(&rq->lock);
out:
    rebalance_tick(cpu, rq, NOT_IDLE);
}
```

## `try_to_wake_up()`

The `try_to_wake_up()` function awakes a sleeping or stopped process by setting its state to `TASK_RUNNING` and inserting it into the runqueue of the local CPU.

```c
static int try_to_wake_up(task_t * p, unsigned int state, int sync)
{
    int cpu, this_cpu, success = 0;
    unsigned long flags;
    long old_state;
    runqueue_t *rq;
#ifdef CONFIG_SMP
    unsigned long load, this_load;
    struct sched_domain *sd;
    int new_cpu;
#endif

    rq = task_rq_lock(p, &flags);
    schedstat_inc(rq, ttwu_cnt);
    old_state = p->state;
    if (!(old_state & state))   /* check state */
        goto out;

    if (p->array)   /* check if already belongs to a runqueue */
        goto out_running;

    cpu = task_cpu(p);
    this_cpu = smp_processor_id();

#ifdef CONFIG_SMP
    /* Check whether the process to be awakened should be migrated to another CPU */
    /* ... */
#endif

    if (old_state == TASK_UNINTERRUPTIBLE) {
        rq->nr_uninterruptible--;
        /*
         * Tasks on involuntary sleep don't earn
         * sleep_avg beyond just interactive state.
         */
        p->activated = -1;
    }

    activate_task(p, rq, cpu == this_cpu);
    if (!sync || cpu != this_cpu) {
        if (TASK_PREEMPTS_CURR(p, rq))
            resched_task(rq->curr);
    }
    success = 1;

out_running:
    p->state = TASK_RUNNING;
out:
    task_rq_unlock(rq, &flags);

    return success;
}
```

## `recalc_task_prio()`

The `recalc_task_prio()` function in `kernel/sched.c` updates the average sleep time and the dynamic priority of a process. It receives as its parameters a process descriptor pointer p and a timestamp now computed by the `sched_clock()` function.

## `schedule()`

The `schedule()` function implements the scheduler. Its objective is to find a process in the runqueue list and then assign the CPU to it. It is invoked, directly or in a lazy (deferred) way, by several kernel routines.

## Direct invocation

The scheduler is invoked directly when the current process must be blocked right away because the resource it needs is not available. In this case, the kernel routine that wants to block it proceeds as follows:

1. Inserts `current` in the proper wait queue.
2. Changes the state of current either to `TASK_INTERRUPTIBLE` or to `TASK_
UNINTERRUPTIBLE`.
3. Invoke `schedule()` to yields the CPU.
4. Checks whether the resource is available; if not, goes to step 2.
5. Once the resource is available, removes current from the `wait queue`.

## Lazy invocation

The scheduler can also be invoked in a lazy way by setting the `TIF_NEED_RESCHED` flag of current to 1. The check is made before resuming the execution of a User Mode process.

- When `current` has used up its quantum of CPU time; this is done by the
`scheduler_tick()` function.
- When a process is woken up and its priority is higher than that of the current
process; this task is performed by `the try_to_wake_up()` function.
- When a `sched_setscheduler()` system call is issued

## Function implementation

```c
asmlinkage void __sched schedule(void)
{
    /* ... */

need_resched:
    preempt_disable();
    prev = current;
    release_kernel_lock(prev);   /* by checking lock_depth */
need_resched_nonpreemptible:
    rq = this_rq();

    /* ... */

    now = sched_clock();
    if (likely(now - prev->timestamp < NS_MAX_SLEEP_AVG))
        run_time = now - prev->timestamp;
    else
        run_time = NS_MAX_SLEEP_AVG;

    run_time /= (CURRENT_BONUS(prev) ? : 1);

    spin_lock_irq(&rq->lock);

    if (unlikely(prev->flags & PF_DEAD))   /* prev might be terminted */
        prev->state = EXIT_DEAD;

    switch_count = &prev->nivcsw;
    if (prev->state && !(preempt_count() & PREEMPT_ACTIVE)) {   /* state != TASK_RUNNING */
        switch_count = &prev->nvcsw;
        if (unlikely((prev->state & TASK_INTERRUPTIBLE) &&
                unlikely(signal_pending(prev))))
            prev->state = TASK_RUNNING;
        else {
            if (prev->state == TASK_UNINTERRUPTIBLE)
                rq->nr_uninterruptible++;
            deactivate_task(prev, rq);
        }
    }

    cpu = smp_processor_id();
    if (unlikely(!rq->nr_running)) {
go_idle:
        idle_balance(cpu, rq);
        if (!rq->nr_running) {
            next = rq->idle;   /* swapper process */
            rq->expired_timestamp = 0;
            wake_sleeping_dependent(cpu, rq);
            if (!rq->nr_running)
                goto switch_tasks;
        }
    } else {
        if (dependent_sleeper(cpu, rq)) {
            next = rq->idle;
            goto switch_tasks;
        }
        if (unlikely(!rq->nr_running))
            goto go_idle;
    }

    array = rq->active;
    if (unlikely(!array->nr_active)) {   /* no active runnable process */
        schedstat_inc(rq, sched_switch);
        rq->active = rq->expired;   /* excahnge active queue and expired queue */
        rq->expired = array;
        array = rq->active;
        rq->expired_timestamp = 0;
        rq->best_expired_prio = MAX_PRIO;
    } else
        schedstat_inc(rq, sched_noswitch);

  /* Look up a runnable process */
    idx = sched_find_first_bit(array->bitmap);
    queue = array->queue + idx;
    next = list_entry(queue->next, task_t, run_list);

    /* ... */

switch_tasks:
    if (next == rq->idle)
        schedstat_inc(rq, sched_goidle);
    prefetch(next);
    clear_tsk_need_resched(prev);   /* clear TIF_NEED_RESCHED flag */
    rcu_qsctr_inc(task_cpu(prev));

    prev->sleep_avg -= run_time;
    if ((long)prev->sleep_avg <= 0)
        prev->sleep_avg = 0;
    prev->timestamp = prev->last_ran = now;

    sched_info_switch(prev, next);
    if (likely(prev != next)) {
        next->timestamp = now;
        rq->nr_switches++;
        rq->curr = next;
        ++*switch_count;

        prepare_arch_switch(rq, next);
        prev = context_switch(rq, prev, next);   /* set up address space for next */
        barrier();

        finish_task_switch(prev);
    } else
        spin_unlock_irq(&rq->lock);

    prev = current;
    if (unlikely(reacquire_kernel_lock(prev) < 0))
        goto need_resched_nonpreemptible;
    preempt_enable_no_resched();
    if (unlikely(test_thread_flag(TIF_NEED_RESCHED)))
        goto need_resched;
}
```

# Runqueue Balancing

## Scheduling Domains

A *scheduling domain* is a set of CPUs whose workloads should be kept balanced by the kernel. Generally speaking, scheduling domains are hierarchically organized: the top-most scheduling domain, which usually spans all CPUs in the system, includes children scheduling domains, each of which include a subset of the CPUs.

Every scheduling domain is represented by a `sched_domain` descriptor defined in `include/linux/sched.h`, while every group inside a scheduling domain is represented by a `sched_group` descriptor. Each `sched_domain` descriptor includes a field groups, which points to the first element in a list of group descriptors. Moreover, the parent field of the `sched_domain` structure points to the descriptor of the parent scheduling domain, if any.

## `rebalance_tick()`

To keep the runqueues in the system balanced, the `rebalance_tick()` function is invoked by `scheduler_tick()` once every tick.

It starts a loop over all scheduling domains in the path from the base domain (referenced by the `sd` field of the local `runqueue` descriptor) to the top-level domain. In each iteration the function determines whether the time has come to invoke the `load_balance()` function, thus executing a rebalancing operation on the scheduling domain.

## `load_balance()`

The `load_balance()` function checks whether a scheduling domain is significantly unbalanced; more precisely, it checks whether unbalancing can be reduced by moving some processes from the busiest group to the runqueue of the local CPU.

```c
static int load_balance(int this_cpu, runqueue_t *this_rq,
            struct sched_domain *sd, enum idle_type idle)
{
    struct sched_group *group;
    runqueue_t *busiest;
    unsigned long imbalance;
    int nr_moved;

    spin_lock(&this_rq->lock);
    schedstat_inc(sd, lb_cnt[idle]);

  /* Analyze the workloads of the groups inside the scheduling domain, and return the busiest group */
    group = find_busiest_group(sd, this_cpu, &imbalance, idle);
    if (!group) {
        schedstat_inc(sd, lb_nobusyg[idle]);
        goto out_balanced;
    }

  /* Find the busiest CPUs in the group */
    busiest = find_busiest_queue(group);
    if (!busiest) {
        schedstat_inc(sd, lb_nobusyq[idle]);
        goto out_balanced;
    }

    /* ... */

    nr_moved = 0;
    if (busiest->nr_running > 1) {
        double_lock_balance(this_rq, busiest);
        /* Try moving some processes from the busiest runqueue to the local runqueue this_rq */
        nr_moved = move_tasks(this_rq, this_cpu, busiest,
                        imbalance, sd, idle);
        spin_unlock(&busiest->lock);
    }
    spin_unlock(&this_rq->lock);

    if (!nr_moved) {
        /* Wake up migration kernel thread */
        /* ... */
    } else {
        sd->nr_balance_failed = 0;

        sd->balance_interval = sd->min_interval;
    }

    return nr_moved;

out_balanced:
    spin_unlock(&this_rq->lock);

    /* ... */

    return 0;
}
```

# System Calls Related to Scheduling

## `nice()`

The `nice()` system call allows processes to change their base priority. The `sys_nice()` service routine in `kernel/sched.c` handles the `nice()` system call:

```c
asmlinkage long sys_nice(int increment)
{
    int retval;
    long nice;

    if (increment < 0) {
        if (!capable(CAP_SYS_NICE))   /* check CAP_SYS_NICE for negative increment */
            return -EPERM;
        if (increment < -40)
            increment = -40;
    }
    if (increment > 40)
        increment = 40;

    nice = PRIO_TO_NICE(current->static_prio) + increment;
    if (nice < -20)
        nice = -20;
    if (nice > 19)
        nice = 19;

    retval = security_task_setnice(current, nice);   /* security hook */
    if (retval)
        return retval;

    set_user_nice(current, nice);
    return 0;
}
```

The `nice()` system call is maintained for backward compatibility only; it has been replaced by the `setpriority()` system call described next.

## `getpriority()` and `setpriority()`

The kernel implements these system calls by means of the `sys_getpriority()` and `sys_setpriority()` service routines. Both of them act essentially on the same group of parameters:

- `which`
  - `PRIO_PROCESS`
  - `PRIO_PGRP`
  - `PRIO_USER`
- `who`
  - the value of `pid`, `pgrp`, or `uid` field (depending on the value of `which`)
  - 0 for `current` process
- `niceval`
  - only for `sys_setpriority()`
  - range between –20 (highest priority) and +19 (lowest priority)

Moreover, `getpriority()` does not return a normal nice value ranging between –20 and +19, but rather a nonnegative value ranging **between 1 and 40**.

## `sched_getaffinity()` and `sched_setaffinity()`

The `sched_getaffinity()` and `sched_setaffinity()` system calls respectively return and set up the CPU affinity mask of a process — the **bit mask** of the CPUs that are allowed to execute the process. This mask is stored in the `cpus_allowed` field of the process descriptor.

The `sys_sched_setaffinity()` system call is more complicated. Besides looking for the descriptor of the target process and updating the `cpus_allowed` field, this function has to check whether the process is included in a runqueue of a CPU that is no longer present in the new affinity mask. In the worst case, the process has to be moved from one runqueue to another one. To avoid problems due to **deadlocks** and **race conditions**, this job is done by the *migration* kernel threads (there is one thread per CPU).

## System Calls Reltated to Real-Time Processes

- `sched_getscheduler()`: queries the scheduling policy (`SCHED_FIFO`, `SCHED_RR`, or `SCHED_NORMAL`) currently applied to the process identified by the `pid` parameter.
- `sched_setscheduler()`: sets both the scheduling policy and the associated parameters for the process identified by the parameter `pid`.
- `sched_getparam()`: retrieves the scheduling parameters for the process identified by `pid`.
- `sched_setparam()`: similar to `sched_setscheduler()` but does not let the caller to set the `policy` field's value.
- `sched_yield()`: relinquish the CPU voluntarily without being suspended and remains `TASK_RUNNING` state.
