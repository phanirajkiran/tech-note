File I/O
========

# File Descriptors

To the kernel, all open files are referred to by file descriptors. A file descriptor is a non-negative integer. By convention, UNIX System shells associate file descriptor 0 with the standard input of a process, file descriptor 1 with the standard output, and file descriptor 2 with the standard error.

Although their values are standardized by POSIX.1, the magic numbers 0, 1, and 2 should be replaced in POSIX-compliant applications with the symbolic constants `STDIN_FILENO`, `STDOUT_FILENO`, and `STDERR_FILENO` to improve readability. These constants are defined in the `<unistd.h>` header.

# `open` and `openat` Functions

```c
#include <fcntl.h>

/* Both return: file descriptor if OK, −1 on error */
int open(const char *path, int oflag, ... /* mode_t mode */ );
int openat(int fd, const char *path, int oflag, ... /* mode_t mode */ );
```

This argument is formed by **OR** together one or more of the following constants from the `<fcntl.h>` header:

- `O_RDONLY`: Open for reading only
- `O_WRONLY`: Open for writing only
- `O_RDWR`: Open for reading and writing
- `O_EXEC`: Open for execute only
- `O_APPEND`: Append to the end of file on each write
- `O_CREAT`: Create the file if it doesn’t exist
  - This option requires a third argument to the `open` function — the `mode`, which specifies the access permission bits of the new file.
- `O_NONBLOCK`: Set the nonblocking mode for FIFO, block special file, or character special file
- `O_SYNC`: Have each `write` wait for **physical I/O** to complete
- `O_TRUNC`: If the file exists and if it is successfully opened for either write-only or read–write, truncate its length to 0
- ...and more

# `creat` Function

A new file can also be created by calling the `creat` function.

```c
#include <fcntl.h>

/* Returns: file descriptor opened for write-only if OK, −1 on error */
int creat(const char *path, mode_t mode);
```

Note that this function is equivalent to `open(path, O_WRONLY | O_CREAT | O_TRUNC, mode)`

# `close` Function

An open file is closed by calling the `close` function.

```c
#include <unistd.h>

/* Returns: 0 if OK, −1 on error */
int close(int fd);
```

Closing a file also releases any record locks that the process may have on the file.

# `lseek` Function

Every open file has an associated "current file offset", normally a non-negative integer that measures the number of bytes from the beginning of the file (certain devices could allow negative offsets). By default, this offset is initialized to 0 when a file is opened, unless the `O_APPEND` option is specified.

```c
#include <unistd.h>

/* Returns: new file offset if OK, −1 on error */
off_t lseek(int fd, off_t offset, int whence);
```

The interpretation of the `offset` depends on the value of the `whence` argument:

- `SEEK_SET`: set to *offset* bytes from the beginning of the file
- `SEEK_CUR`: set to its current value plus the *offset*
- `SEEK_END`: set to the size of the file plus the *offset*

We can seek zero bytes from the current position to determine the current offset:

```c
off_t currpos;
currpos = lseek(fd, 0, SEEK_CUR);
```

The file’s offset can be greater than the file’s current size, in which case the next `write` to the file will extend the file. This is referred to as creating a **hole** in a file and is allowed. Any bytes in a file that have not been written are read back as 0 (can be checked by `od(1)` command).

# `read` Function

```c
#include <unistd.h>

/* Returns: number of bytes read, 0 if end of file, −1 on error */
ssize_t read(int fd, void *buf, size_t nbytes);
```

If the `read` is successful, the **number of bytes read** is returned. If the end of file is encountered, 0 is returned. The `read` operation starts at the file’s current offset. Before a successful return, the offset is incremented by the number of bytes actually read.

# `write` Function

```c
#include <unistd.h>

/* Returns: number of bytes written if OK, −1 on error */
ssize_t write(int fd, const void *buf, size_t nbytes);
```

