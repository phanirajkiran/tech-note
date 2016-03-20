File and Directories
====================

# `stat`, `fstat`, `fstatat` and `lstat` Functions

```c
#include <sys/stat.h>

/* All four return: 0 if OK, −1 on error */
int stat(const char *restrict pathname, struct stat *restrict buf );
int fstat(int fd, struct stat *buf);
int lstat(const char *restrict pathname, struct stat *restrict buf );
int fstatat(int fd, const char *restrict pathname, struct stat *restrict buf, int flag);
```

Given a `pathname`, the `stat` function returns a structure of information about the named file.

- **f**stat: opened file desciptor
- **f**stat**at**: relative path to an open directory
  - `flag=AT_SYMLINK_NOFOLLOW`: not follow symblic links
  - `pathname=AT_FDCWD`: relative to current directory
- **l**stat: returns information about the symbolic link, not the file referenced by the symbolic link

`struct stat` is defined as:

```c
struct stat {
  mode_t st_mode;   /* file type & mode (permissions) */
  ino_t st_ino;   /* i-node number (serial number) */
  dev_t st_dev;   /* device number (file system) */
  dev_t st_rdev;   /* device number for special files */
  nlink_t st_nlink;   /* number of links */
  uid_t st_uid;   /* user ID of owner */
  gid_t st_gid;   /* group ID of owner */
  off_t st_size;   /* size in bytes, for regular files */
  struct timespec st_atim;   /* time of last access */
  struct timespec st_mtim;   /* time of last modification */
  struct timespec st_ctim;   /* time of last file status change */
  blksize_t st_blksize;   /* best I/O block size */
  blkcnt_t st_blocks;   /* number of disk blocks allowed */
};
```

# File Types

- Regular file
- Directory file: contains the names of other files and pointers to information on these files
- Block special file: **buffered I/O** access in fixed-size units
- Character special file: **unbuffered I/O** access in variable-sized units
- FIFO
- Socket
- Symbolic link

The type of a file is encoded in the `st_mode` member of the `stat` structure. We can determine the file type with the macros defined in `sys/stats.h`:

- `S_ISREG()`: regular file
- `S_ISDIR()`: directory file
- `S_ISCHR()`: character special file
- `S_ISBLK()`: block special file
- `S_ISFIFO()`: pipe or FIFO
- `S_ISLNK()`: symbolic link
- `S_ISSOCK()`: socket
- `S_TYPEISMQ()`: message queue
- `S_TYPEISSEM()`: semaphore
- `S_TYPEISSHM()`: shared memory object

# User ID and Group ID

- real user ID, real group ID: who we really are (from password file)
- effective user ID, effective group ID: used for **file access permission checks**
- saved set-user-ID, saved set-group-ID: saved by `exec` functions

Normally, the effective user ID equals the real user ID, and the effective group ID equals the real group ID. However, we can also enable special flags (i.e. `S_ISUID` and `S_ISGID`) in the file’s mode word (`st_mode`) to set effective user ID of newly created process to be the owner of the file (`st_uid`).

# File Access Permissions

There are nine permission bits of `st_mode` be defined:

- `S_IRUSR`: user-read
- `S_IWUSR`: user-write
- `S_IXUSR`: user-execute
- `S_IRGRP`: group-read
- `S_IWGRP`: group-write
- `S_IXGRP`: group-execute
- `S_IROTH`: other-read
- `S_IWOTH`: other-write
- `S_IXOTH`: other-execute

Whenever we want to open any type of file by name, we must have **execute** permission in each directory mentioned in the name, including the current directory, if it is implied. This is why the execute permission bit for a directory is often called the *search bit*.

The file access tests that the kernel performs each time a process opens, creates, or deletes a file depend on the owners of the file (`st_uid` and `st_gid`), the **effective** IDs of the process (effective user ID and effective group ID), and the supplementary group IDs of the process, if supported.

# `access` and `faccessat` Functions

The `access` and `faccessat` functions base their tests on the **real** user and group IDs.

```c
#include <unistd.h>

/* Both return: 0 if OK, −1 on error */
int access(const char *pathname, int mode);
int faccessat(int fd, const char *pathname, int mode, int flag);
```

