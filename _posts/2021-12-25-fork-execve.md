---
layout: post
title: "Linux Processes: fork() and execve() Under the Hood"
category: "Computing Systems"
---

In this post, I would like to give a brief account of two Linux system calls---[<code>fork(2)</code>](https://man7.org/linux/man-pages/man2/fork.2.html) and [<code>execve(2)</code>](https://man7.org/linux/man-pages/man2/execve.2.html)---illustrated with Linux socket programming examples written in C. These two functions are commonly used by Linux processes from both user and kernel spaces. Note that the number enclosed in parentheses indicates the section of the Linux man pages in which the objects are described.

<p style="color:gray; font-size:80%;">
The <a href="https://man7.org/linux/man-pages/index.html">Linux man pages</a> is divided into eight sections:
1. User commands and tools;
2. Linux system calls and system call wrappers;
3. Library functions excluding system call wrappers;
4. Special files (devices);
5. File formats and filesystems;
6. Games and funny little programs available on the system;
7. Overview and miscellany section;
8. Administration and privileged commands.
</p>

<!-- excerpt-end -->

The Linux **system calls**{: style="color: red"} (or *syscalls*) are the only means user applications have of interfacing with the kernel; they are the only legal entry point into the kernel other than exceptions and traps. In other words, the Linux kernel is concerned only with the system calls; it is important for the kernel to keep track of the potential use of a system call and keep the system call as general and flexible as possible. System calls are typically accessed via function calls defined in the **C library**{: style="color: red"}. The C library is used by all C programs and, because of C's nature, is easily wrapped by other programming languages for use in their programs.

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## The Concept of a Process

"**Process**{: style="color: red"}" is viewed by many as one of the greatest abstraction in the history of computing. The notion of process provides us with the illusions that the program file we run, being the only objects in the system, has exclusive use of both the processor and the memory, and the processor executes the instructions in our program without interruption. The classic definition of a process is *an instance of a program in execution*[^1]. Processes are more than just the executing program code (often called the **text section**{: style="color: red"} in Unix). They also include a set of resources such as open files and pending signals, internal operating system kernel data, processor state, a memory address space with one or more memory mappings, one or more **threads**{: style="color: red"} of execution (often shortened to *threads*), and a **data section**{: style="color: red"} containing global (as well as static) variables. The concepts of program, process, and thread might sometimes be confusing. To make clear:

- A program itself is not a process; a process is an active program and related resources[^2].

- A thread (sometimes called *lightweight processes*) is part of a process that is necessary to execute code[^3].

<p style="color:gray; font-size:80%;">
On most computers this means each thread has a pointer to the thread's current instruction ("program counter"), a pointer to the top of the thread's stack (<code>%rsp</code>), general registers, and floating-point or address registers if they are kept separate. Multiple threads can exist within a single process. They share all of the files and memory, including the program text and data sections. In traditional Unix systems, each process consists of one thread. In modern systems, however, multi-threaded programs are common. Linux does not have explicit kernel support (any special scheduling semantics or data structures) for threads; a Linux thread is merely a process that shares certain resources with other processes.
</p>

Most operating systems implement a "spawn" mechanism to create a new process in a new address space, read in an executable, and begin executing it. One Unix process spawns another either by replacing itself when it is done---call one of the six <code>exec()</code> functions---or, if it needs to stay around, by making a copy of itself---call <code>fork()</code>.

<p style="color:gray; font-size:80%;">
The differences in the six <code>exec()</code> functions (<code>execl(), execv(), execle(), execve(), execlp(), execvp()</code>) are: (a) whether the program file to execute is specified by a filename or a pathname; (b) whether the arguments to the new program are listed one by one or referenced through an array of pointers; and (c) whether the environment of the calling process (the process that calls <code>exec()</code>) is passed to the new program or whether a new environment is specified. Normally, only <code>execve()</code> is a system call within the kernel and the other five are library functions that call <code>execve()</code>.
</p>

```c
#include <unistd.h>

/* 
 * On success, the PID of the child process is returned in the
 * parent, 0 is returned in the child. On failure, -1 is returned
 * in the parent.
 */
pid_t
fork(void);

/* Never returns on success; -1 is returned on error. */
int
execve(const char *pathname, char *const argv[], char *const envp[]);
```

The child differs from the parent only in its PID (which is unique), its PPID (parent's PID, which is set to the original process), and certain resources and statistics, such as pending signals, which are not inherited. The <code>fork()</code> function is called once but it returns twice: once in the calling process (the parent), and once in the newly created child process. The parent and child are separate processes that run concurrently. According to SFR(2004)[^4]:

> "The reason <code>fork()</code> returns 0 in the child, instead of the parent's process ID, is because a child has only one parent and it can always obtain the parent's process ID by calling <code>getppid()</code>. A parent, on the other hand, can have any number of children, and there is no way to obtain the process IDs of its children. If a parent wants to keep track of the process IDs of all its children, it must record the return values from <code>fork()</code>."

The <code>execve()</code> function loads and runs the executable object file specified by the first argument with the argument list <code>argv</code> (a null-terminated array of pointers) and the environment variable list <code>envp</code> (a null-terminated array of pointers to name-value pairs). By convention, <code>argv[0]</code> is the name of the executable object file.

When the kernel has started itself (has been loaded into memory, has started running, and has initialized all device drivers and data structures and such), the kernel thread created by process 0 (the *idle process*) executes the <code>init()</code> function, which then invokes the <code>execve()</code> system call to load a user-level program---**init**{: style="color: red"}. The kernel looks for <code>init</code> in a few locations that have been historically used for it, but the proper location for it on a Linux system is <code>/sbin/init</code>. If the kernel cannot find <code>init</code>, it tries to run <code>/bin/sh</code>, and if that also fails, the startup of the system fails.

## Implementation Details of fork() and execve()

### fork()

The <code>start_kernel()</code> function initializes all the data structures needed by the kernel, enables interrupts, and creates the <code>init</code> process by calling the <code>arch_call_rest_init()</code> function near the end. The <code>arch_call_rest_init()</code> function calls the <code>rest_init()</code> function:

```c
noinline void __ref rest_init(void)
{
    /* ... */
    pid = kernel_thread(kernel_init, NULL, CLONE_FS);
    /* ... */
}
```

The above details can be found in the <code>/init/main.c</code> directory of the [Linux source code](https://elixir.bootlin.com/linux/latest/source). The <code>kernel_thread()</code> function (described in <code>/kernel/fork.c</code>) creates the <code>init</code> process by the following line of code:

```c
return kernel_clone(&args);
```

According to the [Linux implementation](https://elixir.bootlin.com/linux/latest/source/kernel/fork.c), it is this <code>kernel_clone()</code> function that actually does the work for the <code>fork()</code> system call:

```c
SYSCALL_DEFINE0(fork)
{
#ifdef CONFIG_MMU
	struct kernel_clone_args args = {
		.exit_signal = SIGCHLD,
	};

	return kernel_clone(&args);
#else
	/* can not support in nommu mode */
	return -EINVAL;
#endif
}
```

## References

[^1]: Randal E. Bryant and David R. O'Hallaron, *Computer Systems: A Programmer's Perspective, Third Edition*, Pearson Education, 2016.

[^2]: Robert Love, *Linux Kernel Development, Third Edition*, Pearson Education, 2010.

[^3]: David R. Butenhof, *Programming with POSIX Threads*, Addison-Wesley, 1997.

[^4]: W. Richard Stevens, Bill Fenner, and Andrew M. Rudoff, *UNIX Network Programming Volume 1, Third Edition: The Sockets Networking API*, Addison-Wesley, 2004.