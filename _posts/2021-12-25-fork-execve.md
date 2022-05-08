---
layout: post
title: "Linux Processes: fork() and execve() Under the Hood"
category: "Computing Systems"
---

In this post, I would like to give a brief account of two Linux system calls: [<code>fork(2)</code>](https://man7.org/linux/man-pages/man2/fork.2.html) and [<code>execve(2)</code>](https://man7.org/linux/man-pages/man2/execve.2.html). They are commonly used by Unix processes from both user and kernel spaces. Note that the number enclosed in parentheses indicates the section of the Linux man pages in which the objects are described.

<p style="color:gray; font-size:80%;">The <a href="https://man7.org/linux/man-pages/index.html">Linux man pages</a> is divided into eight sections:
1. User commands and tools;
2. Linux system calls and system call wrappers;
3. Library functions excluding system call wrappers;
4. Special files (devices);
5. File formats and filesystems;
6. Games and funny little programs available on the system;
7. Overview and miscellany section;
8. Administration and privileged commands.</p>

<!-- excerpt-end -->

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## Processes

~~~ C
#include <unistd.h>

/* 
 * On success, the PID of the child process is returned in the parent,
 * 0 is returned in the child. On failure, -1 is returned in the parent.
 */
pid_t fork(void);

/* Never returns on success; -1 is returned on error. */
int execve(const char *pathname, char *const argv[], char *const envp[]);
~~~

According to SFR(2004)[^1]:

> "the reason <code>fork</code> returns 0 in the child, instead of the parent's process ID, is because a child has only one parent and it can always obtain the parent's process ID by calling <code>getppid</code>. A parent, on the other hand, can have any number of children, and there is no way to obtain the process IDs of its children. If a parent wants to keep track of the process IDs of all its children, it must record the return values from <code>fork</code>."

[^1]: W. Richard Stevens, Bill Fenner, and Andrew M. Rudoff, *UNIX Network Programming Volume 1, Third Edition: The Sockets Networking API*, Addison-Wesley, 2004.