Process Environment
===================

# `main` Function

A C program starts execution with a function called `main`. The prototype for the main function is:

```c
int main(int argc, char *argv[]);
```

where `argc` is the number of command-line arguments, and `argv` is an array of pointers to the arguments.

Each program is also passed an *environment list*. By convention, the environment consists of `name=value` string. The address of the array of pointers is contained in the global variable `environ`:

```c
extern char **environ;
```

When a C program is executed by the kerne, a special **start-up routine** is called before the `main` function is called. The executable program file specifies this routine as the starting address for the program; this is set up by the **linker** when it is invoked by the C compiler. This start-up routine takes values from the kernel — the **command-line arguments** and the **environment** — and sets things up so that the `main` function is called.

# Process Termination

There are eight ways for a process to terminate. Normal termination occurs in five ways:

1. Return from `main`
2. Calling `exit`
3. Calling `_exit` or `_Exit`
4. Return of the last thread from its start routine
5. Calling `pthread_exit` from the last thread

Abnormal termination occurs in three ways:

1. Calling `abort`
2. Receipt of a signal
3. Response of the last thread to a cancellation request

The start-up routine is also written so that if the `main` function returns, the `exit` function is called:

```c
exit(main(argc, argv));
```

## `atexit` Function

With ISO C, a process can register at least 32 functions that are automatically called by `exit`. These are called *exit handlers* and are registered by calling the `atexit` function:

```c
#include <stdlib.h>

/* Returns: 0 if OK, nonzero on error */
int atexit(void (*func)(void));
```

