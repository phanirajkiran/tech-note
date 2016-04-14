Process Control
===============

# Process Identifiers

Every process has a unique process ID, a non-negative integer. There are some special processes:

- Process ID 0
  - scheduler process, known as *swapper*
  - a system process
- Process ID 1
  - *init* process invoked by the kernel at the end of bootstrap procedure.
  - reads the system-dependent initialization files — the `/etc/rc*` files or `/etc/inittab` and the files in `/etc/init.d` — and brings the system to a certain state, such as multiuser.
  - a normal user process, never dies.
- Process ID 2 (in some implementation)
  - *pagedaemon* process, responsible for supporting the paging of the virtual memory system.

```c
#include <unistd.h>

/* Returns: process ID of calling process */
pid_t getpid(void);

/* Returns: parent process ID of calling process */
pid_t getppid(void);

/* Returns: real user ID of calling process */
uid_t getuid(void);

/* Returns: effective user ID of calling process */
uid_t geteuid(void);

/* Returns: real group ID of calling process */
gid_t getgid(void);

/* Returns: effective group ID of calling process */
gid_t getegid(void);
```

# `fork` Function

An existing process can create a new one by calling the `fork` function:

```c
#include <unistd.h>

/* Returns: 0 in child, process ID of child in parent, −1 on error */
pid_t fork(void);
```

The new process created by `fork` is called the *child process*. This function is called once but **returns twice**. The only difference in the returns is that the return value in the child is 0, whereas the return value in the parent is the process ID of the new child. Both the child and the parent continue executing with the instruction that follows the call to `fork`.

The child is a copy of the parent, but modern implementations don’t perform a complete copy of the parent’s data, stack, and heap, since a `fork` is often followed by an `exec`. Instead, a technique called *copy-on-write* (COW) is used. These regions are shared by the parent and the child and have their protection changed by the kernel to **read-only**. If either process tries to modify these regions, the kernel then makes a copy of that piece of memory only, typically a "page" in a virtual memory system.

`fork` may fail due to the following two main reasons:

- Too many processes are already in the system
- The total number of processes for this real user ID exceeds the system’s limit (`CHILD_MAX`)

## Parent process vs. child process

Numerous properties of the parent are inherited by the child:

- Open files
- Real user ID, real group ID, effective user ID, and effective group ID
- Supplementary group IDs
- Process group ID
- Session ID
- Controlling terminal
- The set-user-ID and set-group-ID flags
- Current working directory
- Root directory
- File mode creation mask
- Signal mask and dispositions
- The close-on-exec flag for any open file descriptors
- Environment
- Attached shared memory segments
- Memory mappings
- Resource limits

The differences between the parent and child are:

- The return values from fork are different.
- The process IDs are different.
- The two processes have different parent process IDs: the parent process ID of the child is the parent; the parent process ID of the parent doesn’t change.
- The child’s `tms_utime`, `tms_stime`, `tms_cutime`, and `tms_cstime` values are set to 0.
- File locks set by the parent are not inherited by the child.
- Pending alarms are cleared for the child.
- The set of pending signals for the child is set to the empty set.

# `vfork` Function

The `vfork` function was intended to create a new process for the purpose of executing a new program. The `vfork` function creates the new process, and the child runs in the **address space of the parent** until it calls either `exec` or `exit`. Also, `vfork` guarantees that the child runs first, until the child calls `exec` or `exit`.

# `wait` and `waitpid` Functions

The parent of the process can obtain the termination status of the child processes from either the `wait` or the `waitpid` function. In UNIX System terminology, a process that has terminated, but whose parent has not yet waited for it, is called a *zombie*. Notice that if parent process terminates before the child, it will be inherited by **init** process so that the parent process ID is changed to be 1.

```c
#include <sys/wait.h>

/* Both return: process ID if OK, 0 (see later), or −1 on error */
pid_t wait(int *statloc);
pid_t waitpid(pid_t pid, int *statloc, int options);
```

The differences between these two functions are as follows:

- The `wait` function can block the caller until a child process terminates, whereas `waitpid` has an option that prevents it from blocking.
- The `waitpid` function doesn’t wait for the child that terminates first; it has a number of options that control which process it waits for.

The interpretation of the `pid` argument for `waitpid` depends on its value:

- `pid == -1`: Waits for any child process. In this respect, `waitpid` is equivalent to `wait`.
- `pid > 0`: Waits for the child whose process ID equals `pid`.
- `pid == 0`: Waits for any child whose process group ID equals that of the calling process.
- `pid < -1`: Waits for any child whose process group ID equals the absolute value of `pid`.

The `options` argument lets us further control the operation of `waitpid`. This argument either is 0 or is constructed from the **bitwise OR** of the constants:

- `WCONTINUED`: If the implementation supports job control, the status of any child specified by `pid` that has been continued after being stopped, but whose status has not yet been reported, is returned.
- `WNOHANG`: The `waitpid` function will not block if a child specified by `pid` is not immediately available. In this case, the return value is 0. (**nonblocking**)
- `WUNTRACED`: If the implementation supports job control, the status of any child specified by `pid` that has stopped, and whose status has not been reported since it has stopped, is returned. The `WIFSTOPPED` macro determines whether the return value corresponds to a stopped child process.

