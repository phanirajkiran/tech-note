Process Relationships
=====================

# Terminal Logins

The BSD terminal login procedure has not changed much over the past 35 years.

1. The system administrator creates a file, usually `/etc/ttys`, that has one line per terminal device. Each line specifies the name of the device and other parameters that are passed to the `getty` program.
2. When the system is bootstrapped, the kernel creates process ID 1, the `init` process, and it is `init` that brings the system up in multiuser mode.
3. The `init` process reads the file `/etc/ttys` and, for every terminal device that allows a `login`, does a fork followed by an `exec` of the program `getty`.
4. `getty` calls `open` for the terminal device. The terminal is opened for reading and writing. Once the device is open, file descriptors 0, 1, and 2 are set to the device.
5. `getty` outputs something like `login:` and waits for us to enter our user name.
6. Invokes the `login` program: `execle("/bin/login", "login", "-p", username, (char *)0, envp);`

![State of processes after login has been invoked](http://i.imgur.com/bFhHjAm.png)

The `login` program does many things. Since it has our user name, it can call `getpwnam` to fetch our password file entry. Then `login` calls `getpass(3)` to display the prompt `Password:` and read our password. It calls `crypt(3)` to encrypt the password that we entered and compares the encrypted result to the `pw_passwd` field from our shadow password file entry.

If we log in correctly, `login` will:

- Change to our home directory (`chdir`)
- Change the ownership of our terminal device (`chown`) so we own it
- Change the access permissions for our terminal device so we have permission to read from and write to it
- Set our group IDs by calling `setgid` and `initgroups`
- Initialize the environment with all the information that login has: our home directory (`HOME`), shell (`SHELL`), user name (`USER` and `LOGNAME`), and a default path (`PATH`)
- Change to our user ID (`setuid`) and invoke our login shell, as in `execl("/bin/sh", "-sh", (char *)0)`;

![Arrangement of processes after everything is set for a terminal login](http://i.imgur.com/Yjc4y76.png)

# Network Logins

As part of the system start-up, `init` invokes a shell that executes the shell script `/etc/rc`. One of the daemons that is started by this shell script is `inetd`. `inetd` waits for TCP/IP connection requests to arrive at the host. When a connection request arrives for it to handle, `inetd` does a `fork` and `exec` of the appropriate program.

![Sequence of processes involved in executing TELNET server](http://i.imgur.com/91zu9hJ.png)

The launched program, for exmaple, `telnetd`, then opens a *pseudo terminal device* and splits into two processes using `fork`. The parent handles the communication across the network connection, and the child does an `exec` of the login program.

![Arrangement of processes after everything is set for a network login](http://i.imgur.com/ivf6gjp.png)

# Process Groups

A process group is a collection of one or more processes, usually associated with the same job, that can receive signals from the same terminal. Each process group can have a process group leader. The leader is identified by its process group ID being equal to its process ID.

```c
#include <unistd.h>

/* Returns: process group ID of calling process */
pid_t getpgrp(void);

/* Returns: process group ID if OK, −1 on error */
pid_t getpgid(pid_t pid);
```

If `pid` is 0, the process group ID of the calling process is returned; thus, `getpgid(0)` is equivalent to `getpgrp()`.

A process joins an existing process group or creates a new process group by calling `setpgid`. A process can set the process group ID of **only itself or any of its children**. Furthermore, it can’t change the process group ID of one of its children after that child has called one of the `exec` functions.

```c
#include <unistd.h>

/* Returns: 0 if OK, −1 on error */
int setpgid(pid_t pid, pid_t pgid);
```

If the two arguments are equal, the process specified by `pid` becomes a process group leader. If `pid` is 0, the process ID of the caller is used. Also, if `pgid` is 0, the process ID specified by `pid` is used as the process group ID.

# Sessions

A session is a collection of one or more process groups.

![Arrangement of processes into process groups and sessions](http://i.imgur.com/Owb3jEq.png)

The processes in a process group are usually placed there by a shell pipeline. For example,

```bash
$ proc1 | proc2 &
$ proc3 | proc4 | proc5
```

A process establishes a new session by calling the `setsid` function:

```c
#include <unistd.h>

/* Returns: process group ID if OK, −1 on error */
pid_t setsid(void);

/* Returns: session leader’s process group ID if OK, −1 on error */
pid_t getsid(pid_t pid);
```

If the calling process is not a process group leader, this function creates a new session. Three things happen:


1. The process becomes the *session leader* of this new session. (A session leader is the process that creates a session.) The process is **the only process in this new session**.
2. The process becomes the process group leader of a new process group. The new process group ID is the process ID of the calling process.
3. The process **has no controlling terminal**. If the process had a controlling terminal before calling setsid, that association is broken.

This function returns an error if the caller is already a process group leader.

# Controlling Terminal

Sessions and process groups have a few other characteristics.

- A session can have a **single** *controlling terminal*. This is usually the terminal device (in the case of a terminal login) or pseudo terminal device (in the case of a network login) on which we log in.
- The session leader that establishes the connection to the controlling terminal is called the *controlling process*.
- The process groups within a session can be divided into a **single** *foreground process group* and one or more *background process groups*.
- If a session has a controlling terminal, it has a single foreground process group and all other process groups in the session are background process groups.
- Whenever we press the terminal’s interrupt key (often DELETE or Control-C), the interrupt signal is sent to all processes in the **foreground process group**.
- Whenever we press the terminal’s quit key (often Control-backslash), the quit signal is sent to all processes in the **foreground process group**.
- If a modem (or network) disconnect is detected by the terminal interface, the hang-up signal is sent to the **controlling process** (the session leader).

![Process groups and sessions showing controlling terminal](http://i.imgur.com/RNiMTv5.png)

# `tcgetpgrp`, `tcsetpgrp`, and `tcgetsid` Functions

```c
#include <unistd.h>

/* Returns: process group ID of foreground process group if OK, −1 on error */
pid_t tcgetpgrp(int fd);

/* Returns: 0 if OK, −1 on error */
int tcsetpgrp(int fd, pid_t pgrpid);
```

The function `tcgetpgrp` returns the process group ID of the foreground process group associated with the terminal open on `fd`.

If the process has a controlling terminal, the process can call `tcsetpgrp` to set the foreground process group ID to `pgrpid`. The value of `pgrpid` must be the process group ID of a process group in the same session, and `fd` must refer to the controlling terminal of the session.

The `tcgetsid` function allows an application to obtain the process group ID for the session leader given a file descriptor for the controlling TTY:

```c
#include <termios.h>

/* Returns: session leader’s process group ID if OK, −1 on error */
pid_t tcgetsid(int fd);
```

# Job Control

The interaction with the terminal driver arises because a special terminal character affects the foreground job:

- The interrupt character (typically DELETE or Control-C) generates `SIGINT`.
- The quit character (typically Control-backslash) generates `SIGQUIT`.
- The suspend character (typically Control-Z) generates `SIGTSTP`.

When a background job tries to read from the controlling terminal, it causes the signal `SIGTTIN` to be generated.

The following figure summarizes some of the features of job control:

![Summary of job control features with foreground and background jobs, and terminal driver](http://i.imgur.com/MFkouc6.png)
