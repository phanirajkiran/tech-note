Standard I/O Library
====================

# Standard Input, Standard Output, and Standard Error

Three streams are predefined and automatically available to a process: **standard input**, **standard output**, and **standard error**. These streams refer to the same files as the file descriptors `STDIN_FILENO`, `STDOUT_FILENO`, and `STDERR_FILENO`, respectively.

These three standard I/O streams are referenced through the predefined file pointers `stdin`, `stdout`, and `stderr`. The file pointers are defined in the `<stdio.h>` header.

# Buffering

The goal of the buffering provided by the standard I/O library is to use the minimum number of `read` and `write` calls.

- Fully buffered
  - Actual I/O takes place when the standard I/O buffer is filled.
  - The term **flush** describes the writing of a standard I/O buffer. A buffer can be flushed automatically by the standard I/O routines, such as when a buffer fills, or we can call the function `fflush` to flush a stream.
- Line buffered
  - Standard I/O library performs I/O when a **newline character** is encountered on input or output.
  - Typically used on a stream when it refers to a terminal - e.g. standard input and standard output.
- Unbuffered
  - The standard I/O library does not buffer the characters.

Most implementations default to the following types of buffering:

- Standard error is always unbuffered.
- All other streams are line buffered if they refer to a terminal device; otherwise, they are fully buffered.

We can call `setbuf` or `setvbuf` to change the default buffering type:

```c
#include <stdio.h>

/* Returns: 0 if OK, nonzero on error */
void setbuf(FILE *restrict fp, char *restrict buf );
int setvbuf(FILE *restrict fp, char *restrict buf, int mode, size_t size);
```

- `setbuf`
  - To enable buffering, `buf` must point to a buffer of length `BUFSIZ`, a constant defined in `<stdio.h>`
  - To disable buffering, we set `buf` to `NULL`.
  - Normally, the stream is fully buffered, but some systems may set line buffering if the stream is associated with a terminal device.
- `setvbuf`
  - We can set `mode` argument to specifty the exactly buffering type
    - `_IOFBF`: fully buffered
    - `_IOLBF`: line buffered
    - `_IONBF`: unbuffered
  - If the stream is buffered and buf is `NULL`, the standard I/O library will automatically allocate its own buffer of the appropriate size (i.e. `BUFSIZE`) for the stream.

At any time, we can force a stream to be flushed. The `fflush` function causes any unwritten data for the stream to be passed to the kernel. As a special case, if `fp` is `NULL`, `fflush` causes all output streams to be flushed.

```c
#include <stdio.h>

/* Returns: 0 if OK, EOF on error */
int fflush(FILE *fp);
```

# Opening a Stream

The `fopen`, `freopen`, and `fdopen` functions open a standard I/O stream.

```c
#include <stdio.h>

/* All three return: file pointer if OK, NULL on error */
FILE *fopen(const char *restrict pathname, const char *restrict type);
FILE *freopen(const char *restrict pathname, const char *restrict type, FILE *restrict fp);
FILE *fdopen(int fd, const char *type);
```

- `fopen`
  - Opens a specified file.
- `freopen`
  - Opens a specified file on a specified stream, closing the stream first if it is already open.
  - Typically used to open a specified file as one of the **predefined streams**: standard input, standard output, or standard error.
- `fdopen`
  - Takes an existing file descriptor.
  - Often used with descriptors that are returned by the functions that create pipes and network communication channels (cannot opened with `fopen`).

The `type` argument support the following modes:

![The type argument for opening a standard I/O stream](http://i.imgur.com/2syIBbz.png)

Note that if a new file is created by specifying a `type` of either `w` or `a`, we are not able to specify the file’s access permission bits, as we were able to do with the `open` function and the `creat` function.

Each standard I/O stream has an associated **file descriptor**, and we can obtain the descriptor for a stream by calling `fileno`:

```c
#include <stdio.h>

/* Returns: the file descriptor associated with the stream */
int fileno(FILE *fp);
```

An open stream is closed by calling `fclose`. Any buffered output data is flushed before the file is closed. Any input data that may be buffered is discarded. If the standard I/O library had automatically allocated a buffer for the stream, that buffer is released.

```c
#include <stdio.h>

/* Returns: 0 if OK, EOF on error */
int fclose(FILE *fp);
```

# Reading and Writing a Stream

## Character-at-a-time I/O

### Input Functions

Three functions allow us to read **one character** at a time, and thet return the next character as an `unsigned char` converted to an `int`:

```c
#include <stdio.h>

/* All three return: next character if OK, EOF on end of file or error */
int getc(FILE *fp);
int fgetc(FILE *fp);
int getchar(void);
```

The function `getchar` is defined to be equivalent to `getc(stdin)`. The difference between `getc` and `fgetc` is that `getc` can be implemented as a macro, whereas `fgetc` cannot be implemented as a macro.

Note that these functions return the same value whether an error occurs or the end of file is reached. To distinguish between the two, we must call either `ferror` or `feof`:

```c
#include <stdio.h>

/* Both return: nonzero (true) if condition is true, 0 (false) otherwise */
int ferror(FILE *fp);
int feof(FILE *fp);
```

After reading from a stream, we can push back characters by calling `ungetc` (e.g. after peek):

```c
#include <stdio.h>

/* Returns: c if OK, EOF on error */
int ungetc(int c, FILE *fp);
```

The character that we push back does not have to be the same character that was read. Although ISO C allows an implementation to support any amount of pushback, an implementation is required to provide only a **single** character of pushback. We should not count on more than a single character.

### Output Functions

Output functions are available that correspond to each of the input functions:

```c
#include <stdio.h>

/* All three return: c if OK, EOF on error */
int putc(int c, FILE *fp);
int fputc(int c, FILE *fp);
int putchar(int c);
```

As with the input functions, `putchar(c)` is equivalent to `putc(c, stdout)`, and `putc` can be implemented as a macro, whereas `fputc` cannot be implemented as a macro.

## Line-at-a-Time I/O

### Input Functions

Line-at-a-time input is provided by the two functions, `fgets` and `gets`:

```c
#include <stdio.h>

/* Both return: buf if OK, NULL on end of file or error */
char *fgets(char *restrict buf, int n, FILE *restrict fp);
char *gets(char *buf);
```

Both specify the address of the buffer to read the line into. The `gets` function reads from standard input, whereas `fgets` reads from the specified stream.

With `fgets`, we have to specify the size of the buffer, `n`. This function reads up through and including the next newline, but no more than `n−1` characters, into the buffer. The buffer is terminated with a null byte.

**IMPORTANT:** The `gets` function should never be used. The problem is that it doesn’t allow the caller to specify the buffer size. This allows the buffer to **overflow** if the line is longer than the buffer, writing over whatever happens to follow the buffer in memory.

### Output Functions

Line-at-a-time output is provided by `fputs` and `puts`:

```c
#include <stdio.h>

/* Both return: non-negative value if OK, EOF on error */
int fputs(const char *restrict str, FILE *restrict fp);
int puts(const char *str);
```

The `puts` function writes the null-terminated string to the standard output, without writing the null byte. But `puts` then writes a **newline character** to the standard output.

Note that this need not be line-at-a-time output, since the string need not contain a newline as the last non-null character.

## Binary I/O

Binary I/O, or direct I/O, is often used for binary files where we read or write a structure with each operation.

```c
#include <stdio.h>

/* Both return: number of objects read or written */
size_t fread(void *restrict ptr, size_t size, size_t nobj, FILE *restrict fp);
size_t fwrite(const void *restrict ptr, size_t size, size_t nobj, FILE *restrict fp);
```

A fundamental problem with binary I/O is that it can be used to read only data that has been written on the **same system** due to per-compiler, per-system offset of a member within a structure, and the binary formats.

# Positioning a Stream

## `ftell` and `fseek` Functions

The following functions assume that a file’s position can be stored in a `long` integer.

```c
#include <stdio.h>

/* Returns: current file position indicator if OK, −1L on error */
long ftell(FILE *fp);

/* Returns: 0 if OK, −1 on error */
int fseek(FILE *fp, long offset, int whence);

void rewind(FILE *fp);
```

For a binary file, a file’s position indicator is measured in **bytes** from the beginning of the file. The value returned by `ftell` for a binary file is this byte position. To position a binary file using `fseek`, we must specify a byte offset and indicate how that offset is interpreted.

For the values of `whence`, it indicates how that offset is interpreted:

- `SEEK_SET`: from the beginning of the file
- `SEEK_CUR`: from the current file position
- `SEEK_END`: from the end of file

## `ftello` and `fseeko` Functions

The `ftello` function is the same as `ftell`, and the `fseeko` function is the same as `fseek`, except that the type of the offset is `off_t` instead of `long`.

```c
#include <stdio.h>

/* Returns: current file position indicator if OK, (off_t)−1 on error */
off_t ftello(FILE *fp);

/* Returns: 0 if OK, −1 on error */
int fseeko(FILE *fp, off_t offset, int whence);
```

## `fgetpos` and `fsetpos` Functions

The `fgetpos` and `fsetpos` functions were introduced by the ISO C standard. They are recommended to be used When porting applications to non-UNIX systems.

```c
#include <stdio.h>

/* Both return: 0 if OK, nonzero on error */
int fgetpos(FILE *restrict fp, fpos_t *restrict pos);
int fsetpos(FILE *fp, const fpos_t *pos);
```

# Formatted I/O

## Formatted Output

Formatted output is handled by the five `printf` functions:

```c
#include <stdio.h>

/* All three return: number of characters output if OK, negative value if output error */
int printf(const char *restrict format, ...);
int fprintf(FILE *restrict fp, const char *restrict format, ...);
int dprintf(int fd, const char *restrict format, ...);

/* Returns: number of characters stored in array if OK, negative value if encoding error */
int sprintf(char *restrict buf, const char *restrict format, ...);

/* Returns: number of characters that would have been stored in array if buffer was large enough, negative value if encoding error */
int snprintf(char *restrict buf, size_t n, const char *restrict format, ...);
```

The `printf` function writes to the standard output, `fprintf` writes to the specified stream, `dprintf` writes to the specified file descriptor, and `sprintf` places the formatted characters in the array `buf`. The `sprintf` function automatically appends a **null** byte at the end of the array, but this null byte is not included in the return value.

A format conversion specification has four optional components, shown in square brackets below:

```
 %[flags][fldwidth][precision][lenmodifier]convtype
```

- `flags`
  - ![The flags component of a conversion specification](http://i.imgur.com/8U6Yl4C.png)
- `fldwidth`
  - Specifiy a minimum field width for the conversion.
  - If the conversion results in fewer characters, it is padded with spaces.
- `precision`
  - Specify the minimum number of digits to appear for integer conversions, the minimum number of digits to appear to the right of the decimal point for floating-point conversions, or the maximum number of bytes for string conversions.
  - A period (`.`) followed by a optional non-negative decimal integer or an asterisk (`*`).
- `lenmodifier`
  - Specify the size of the argument.
  - ![The length modifier component of a conversion specification](http://i.imgur.com/L5Iohxu.png)
- `convtype` **(required)**
  - Control how the argument is interpreted.
  - ![The conversion type component of a conversion specification](http://i.imgur.com/frJ8joL.png)

The following five variants of the `printf` family are similar to the previous five, but the variable argument list (`...`) is replaced with `arg`.

```c
#include <stdarg.h>
#include <stdio.h>

/* All three return: number of characters output if OK, negative value if output error */
int vprintf(const char *restrict format, va_list arg);
int vfprintf(FILE *restrict fp, const char *restrict format,
va_list arg);
int vdprintf(int fd, const char *restrict format, va_list arg);

/* Returns: number of characters stored in array if OK, negative value if encoding error */
int vsprintf(char *restrict buf, const char *restrict format, va_list arg);

/* Returns: number of characters that would have been stored in array if buffer was large enough, negative value if encoding error */
int vsnprintf(char *restrict buf, size_t n,
const char *restrict format, va_list arg);
```

## Formatted Input

Formatted input is handled by the three `scanf` functions:

```c
#include <stdio.h>

/* All three return: number of input items assigned, EOF if input error or end of file before any conversion */
int scanf(const char *restrict format, ...);
int fscanf(FILE *restrict fp, const char *restrict format, ...);
int sscanf(const char *restrict buf, const char *restrict format, ...);
```

The `scanf` family is used to parse an input string and convert character sequences into variables of specified types.

There are three optional components to a format conversion specification, shown in square brackets below:

```
%[*][fldwidth][m][lenmodifier]convtype
```

- The optional leading asterisk (`*`)
  - Used to suppress conversion.
  - Input is converted as specified by the rest of the conversion specification, but the result is not stored in an argument.
- `fldwidth`
  - Specify the maximum field width in characters.
- `lenmodifier`
  - Specify the size of the argument to be initialized with the result of the conversion.
  - Same as `printf` family.
- The optional `m` character
  - *assignment-allocation character*
  - It can be used with the `%c`, `%s`, and `%[` conversion specifiers to force a memory buffer to be allocated to hold the converted string; the caller is responsible for freeing the buffer by calling the free function when the buffer is no longer needed.
- `convtype`
  - ![The conversion type component of a conversion specification](http://i.imgur.com/5Q41LmR.png)

Like the `printf` family, the `scanf` family supports functions that use variable argument lists as specified by `<stdarg.h>`:

```c
#include <stdarg.h>
#include <stdio.h>

/* All three return: number of input items assigned, EOF if input error or end of file before any conversion */
int vscanf(const char *restrict format, va_list arg);
int vfscanf(FILE *restrict fp, const char *restrict format,
va_list arg);
int vsscanf(const char *restrict buf, const char *restrict format, va_list arg);
```

# Temporary Files

## `tmpnam` and `tmpfile` Functions

The ISO C standard defines two functions that are provided by the standard I/O library to assist in creating temporary files:

```c
#include <stdio.h>

/* Returns: pointer to unique pathname */
char *tmpnam(char *ptr);

/* Returns: file pointer if OK, NULL on error */
FILE *tmpfile(void);
```

The `tmpnam` function generates a string that is a valid pathname and that does not match the name of any existing file. If `ptr` is `NULL`, the generated pathname is stored in a **static area**, and a pointer to this area is returned as the value of the function.

A big drawback is that a **window** exists between the time that the unique pathname is returned and the time that an application creates a file with that name. During this timing window, another process can create a file of the same name.

The `tmpfile` function creates a temporary binary file (type `wb+`) that is automatically removed when it is closed or on program termination. The standard technique often used by the `tmpfile` function is to create a unique pathname by calling tmpnam, then create the file, and immediately **unlink** it.

## `mkdtemp` and `mkstemp` Functions

The Single UNIX Specification defines two additional functions as part of the XSI option for dealing with temporary files: `mkdtemp` and `mkstemp`:

```c
#include <stdlib.h>

/* Returns: pointer to directory name if OK, NULL on error */
char *mkdtemp(char *template);

/* Returns: file descriptor if OK, −1 on error */
int mkstemp(char *template);
```

The `mkdtemp` function creates a directory with a unique name, and the `mkstemp` function creates a regular file with a unique name. The name is selected using the **template string**. This string is a pathname whose last six characters are set to XXXXXX. The function replaces these placeholders with different characters to create a unique pathname. If successful, these functions modify the template string to reflect the name of the temporary file.

```c
char good_template[] = "/tmp/dirXXXXXX";   /* right way */

char *bad_template = "/tmp/dirXXXXXX";   /* wrong way*/
/* Only the memory for the pointer itself resides on the stack; the compiler arranges for the string to be stored in the read-only segment of the executable. When the mkstemp function tries to modify the string, a segmentation fault occurs. */
```

Unlike `tmpfile`, the temporary file created by mkstemp is not removed automatically for us. If we want to remove it from the file system namespace, we need to unlink it ourselves.

# Memory Streams

In Version 4, the Single UNIX Specification added support for *memory streams*. These are standard I/O streams for which there are no underlying files, although they are still accessed with `FILE` pointers. All I/O is done by transferring bytes to and from buffers in **main memory**.

```c
#include <stdio.h>

/* Returns: stream pointer if OK, NULL on error */
FILE *fmemopen(void *restrict buf, size_t size, const char *restrict type);
```