For both functions, the argument `statloc` is a **pointer to an integer**. If this argument is not a null pointer, the termination status of the terminated process is stored in the location pointed to by the argument. POSIX.1 specifies that the termination status is to be looked at using various macros that are defined in `<sys/wait.h>`. Four mutually exclusive macros tell us how the process terminated:

- `WIFEXITED(status)`
  - True if status was returned for a child that terminated normally.
  - `WEXITSTATUS(status)` can be used to fetch the low-order 8 bits of the argument that the child passed to `exit`, `_exit`, or `_Exit`.
- `WIFSIGNALED(status)`
  - True if status was returned for a child that terminated abnormally, by receipt of a signal that it didn’t catch.
  - `WTERMSIG(status)` can be used to fetch the signal number that caused the termination.
- `WIFSTOPPED(status)`
  - True if status was returned for a child that is currently stopped.
  - `WSTOPSIG(status)` can be used to fetch the signal number that caused the child to stop.
- `WIFCONTINUED(status)`
  - True if status was returned for a child that has been continued after a job control stop.

# `waitid` Function

The `waitid` function is similar to `waitpid`, but provides extra flexibility:

```c
#include <sys/wait.h>

/* Returns: 0 if OK, −1 on error */
int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);
```

The `id` parameter is interpreted based on the value of `idtype`:

- `P_PID`: Wait for a particular process: `id` contains the process ID of the child to wait for.
- `P_PGID`: Wait for any child process in a particular process group: `id` contains the process group ID of the children to wait for.
- `P_ALL`: Wait for any child process: `id` is ignored.

The `options` argument is a bitwise OR of the flags:

- `WCONTINUED`: Wait for a process that has previously stopped and has been continued, and whose status has not yet been reported.
- `WEXITED`: Wait for processes that have exited.
- `WNOHANG`: Return immediately instead of blocking if there is no child exit status available.
- `WNOWAIT`: Don’t destroy the child exit status. The child’s exit status can be retrieved by a subsequent call to `wait`, `waitid`, or `waitpid`.
- `WSTOPPED`: Wait for a process that has stopped and whose status has not yet been reported.

At least one of `WCONTINUED`, `WEXITED`, or `WSTOPPED` must be specified in the `options` argument.

The `infop` argument is a pointer to a `siginfo` structure. This structure contains detailed information about the signal generated that caused the state change in the child process.

# `wait3` and `wait4` Functions

The only feature provided by these two functions that isn’t provided by the `wait`, `waitid`, and `waitpid` functions is an additional argument that allows the kernel to return a summary of the resources used by the terminated process and all its child processes.

```c
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/time.h>
#include <sys/resource.h>

/* Both return: process ID if OK, 0, or −1 on error */
pid_t wait3(int *statloc, int options, struct rusage *rusage);
pid_t wait4(pid_t pid, int *statloc, int options, struct rusage *rusage);
```

The resource information includes such statistics as the amount of user CPU time, amount of system CPU time, number of page faults, number of signals received, and the like.

# `exec` Functions

When a process calls one of the `exec` functions, that process is completely replaced by the new program, and the new program starts executing at its `main` function. The process ID does not change across an exec, because a new process is not created; `exec` merely replaces the current process — its **text, data, heap, and stack segments** — with a brand-new program from disk.

```c
#include <unistd.h>

/* All seven return: −1 on error, no return on success */
int execl(const char *pathname, const char *arg0, ..., (char *)0);
int execv(const char *pathname, char *const argv[]);
int execle(const char *pathname, const char *arg0, ..., (char *)0, char *const envp[]);
int execve(const char *pathname, char *const argv[], char *const envp[]);
int execlp(const char *filename, const char *arg0, ..., (char *)0);
int execvp(const char *filename, char *const argv[]);
int fexecve(int fd, char *const argv[], char *const envp[]);
```

When a `filename` argument is specified,

- If `filename` contains a slash, it is taken as a `pathname`.
- Otherwise, the executable file is searched for in the directories specified by the `PATH` environment variable.

The arguments for these seven `exec` functions are difficult to remember. The letters in the function names help somewhat:

- The letter *p*: takes a filename argument and uses the PATH environment variable to find the executable file.
- The letter *l*: takes a list of arguments and is mutually exclusive with the letter v, which means that it takes an `argv[]` **vector**.
- The letter *e*: takes an `envp[]` array instead of using the current environment.