The mode is either the value `F_OK` to test if a file exists, or the bitwise OR of any of the following flags:

- `R_OK`: test for read permission
- `W_OK`: test for write permission
- `X_OK`: test for execute permission

# `umask` Function

The `umask` function sets the file mode creation mask for the process and returns the previous value.

```c
#include <sys/stat.h>

/* Returns: previous file mode creation mask */
mode_t umask(mode_t cmask);
```

The `cmask` argument is formed as the bitwise OR of any of the nine permission constants.

Some common `umask` values include:

- `002`: prevent others from writing your files
- `022`: prevent group members and others from writing your files
- `027`: prevent group members from writing your files and other from reading, writing, or executing your files

# `chmod`, `fchmod`, and `fchmodat` Functions

```c
#include <sys/stat.h>

/* All three return: 0 if OK, −1 on error */
int chmod(const char *pathname, mode_t mode);
int fchmod(int fd, mode_t mode);
int fchmodat(int fd, const char *pathname, mode_t mode, int flag);
```

To change the permission bits of a file, the effective user ID of the process must be equal to the owner ID of the file, or the process must have superuser permissions.

The `mode` is specified as the bitwise OR of the constants:

- `S_ISUID`: set-user-ID on execution
- `S_ISGID`: set-group-ID on execution
- `S_ISVTX`: saved-text (sticky bit)
- `S_IRWXU`: read, write, and execute by user (owner)
- `S_IRWXG`: read, write, and execute by group
- `S_IRWXO`: read, write, and execute by other (world)
- ...and other nine permission constants

## Sticky Bit (`S_ISVTX`)

On versions of the UNIX System that predated demand paging, this bit was known as the sticky bit. If it was set for an executable program file, then the first time the program was executed, a copy of the program’s text (i.e. machine instructions) was saved in the *swap area* when the process terminated. The program would then load into memory more quickly the next time it was executed.

On contemporary systems, the use of the sticky bit has been extended. The Single UNIX Specification allows the sticky bit to be set for a **directory**. If the bit is set for a directory, a file in the directory can be removed or renamed only if the user has write permission for the directory and meets one of the following criteria:

- Owns the file
- Owns the directory
- Is the superuser

# `chown`, `fchown`, `fchownat`, and `lchown` Functions

The `chown` functions allow us to change a file’s user ID and group ID, but if either of the arguments `owner` or `group` is −1, the corresponding ID is left unchanged.

```c
#include <unistd.h>

/* All four return: 0 if OK, −1 on error */
int chown(const char *pathname, uid_t owner, gid_t group);
int fchown(int fd, uid_t owner, gid_t group);
int fchownat(int fd, const char *pathname, uid_t owner, gid_t group, int flag);
int lchown(const char *pathname, uid_t owner, gid_t group);
```

If `_POSIX_CHOWN_RESTRICTED` defined in `unistd.h` is in effect for the specified file, you can’t change the user ID of your files. You can change the group ID of files that you own, but only to groups that you belong to.

# File Truncation

```c
#include <unistd.h>

/* Both return: 0 if OK, −1 on error */
int truncate(const char *pathname, off_t length); int ftruncate(int fd, off_t length);
```

- previous size > `length`: the data beyond `length` is no longer accessible
- previous size <= `length`: the file size will **increase** and the data between the old end of file and the new end of file will read as 0

# `link`, `linkat`, `unlink`, `unlinkat`, and `remove` Functions

```c
#include <unistd.h>

/* All return: 0 if OK, −1 on error */
int link(const char *existingpath, const char *newpath);
int linkat(int efd, const char *existingpath, int nfd, const char *newpath, int flag);
int unlink(const char *pathname);
int unlinkat(int fd, const char *pathname, int flag);
int remove(const char *pathname);
```

`link` create a new directory entry, `newpath`, that references the existing file `existingpath`. If the newpath already exists, an error is returned.

For a file, `remove` is identical to `unlink`. For a directory, `remove` is identical to `rmdir`.

# `rename` and `renameat` Functions

