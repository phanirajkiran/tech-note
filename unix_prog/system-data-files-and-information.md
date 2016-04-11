System Data Files and Information
=================================

# Password File

The UNIX System’s password file, called the user database by POSIX.1, contains the fields shown in the following table. These fields are contained in a `passwd` structure that is defined in `<pwd.h>`:

![Fields in /etc/passwd file](http://i.imgur.com/WZpqLsq.png)

Historically, the password file has been stored in `/etc/passwd` and has been an ASCII file. For example,

```
root:x:0:0:root:/root:/bin/bash
squid:x:23:23::/var/spool/squid:/dev/null
nobody:x:65534:65534:Nobody:/home:/bin/sh
sar:x:205:105:Stephen Rago:/home/sar:/bin/bash
```

- The encrypted password field contains a single character as a **placeholder** where older versions of the UNIX System used to store the encrypted password. Because it is a security hole to store the encrypted password in a file that is readable by everyone, encrypted passwords are now kept elsewhere.
- The shell field contains the name of the executable program to be used as the login shell for the user. The default value for an empty shell field is usually `/bin/sh`. However, some user has `/dev/null` as the login shell, and its use is to prevent anyone from logging in to the system. There are several alternatives including `/bin/false` and `/bin/true`.
- Some systems that provide the `finger(1)` command support additional information in the comment field.

POSIX.1 defines two functions to fetch entries from the password file:

```c
#include <pwd.h>

/* Both return: pointer if OK, NULL on error */
struct passwd *getpwuid(uid_t uid);
struct passwd *getpwnam(const char *name);
```

The `getpwuid` function is used by the `ls(1)` program to map the numerical user ID contained in an i-node into a user’s login name. The `getpwnam` function is used by the `login(1)` program when we enter our login name.

**IMPORTANT:** Both functions return a pointer to a `passwd` structure that the functions fill in. This structure is usually a **static** variable within the function, so its contents are overwritten each time we call either of these functions.

Another three functions can be used to go through the entire password file:

```c
#include <pwd.h>

/* Returns: pointer if OK, NULL on error or end of file */
struct passwd *getpwent(void);

void setpwent(void);
void endpwent(void);
```

The following snippet shows the implementation of function `getpwnam`:

```c
struct passwd *
getpwnam(const char *name)
{
   struct passwd *ptr;
   setpwent();
   while ((ptr = getpwent()) != NULL)
       if (strcmp(name, ptr->pw_name) == 0)
           break;   /* found a match */
   endpwent();
   return(ptr);   /* ptr is NULL if no match found */
}
```

# Shadow Passwords

The encrypted password is a copy of the user’s password that has been put through a **one-way** encryption algorithm. Because this algorithm is one-way, we can’t guess the original password from the encrypted version.

To make it more difficult to obtain the raw materials (the encrypted passwords), systems now store the encrypted password in another file, often called the *shadow password file*. Minimally, this file has to contain the user name and the encrypted password:

![Fields in /etc/shadow file](http://i.imgur.com/5iW2j2z.png)

On Linux, a separate set of functions is available to access the shadow password file:

```c
#include <shadow.h>

/* Both return: pointer if OK, NULL on error */
struct spwd *getspnam(const char *name);
struct spwd *getspent(void);

void setspent(void);
void endspent(void);
```

# Group File

The UNIX System’s group file, called the group database by POSIX.1, contains the fields shown in the following table. These fields are contained in a `group` structure that is defined in `<grp.h>`:

![Fields in /etc/group file](http://i.imgur.com/hkd0NAB.png)

The field `gr_mem` is an array of pointers to the user names that belong to this group. This array is terminated by a null pointer.

We can look up either a group name or a numerical group ID with the following two functions, which are defined by POSIX.1:

```c
#include <grp.h>

/* Both return: pointer if OK, NULL on error */
struct group *getgrgid(gid_t gid);
struct group *getgrnam(const char *name);
```

Like the password file functions, both of these functions normally return pointers to a *static* variable, which is overwritten on each call. If we want to search the entire group file, we need some additional functions:

```c
#include <grp.h>

/* Returns: pointer if OK, NULL on error or end of file */
struct group *getgrent(void);

void setgrent(void);
void endgrent(void);
```

# Supplementary Group IDs

With supplementary group IDs, not only did we belong to the group corresponding to the group ID in our password file entry, but we could also belong to as many as 16 additional groups. The advantage of using supplementary group IDs is that we no longer have to change groups explicitly.

Three functions are provided to fetch and set the supplementary group IDs:

```c
#include <unistd.h>

/* Returns: number of supplementary group IDs if OK, −1 on error */
int getgroups(int gidsetsize, gid_t grouplist[]);

#include <grp.h> /* on Linux */
#include <unistd.h> /* on FreeBSD, Mac OS X, and Solaris */

/* Returns: 0 if OK, −1 on error */
int setgroups(int ngroups, const gid_t grouplist[]);

#include <grp.h> /* on Linux and Solaris */
#include <unistd.h> /* on FreeBSD and Mac OS X */

/* Returns: 0 if OK, −1 on error */
int initgroups(const char *username, gid_t basegid);
```

- `getgroups`
  - Fills in the array grouplist with the supplementary group IDs.
  - As a special case, if gidsetsize is 0, the function returns only the number of supplementary group IDs. The array grouplist is not modified.
- `setgroups`
  - Called by the superuser to set the supplementary group ID list for the calling process.
  - Usually called from the `initgroups` function to initialize the supplementary group ID list for the user.

# Other Data Files

Except the password file and the group file, there are also numerous other files are used by UNIX systems in normal day-to-day operation, including `/etc/services`, `/etc/protocols`, and `/etc/networks`.

The general principle is that every data file has at least three functions:

- `get` function: Reads the next record, opening the file if necessary. Notice that most of the `get` functions return a pointer to a **static** structure.
- `set` function: Opens the file, if not already open, and rewinds the file.
- `end` function: Closes the data file.

![Similar routines for accessing system data files](http://i.imgur.com/QvVs2Sz.png)

# Login Accounting

Two data files provided with most UNIX systems are the `utmp` file, which keeps track of all the users **currently** logged in, and the `wtmp` file, which keeps track of all logins and logouts.

```c
struct utmp {
    char  ut_line[8]; /* tty line: "ttyh0", "ttyd0", "ttyp0", ... */
    char  ut_name[8]; /* login name */
    long  ut_time;    /* seconds since Epoch */
};
```

- On login, one of these structures was filled in and written to the `utmp` file by the login program, and the same structure was appended to the `wtmp` file.
- On logout, the entry in the `utmp` file was erased — filled with null bytes — by the init process, and a new entry was appended to the `wtmp` file. This logout entry in the `wtmp` file had the `ut_name` field zeroed out.
- Special entries were appended to the `wtmp` file to indicate when the system was rebooted and right before and after the system’s time and date was changed.

# System Identification

POSIX.1 defines the `uname` function to return information on the current host and operating system:

```c
#include <sys/utsname.h>

/* Returns: non-negative value if OK, −1 on error */
int uname(struct utsname *name);
```

We pass the address of a `utsname` structure to this function, and the function then fills it in. POSIX.1 defines only the minimum fields in the structure:

```c
struct utsname {
    char  sysname[];
    char  nodename[];
    char  release[];
    char  version[];
    char  machine[];
};
```

Historically, BSD-derived systems provided the `gethostname` function to return only the name of the host. This name is usually the name of the host on a TCP/IP network:

```c
#include <unistd.h>

/* Returns: 0 if OK, −1 on error */
int gethostname(char *name, int namelen);
```

# Time and Date Routines

![Relationship of the various time functions](http://i.imgur.com/FAE2qmP.png)

Notice that the functions with dashed lines were affected by the `TZ` (timezone) environment variable.

## Calendar time

The basic time service provided by the UNIX kernel counts the number of seconds that have passed since the Epoch: **00:00:00 January 1, 1970**, Coordinated Universal Time (UTC). These seconds are represented in a `time_t` data type, and we call them *calendar times*.

The `time` function returns the current time and date:

```c
#include <time.h>

/* Returns: value of time if OK, −1 on error */
time_t time(time_t *calptr);
```

Version 4 of the Single UNIX Specification specifies that the `gettimeofday` function is now obsolescent. However, a lot of programs still use it, because it provides greater resolution (**up to a microsecond**) than the time function.

```c
#include <sys/time.h>

/* Returns: 0 always */
int gettimeofday(struct timeval *restrict tp, void *restrict tzp);
```

The only legal value for `tzp` is **NULL**; other values result in unspecified behavior. Some platforms support the specification of a time zone through the use of `tzp`, but this is implementation specific and not defined by the Single UNIX Specification.

## Broken-down time

Once we have the integer value that counts the number of seconds since the Epoch, we normally call a function to convert it to a broken-down time structure, and then call another function to generate a human-readable time and date.

The two functions `localtime` and `gmtime` convert a calendar time into what’s called a broken-down time, a `tm` structure:

```c
struct tm {
    int  tm_sec;   /* seconds after the minute: [0 - 60], 60 for leap second */
    int  tm_min;   /* minutes after the hour: [0 - 59] */
    int  tm_hour;  /* hours after midnight: [0 - 23] */
    int  tm_mday;  /* day of the month: [1 - 31] */
    int  tm_mon;   /* months since January: [0 - 11] */
    int  tm_year;  /* years since 1900 */
    int  tm_wday;  /* days since Sunday: [0 - 6] */
    int  tm_yday;  /* days since January 1: [0 - 365] */
    int  tm_isdst; /* daylight saving time flag: <0, 0, >0 */
};
```

```c
#include <time.h>

/* Both return: pointer to broken-down time, NULL on error */
struct tm *gmtime(const time_t *calptr);
struct tm *localtime(const time_t *calptr);
```

The difference between `localtime` and `gmtime` is that the first converts the calendar time to the local time, taking into account the **local time zone** and **daylight saving time** flag, whereas the latter converts the calendar time into a broken-down time expressed as **UTC**.

The function `mktime` takes a broken-down time, expressed as a local time, and converts it into a `time_t` value:

```c
#include <time.h>

/* Returns: calendar time if OK, −1 on error */
time_t mktime(struct tm *tmptr);
```

## Time formatting

The `strftime` function is a printf-like function for time values:

```c
#include <time.h>

/* Both return: number of characters stored in array if room, 0 otherwise */
size_t strftime(char *restrict buf, size_t maxsize,
                const char *restrict format,
                const struct tm *restrict tmptr);
size_t strftime_l(char *restrict buf, size_t maxsize,
                const char *restrict format,
                const struct tm *restrict tmptr, locale_t locale);
```

The format argument controls the formatting of the time value. Like the `printf` functions, conversion specifiers are given as a percent sign followed by a special character:

![Conversion specifiers for strftime](http://i.imgur.com/tnPdvft.png)

The `strptime` function is the inverse of strftime. It takes a string and converts it into a broken-down time:

```c
#include <time.h>

/* Returns: pointer to one character past last character parsed, NULL otherwise */
char *strptime(const char *restrict buf, const char *restrict format, struct tm *restrict tmptr);
```

The format specification is similar, although it differs slightly from the specification for the `strftime` function:

![Conversion specifiers for strptime](http://i.imgur.com/56WKZfg.png)