![How a C program is started and how it terminates](http://i.imgur.com/kRC04Li.png)

# Memory Layout of a C Program

![Typical memory arrangement](http://i.imgur.com/dAIwchg.png)

- Text segment
  - Consists of the machine instructions
  - **Sharable**, read-only
- Data segment
  - Contains variables that are specifically initialized in the program
  - For example, `int maxcount = 99;`
- bss segment
  - Uninitialized data segment
  - Initialized by the kernel to arithmetic 0 or null pointers before the program starts executing
  - For example, `long sum[1000]`
- Stack
  - Stores automatic variables
  - Each time a function is called, the address of where to return to and certain information about the caller’s environment, such as some of the machine registers, are saved on the stack.
- Heap
  - Dynamic memory allocation

# Memory Allocation

ISO C specifies three functions for memory allocation:

```c
#include <stdlib.h>

/* All three return: non-null pointer if OK, NULL on error */
void *malloc(size_t size);
void *calloc(size_t nobj, size_t size);   /* The space is initialized to all 0 bits */
void *realloc(void *ptr, size_t newsize);

void free(void *ptr);
```

The allocation routines are usually implemented with the `sbrk(2)` system call. This system call expands (or contracts) the **heap** of the process. Although `sbrk` can expand or contract the memory of a process, most versions of malloc and free never decrease their memory size. The space that we free is available for a later allocation, but the freed space is not usually returned to the kernel; instead, that space is kept in the **malloc pool**.

## Alternative: `alloca` Function

The function `alloca` has the same calling sequence as `malloc`; however, instead of allocating memory from the heap, the memory is allocated from the **stack** frame of the current function. The advantage is that we don’t have to free the space; it goes away automatically when the function returns.

# Environment Variables

ISO C defines a function that we can use to fetch values from the environment:

```c
#include <stdlib.h>

/* Returns: pointer to value associated with name, NULL if not found */
char *getenv(const char *name);
```

Note that this function returns a pointer to the value of a `name=value` string. We should always use `getenv` to fetch a specific value from the environment, instead of accessing `environ` directly.

There are also some functions provided for changing the value of an existing variable or adding a new variable to the environment:

```c
#include <stdlib.h>

/* Returns: 0 if OK, nonzero on error */
int putenv(char *str);
int setenv(const char *name, const char *value, int rewrite);

/* Both return: 0 if OK, −1 on error */
int unsetenv(const char *name);
```

Note that the `putenv` function takes a string of the form `name=value`, but the `setenv` function sets `name` to `value`.

# `setjmp` and `longjmp` Functions

In C, we can’t `goto` a label that’s in another function. Instead, we must use the **nonlocal goto** - `setjmp` and `longjmp` functions - to perform this type of branching:

```c
#include <setjmp.h>

/* Returns: 0 if called directly, nonzero if returning from a call to longjmp */
int setjmp(jmp_buf env);

void longjmp(jmp_buf env, int val);
```

We call `setjmp` from the location that we want to return to. In the call to `setjmp`, the argument `env` is of the special type `jmp_buf`. This data type is some form of array that is capable of holding all the information required to restore the status of the stack to the state when we call `longjmp`. Normally, the `env` variable is a **global** variable, since we’ll need to reference it from another function.

When we encounter an error and want to return, we call `longjmp` with two arguments. The first is the same `env` that we used in a call to `setjmp`, and the second, `val`, is a nonzero value that becomes the return value from `setjmp`.

# `getrlimit` and `setrlimit` Functions

Every process has a set of resource limits, some of which can be queried and changed by the `getrlimit` and `setrlimit` functions:

```c
#include <sys/resource.h>

/* Both return: 0 if OK, −1 on error */
int getrlimit(int resource, struct rlimit *rlptr);
int setrlimit(int resource, const struct rlimit *rlptr);
```

Each call to these two functions specifies a single resource and a pointer to the following structure:

```c
struct rlimit {
    rlim_t  rlim_cur;  /* soft limit: current limit */
    rlim_t  rlim_max;  /* hard limit: maximum value for rlim_cur */
};
```

Three rules govern the changing of the resource limits:

1. A process can change its soft limit to a value less than or equal to its hard limit.
2. A process can lower its hard limit to a value greater than or equal to its soft limit. This lowering of the hard limit is irreversible for normal users.
3. Only a superuser process can raise a hard limit.

The resource limits affect the calling process and are inherited by any of its children.

The following list shows which limits are defined by the Single UNIX Specification and supported by each implementation:

- `RLIMIT_AS`: The maximum size in bytes of a process’s total available memory. This affects the `sbrk` function and the `mmap` function.
- `RLIMIT_CORE`: The maximum size in bytes of a core file. A limit of 0 prevents the creation of a core file.
- `RLIMIT_CPU`: The maximum amount of CPU time in seconds. When the soft limit is exceeded, the `SIGXCPU` signal is sent to the process.
- `RLIMIT_DATA`: The maximum size in bytes of the data segment: the sum of the initialized data, uninitialized data, and heap.
- `RLIMIT_FSIZE`:
The maximum size in bytes of a file that may be created. When the soft limit is exceeded, the process is sent the `SIGXFSZ` signal.
- `RLIMIT_MEMLOCK`: The maximum amount of memory in bytes that a process can lock into memory using `mlock(2)`.
- `RLIMIT_MSGQUEUE`: The maximum amount of memory in bytes that a process can allocate for POSIX message queues.
- `RLIMIT_NICE`: The limit to which a process’s nice value can be raised to affect its scheduling priority.
- `RLIMIT_NOFILE`: The maximum number of open files per process.
- `RLIMIT_NPROC`: The maximum number of child processes per real user ID.
- `RLIMIT_NPTS`: The maximum number of pseudo terminals that a user can have open at one time.
- `RLIMIT_RSS`: Maximum resident set size (RSS) in bytes. If available physical memory is low, the kernel takes memory from processes that exceed their RSS.
- `RLIMIT_SBSIZE`: The maximum size in bytes of socket buffers that a user can consume at any given time.
- `RLIMIT_SIGPENDING`: The maximum number of signals that can be queued for a process. This limit is enforced by the `sigqueue` function.
- `RLIMIT_STACK`: The maximum size in bytes of the stack.
- `RLIMIT_SWAP`: The maximum amount of swap space in bytes that a user can consume.
- `RLIMIT_VMEM`: This is a synonym for `RLIMIT_AS`.