```c
#include <stdio.h>

/* Both return: 0 if OK, −1 on error */
int rename(const char *oldname, const char *newname);
int renameat(int oldfd, const char *oldname, int newfd, const char *newname);
```

If `newname` already exists, we need permissions as if we were **deleting** it. Also, because we’re removing the directory entry for `oldname` and possibly creating a directory entry for `newname`, we need write permission and execute permission in the directory containing `oldname` and in the directory containing `newname`.

# Creating and Reading Symbolic Links

```c
#include <unistd.h>

/* Both return: 0 if OK, −1 on error */
int symlink(const char *actualpath, const char *sympath);
int symlinkat(const char *actualpath, int fd, const char *sympath);
```

Because the `open` function follows a symbolic link, we need a way to open the link itself and read the name in thelink.

```c
#include <unistd.h>

/* Both return: number of bytes read if OK, −1 on error */
ssize_t readlink(const char* restrict pathname, char *restrict buf,
size_t bufsize);
ssize_t readlinkat(int fd, const char* restrict pathname,
char *restrict buf, size_t bufsize);
```

# File Times

- `st_atim`: last-access time of file data, e.g. `read`
- `st_mtim`: last-modification time of file data, e.g. `write`
- `st_ctim`: last-change time of **i-node** status, e.g. `chmod`, `chown`

Several functions are available to change the access time and the modification time of a file:

```c
#include <sys/stat.h>

/* Both return: 0 if OK, −1 on error */
int futimens(int fd, const struct timespec times[2]);
int utimensat(int fd, const char *path, const struct timespec times[2], int flag);
int utimes(const char *pathname, const struct timeval times[2]);
```

Timestamps can be specified in one of four ways:

- `time` is null pointer: set to current time
- `tv_nsec=UTIME_NOW`: set to current time
- `tv_nsec=UTIME_OMIT`: unchanged
- otherwise, set to the value specified by the corresponding `tv_sec` and `tv_nsec` fields

# `mkdir`, `mkdirat`, and `rmdir` Functions

```c
#include <sys/stat.h>

/* Both return: 0 if OK, −1 on error */
int mkdir(const char *pathname, mode_t mode);
int mkdirat(int fd, const char *pathname, mode_t mode);
int rmdir(const char *pathname);
```

A common mistake is to specify the same mode as for a file: read and write permissions only. But for a directory, we normally want at least one of the **execute bits** enabled, to allow access to filenames within the directory.

# Reading Directories

```c
#include <dirent.h>

/* Both return: pointer if OK, NULL on error */
DIR *opendir(const char *pathname);
DIR *fdopendir(int fd);

/* Returns: pointer if OK, NULL at end of directory or error */
struct dirent *readdir(DIR *dp);

/* Returns: 0 if OK, −1 on error */
void rewinddir(DIR *dp);
int closedir(DIR *dp);

/* Returns: current location in directory associated with dp */
long telldir(DIR *dp);

void seekdir(DIR *dp, long loc);
```

The `DIR` structure is an internal structure used by these seven functions to maintain information about the directory being read.

The `dirent` structure defined in `dirent.h` is implementation dependent. Implementations define the structure to contain at least the following two members:

```c
ino_t  d_ino;     /* i-node number */
char   d_name[];  /* null-terminated filename */
```

On Linux, it's defined as:

```c
struct dirent {
    ino_t          d_ino;       /* inode number */
    off_t          d_off;       /* offset to the next dirent */
    unsigned short d_reclen;    /* length of this record */
    unsigned char  d_type;      /* type of file; not supported
                                   by all file system types */
    char           d_name[256]; /* filename */
};
```

The `readdir()` function returns a pointer to a dirent structure representing the **next** directory entry in the directory stream pointed to by dirp. It returns NULL on reaching the end of the directory stream or if an error occurred.

```c
while ((dirp = readdir(dp)) != NULL) {
  /* ... */
}
```

# `chdir`, `fchdir`, and `getcwd` Functions

```c
#include <unistd.h>

/* Both return: 0 if OK, −1 on error */
int chdir(const char *pathname);
int fchdir(int fd);

/* Returns: buf if OK, NULL on error */
char *getcwd(char *buf, size_t size);
```