![Differences among the seven exec functions](http://i.imgur.com/aYNjJRI.png)

In many UNIX system implementations, only one of these seven functions, `execve`, is a system call within the kernel. The other six are just **library functions** that eventually invoke this system call.

![Relationship of the seven exec functions](http://i.imgur.com/JIq0lVZ.png)

# Changing User IDs and Group IDs

In general, we try to use the *least-privilege* model when we design our applications. According to this model, our programs should use the least privilege necessary to accomplish any given task. We can set the real user ID and effective user ID with the `setuid` function. Similarly, we can set the real group ID and the effective group ID with the `setgid` function:

```c
#include <unistd.h>

/* Both return: 0 if OK, −1 on error */
int setuid(uid_t uid);
int setgid(gid_t gid);
```

There are rules for who can change the IDs:

- If the process has **superuser** privileges, the `setuid` function sets the real user ID, effective user ID, and saved set-user-ID to `uid`.
- If the process does not have superuser privileges, but `uid` equals either the real user ID or the saved set-user-ID, `setuid` sets only the effective user ID to `uid`. The real user ID and the saved set-user-ID are not changed.
- If neither of these two conditions is true, `errno` is set to `EPERM` and −1 is returned.

For the three user IDs that the kernel maintains:

- Only a superuser process can change the real user ID. Normally, the real user ID is set by the `login(1)` program when we log in and never changes. Because login is a superuser process, it sets all three user IDs when it calls `setuid`.
- The effective user ID is set by the `exec` functions only if the **set-user-ID** bit is set for the program file. If the set-user-ID bit is not set, the `exec` functions leave the effective user ID as its current value. We can call `setuid` at any time to set the effective user ID to either the real user ID or the saved set-user-ID. Naturally, we can’t set the effective user ID to any random value.
- The saved set-user-ID is copied from the effective user ID by `exec`. If the file’s set-user-ID bit is set, this copy is saved after exec stores the effective user ID from the file’s user ID.

![Ways to change the three user IDs](http://i.imgur.com/ukJ0eVc.png)

![Summary of all the functions that set the various user IDs](http://i.imgur.com/bb4KVBb.png)

# Interpreter Files

All contemporary UNIX systems support interpreter files. These files are text files that begin with a line of the form:

```
#! pathname [ optional-argument ]
```

The `pathname` is normally an absolute pathname, for example, `#!/bin/sh`.

The recognition of these files is done within the kernel as part of processing the `exec` system call. The actual file that gets executed by the kernel is not the interpreter file, but rather the file specified by the `pathname` on the first line of the interpreter file.

# `system` Function

We can use `system` function to execute a command string from within a program:

```c
#include <stdlib.h>

int system(const char *cmdstring);
```

`system` is basically implemented by calling `fork`, `exec`, and `waitpid`, the following snippet shows a simplified version:

```c
int
system(const char *cmdstring)   /* version without signal handling */
{
    pid_t   pid;
    int     status;

    if (cmdstring == NULL)
        return(1);      /* always a command processor with UNIX */
    if ((pid = fork()) < 0) {
        status = -1;    /* probably out of processes */
    } else if (pid == 0) {              /* child */
        execl("/bin/sh", "sh", "-c", cmdstring, (char *)0);
        _exit(127);     /* execl error */
    } else {                            /* parent */
        while (waitpid(pid, &status, 0) < 0) {
            if (errno != EINTR) {
                status = -1; /* error other than EINTR from waitpid() */
                break;
            }
        }
    }
    return(status);
}
```

Notice that if we call `system` from a **set-user-ID** program, it may create a security hole and should never be attempted.

# User Identification

We could call `getpwuid(getuid())` to find out the login name of the user; however, a person might have multiple entries in the password file with the same user ID to have a different login shell for each entry. `getlogin` function provides a way to fetch the right login name:

```c
#include <unistd.h>

/* Returns: pointer to string giving login name if OK, NULL on error */
char *getlogin(void);
```

# Process Scheduling

Historically, the UNIX System provided processes with only coarse control over their scheduling priority. The scheduling policy and priority were determined by the kernel. A process could choose to run with lower priority by adjusting its `nice` value (thus a process could be "nice" and reduce its share of the CPU by adjusting its nice value).

```c
#include <unistd.h>

/* Returns: new nice value − NZERO if OK, −1 on error */
int nice(int incr);
```

The `incr` argument is added to the nice value of the calling process.

There are also two functions provided to get or set the nice value for a process:

```c
#include <sys/resource.h>

/* Returns: nice value between −NZERO and NZERO−1 if OK, −1 on error */
int getpriority(int which, id_t who);

/* Returns: 0 if OK, −1 on error */
int setpriority(int which, id_t who, int value);
```

The `which` argument can take on one of three values: `PRIO_PROCESS` to indicate a process, `PRIO_PGRP` to indicate a process group, and `PRIO_USER` to indicate a user ID. The `which` argument controls how the `who` argument is interpreted and the `who` argument selects the process or processes of interest.

# Process Times

```c
#include <sys/times.h>

/* Returns: elapsed wall clock time in clock ticks if OK, −1 on error */
clock_t times(struct tms *buf);
```

This function fills in the `tms` structure pointed to by `buf`:

```c
struct tms {
    clock_t  tms_utime;  /* user CPU time */
    clock_t  tms_stime;  /* system CPU time */
    clock_t  tms_cutime; /* user CPU time, terminated children */
    clock_t  tms_cstime; /* system CPU time, terminated children */
};
```

Note that the structure does not contain any measurement for the wall clock time. Instead, the function returns the wall clock time as the value of the function, each time it’s called.

The `clock_t` values returned by this function can be converted to seconds using the number of clock ticks per second — the `_SC_CLK_TCK` value returned by `sysconf`.
