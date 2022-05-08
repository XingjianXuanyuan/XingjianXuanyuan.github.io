---
layout: post
title: "Linux Processes: fork() and execve() Under the Hood"
category: "Computing Systems"
---

In this post, I would like to take a brief look at two Linux system calls: [<code>fork(2)</code>](https://man7.org/linux/man-pages/man2/fork.2.html) and [<code>execve(2)</code>](https://man7.org/linux/man-pages/man2/execve.2.html). The number enclosed in parentheses indicates the section of the Linux man pages in which the objects are described.

<p style="color:gray; font-size:60%;">Note that the [Linux man pages](https://man7.org/linux/man-pages/index.html) is divided into eight sections:

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

~~~ C
#include <unistd.h>

pid_t fork(void);

int execve(const char *pathname, char *const argv[], char *const envp[]);
~~~