The return value is usually equal to the *nbytes* argument; otherwise, an error has occurred. A common cause for a write error is either filling up a disk or exceeding the file size limit for a given process.

For a regular file, the `write` operation starts at the file’s current offset. If the `O_APPEND` option was specified when the file was opened, the file’s offset is set to the current end of file before each write operation. After a successful write, the file’s offset is incremented by the number of bytes actually written.

# File Sharing

The kernel uses three data structures to represent an open file, and the relationships among them determine the effect one process has on another with regard to file sharing.

1. Every process has an entry in the process table. Within each process table entry is a table of open file descriptors. Associated with each file descriptor are:
  - The file descriptor flags
  - A pointer to a file table entry
2. The kernel maintains a file table for all open files. Each file table entry contains:
  - The file status flags for the file, such as read, write, append, sync, and nonblocking
  - The current file offset
  - A pointer to the v-node table entry for the file
3. Each open file (or device) has a **v-node** structure that contains information about the type of file and pointers to functions that operate on the file. For most files, the v-node also contains the **i-node** for the file.

![Kernel data structures for open files](http://i.imgur.com/cCBnqFG.png)

If two independent processes have the same file open, we could have the following arrangement:

![Two independent processes with the same file open](http://i.imgur.com/Bx8dbWM.png)

# Atomic Operations

## Appending to a File

Consider a single process that wants to append to the end of a file. Older versions of the UNIX System didn’t support the `O_APPEND` option to open, so the program was coded as follows:

```c
if (lseek(fd, 0L, 2) < 0)         /* position to EOF */
  err_sys("lseek error");
if (write(fd, buf, 100) != 100)   /* and write */
  err_sys("write error");
```

This works fine for a single process, but problems arise if multiple processes use this technique to append to the same file (because of the context switch occured between two operations).

The UNIX System provides an **atomic** way to do this operation if we set the `O_APPEND` flag when a file is opened. This causes the kernel to position the file to its current end of file before each write, without need to call `lseek` before each `write`.

## `pread` and `pwrite` Functions

The Single UNIX Specification includes two functions that allow applications to seek and perform I/O atomically: `pread` and `pwrite`.

```c
#include <unistd.h>

/* Returns: number of bytes read, 0 if end of file, −1 on error */
ssize_t pread(int fd, void *buf, size_t nbytes, off_t offset);

/* Returns: number of bytes written if OK, −1 on error */
ssize_t pwrite(int fd, const void *buf, size_t nbytes, off_t offset);
```

Calling pread is equivalent to calling `lseek` followed by a call to `read`, with the following exceptions:

- There is no way to interrupt the two operations that occur when we call `pread` => **atomic**
- The current file offset is not updated

# `dup` and `dup2` Functions

An existing file descriptor is duplicated by either of the following functions:

```c
#include <unistd.h>

/* Both return: new file descriptor if OK, −1 on error */
int dup(int fd);
int dup2(int fd, int fd2);
```

The new file descriptor returned by `dup` is guaranteed to be the lowest-numbered available file descriptor. With `dup2`, we specify the value of the new descriptor with the `fd2` argument. If `fd2` is already open, it is first closed. If `fd` equals `fd2`, then `dup2` returns `fd2` without closing it. Otherwise, the `FD_CLOEXEC` file descriptor flag is cleared for `fd2`, so that `fd2` is left open if the process calls exec.

The new file descriptor that is returned as the value of the functions shares the same file table entry as the `fd` argument.

![Kernel data structures after dup(1)](http://i.imgur.com/LQedZAe.png)

Another way to duplicate a descriptor is with the `fcntl` function:

- `dup(fd)` is equivalent to `fcntl(fd, F_DUPFD, 0)`
- `dup2(fd, fd2)` is equivalent to `close(fd2); fcntl(fd, F_DUPFD, fd2)`
  - not exactly the same => `dup2` is **atomic**

# `sync`, `fsync`, and `fdatasync` Functions

Traditional implementations of the UNIX System have a **buffer cache** or **page cache** in the kernel through which most disk I/O passes. When we write data to a file, the data is normally copied by the kernel into one of its buffers and queued for writing to disk at some later time. This is called *delayed write*.

The kernel eventually writes all the delayed-write blocks to disk, normally when it needs to reuse the buffer for some other disk block. To ensure consistency of the file system on disk with the contents of the buffer cache, the `sync`, `fsync`, and `fdatasync` functions are provided.

```c
#include <unistd.h>

void sync(void);

/* Returns: 0 if OK, −1 on error */
int fsync(int fd);
int fdatasync(int fd);
```

- `sync`
  - queue all the modified block buffers for writing and returns, without waiting for the disk writes to take place.
  - normally called periodically (usually every 30 seconds) from a system daemon, often called *update*.
- `fsync`
  - refer only to a single file pointed by `fd`
  - wait for the disk writes to complete before returning
  - file’s attributes are also updated synchronously
- `fdatasync`
  - similar to `fsync`, but affect only the data portions of a file.

# `fcntl` Function

The `fcntl` function can change the properties of a file that is already open.

```c
#include <fcntl.h>

/* Returns: depends on cmd if OK (see following), −1 on error */
int fcntl(int fd, int cmd, ... /* int arg */ );
```

The fcntl function is used for five different purposes.

1. Duplicate an existing descriptor (cmd = `F_DUPFD` or `F_DUPFD_CLOEXEC`)
2. Get/set file descriptor flags (cmd = `F_GETFD` or `F_SETFD`)
  - Currently, only one file descriptor flag is defined: the `FD_CLOEXEC` flag.
3. Get/set file status flags (cmd = `F_GETFL` or `F_SETFL`)
  - The five access-mode flags — `O_RDONLY`, `O_WRONLY`, `O_RDWR`, `O_EXEC`, and `O_SEARCH` — are not separate bits that can be tested => first use the `O_ACCMODE` mask to obtain the access-mode bits
4. Get/set asynchronous I/O ownership (cmd = `F_GETOWN` or `F_SETOWN`)
5. Get/set record locks (cmd = `F_GETLK`, `F_SETLK`, or `F_SETLKW`)

When we modify either the file descriptor flags or the file status flags, we must be careful to fetch the existing flag value, modify it as desired, and then set the new flag value. We can’t simply issue an `F_SETFD` or an `F_SETFL` command, as this could turn off flag bits that were previously set.

```c
void
set_fl(int fd, int flags) /* flags are file status flags to turn on */
{
  int val;
  if ((val = fcntl(fd, F_GETFL, 0)) < 0)
    err_sys("fcntl F_GETFL error");

  val |= flags;       /* turn on flags */
  /* val &= ~flags for clr_fl(fd, flags) */

  if (fcntl(fd, F_SETFL, val) < 0)
    err_sys("fcntl F_SETFL error");
}
```

# `ioctl` Function

Anything that couldn’t be expressed using one of the other functions in this chapter usually ended up being specified with an `ioctl`. Terminal I/O was the biggest user of this function.

```c
#include <sys/ioctl.h>

/* Returns: −1 on error, something else if OK */
int ioctl(int fd, int request, ...);
```

Each device driver can define its own set of `ioctl` commands.

# `/dev/fd`

Newer systems provide a directory named `/dev/fd` whose entries are files named 0, 1, 2, and so on. Opening the file `/dev/fd/n` is equivalent to duplicating descriptor n, assuming that descriptor n is open.

For example, `fd = open("/dev/fd/0", mode)` is equivalent to `fd = dup(0)`.

The main use of the `/dev/fd` files is from the shell. It allows programs that use pathname arguments to handle standard input and standard output in the same manner as other pathnames. For example, ` filter file2 | cat file1 /dev/fd/0 file3 | lpr`.
