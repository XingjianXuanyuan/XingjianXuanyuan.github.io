---
layout: post
title: "Linux Processes: fork() and execve() Under the Hood"
category: "Computing Systems"
---

In this post, I would like to give a brief account of two Linux system calls---[<code>fork(2)</code>](https://man7.org/linux/man-pages/man2/fork.2.html) and [<code>execve(2)</code>](https://man7.org/linux/man-pages/man2/execve.2.html)---with operating system kernel implementation details (not glibc wrappers) presented and two code examples explained. <code>fork(2)</code> and <code>execve(2)</code> are commonly used by Linux processes from both user and kernel spaces. In particular, as you can see below, they are involved in the Linux kernel initialization process. [The GitBook "Linux Insides"](https://0xax.gitbooks.io/linux-insides/content/) provides a comprehensive discussion on this topic.

<p style="color:gray; font-size:80%;">
Note that the number enclosed in parentheses after the object name indicates the section of the Linux man pages in which the object is described. The <a href="https://man7.org/linux/man-pages/index.html">Linux man pages</a> is divided into eight sections:
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

"**Process**{: style="color: red"}" is viewed by many as one of the greatest abstraction in the history of computing. The notion of process provides us with the illusions that the program file we run, being the only objects in the system, has exclusive use of both the processor and the memory, and the processor executes the instructions in our program without interruption. The classic definition of a process is *an instance of a program in execution*[^1]. Processes are more than just the executing program code (often called the **text section**{: style="color: red"} in UNIX). They also include a set of resources such as open files and pending signals, internal operating system kernel data, processor state, a memory address space with one or more memory mappings, one or more **threads**{: style="color: red"} of execution (often shortened to *threads*), and a **data section**{: style="color: red"} containing global (as well as static) variables. The concepts of program, process, and thread might sometimes be confusing. To make clear:

- A program itself is not a process; a process is an active program and related resources[^2].

- A thread (sometimes called *lightweight processes*) is part of a process that is necessary to execute code[^3].

<p style="color:gray; font-size:80%;">
On most computers this means each thread has a pointer to the thread's current instruction ("program counter"), a pointer to the top of the thread's stack (<code>%rsp</code>), general registers, and floating-point or address registers if they are kept separate. Multiple threads can exist within a single process. They share all of the files and memory, including the program text and data sections. In traditional UNIX systems, each process consists of one thread. In modern systems, however, multi-threaded programs are common. Linux does not have explicit kernel support (any special scheduling semantics or data structures) for threads; a Linux thread is merely a process that shares certain resources with other processes.
</p>

Most operating systems implement a "spawn" mechanism to create a new process in a new address space, read in an executable, and begin executing it. One UNIX process spawns another either by replacing itself when it is done---call one of the six <code>exec()</code> functions---or, if it needs to stay around, by making a copy of itself---call <code>fork()</code>.

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

The child differs from the parent only in its PID (which is unique), its PPID (parent's PID, which is set to the original process), and certain resources and statistics, such as pending signals, which are not inherited.

<p style="color:gray; font-size:80%;">
A child created via <code>fork()</code> initially has an empty pending signal set; the pending signal set is preserved across an <code>execve()</code>. However, a child created via <code>fork()</code> inherits a copy of its parent's signal dispositions. During an <code>execve()</code>, the dispositions of handled signals are reset to the default; the dispositions of ignored signals are left unchanged.
</p>

The <code>fork()</code> function is called once but it returns twice: once in the calling process (the parent), and once in the newly created child process. The parent and child are separate processes that run concurrently. It is thus important to realize that after a <code>fork()</code>, it is indeterminate which of the two processes is next scheduled to use the CPU. According to SFR(2004)[^4]:

> "The reason <code>fork()</code> returns 0 in the child, instead of the parent's process ID, is because a child has only one parent and it can always obtain the parent's process ID by calling <code>getppid()</code>. A parent, on the other hand, can have any number of children, and there is no way to obtain the process IDs of its children. If a parent wants to keep track of the process IDs of all its children, it must record the return values from <code>fork()</code>."

Conceptually, we can consider <code>fork()</code> as creating copies of the parent's text, data, heap, and stack segments. However, actually performing a simple copy of the parent's virtual memory pages into the new child process would be wasteful. According to [TLPI](https://man7.org/tlpi/index.html)[^5], most modern UNIX implementations, including Linux, use two techniques to avoid such wasteful copying:

> "The kernel marks the text segment of each process as read-only, so that a process cannot modify its own code. This means that the parent and child can share the same text segment. The <code>fork()</code> system call creates a text segment for the child by building a set of per-process page-table entries that refer to the same virtual memory page frames already used by the parent. For the pages in the data, heap, and stack segments of the parent process, the kernel employs a technique known as **copy-on-write**{: style="color: red"}. Initially, the kernel sets things up so that the page-table entries for these segments refer to the same physical memory pages as the corresponding page-table entries in the parent, and the pages themselves are marked read-only. After the code>fork()</code>, the kernel traps any attempts by either the parent or the child to modify one of these pages, and makes a duplicate copy of the about-to-be-modified page. This new page copy is assigned to be the faulting process, and the corresponding page-table entry for the child is adjusted appropriately."

The <code>execve()</code> function loads and runs the executable object file specified by the first argument with the argument list <code>argv</code> (a null-terminated array of pointers) and the environment variable list <code>envp</code> (a null-terminated array of pointers to name-value pairs in the form <code>name=value</code>). By convention, <code>argv[0]</code> is the name of the executable object file. After an <code>execve()</code>, the process ID of the process remains the same. If the set-user-ID (set-group-ID) permission bit of the program file specified by <code>pathname</code> is set, then, when the file is <code>exec</code>ed, the effective user (group) ID of the process is changed to be the same as the owner (group) of the program file. After optionally changing the effective IDs, and regardless of whether they were changed, an <code>execve()</code> copies the value of the process's effective user ID into its saved set-user-ID, and copies the value of the process's effective group ID into its saved set-group-ID. For more details about the effect of <code>fork()</code> and <code>execve()</code> on process attributes, please refer to TLPI Chapter 28. Furthermore, the POSIX specification now lists 25 special cases in how the parent's state is copied to the child: file locks, timers, asynchronous I/O operations, tracing, etc[^6].

When the kernel has started itself (has been loaded into memory, has started running, and has initialized all device drivers and data structures and such), the kernel thread created by the *idle process* (<code>PID=0</code>) executes the <code>init()</code> function, which then invokes the <code>execve()</code> system call to load a user-level program---**init**{: style="color: red"}. The kernel looks for <code>init</code> in a few locations that have been historically used for it, but the proper location for it on a Linux system is <code>/sbin/init</code>. If the kernel cannot find <code>init</code>, it tries to run <code>/bin/sh</code>, and if that also fails, the startup of the system fails.

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

The above details can be found in the <code>/init/main.c</code> directory of the [Linux source code](https://elixir.bootlin.com/linux/latest/source). The <code>kernel_thread()</code> function (described in <code>/kernel/fork.c</code>) creates the <code>init</code> process using this line of code:

```c
return kernel_clone(&args);
```

It is this <code>kernel_clone()</code> function that actually does the work for the <code>fork()</code> system call:

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

Here, <code>SYSCALL_DEFINE0()</code> is simply a macro that defines a system call with no parameters (hence the 0).

<p style="color:gray; font-size:80%;">
The <code>clone()</code> system call is similar to <code>fork()</code> except that more arguments are specified by the user to gain more control over what pieces of execution context are shared between the calling process and the child process.
</p>

### execve()

We can find the definition of the <code>execve()</code> function inside [<code>/fs/exec.c</code>](https://elixir.bootlin.com/linux/latest/source/fs/exec.c):

```c
SYSCALL_DEFINE3(execve,
		const char __user *, filename,
		const char __user *const __user *, argv,
		const char __user *const __user *, envp)
{
	return do_execve(getname(filename), argv, envp);
}
```

The <code>do_execve()</code> function is a wrapper function that calls <code>do_execveat_common()</code>. The latter does the actual work for <code>execve()</code>:

```c
static int
do_execveat_common(int fd, struct filename *filename,
				   struct user_arg_ptr argv,
				   struct user_arg_ptr envp,
				   int flags)
{
	struct linux_binprm *bprm;
	int retval;

	if (IS_ERR(filename))
		return PTR_ERR(filename);
	
	if ((current->flags & PF_NPROC_EXCEEDED) && is_ucounts_overlimit(current_ucounts(), UCOUNT_RLIMIT_NPROC, rlimit(RLIMIT_NPROC))) {
		retval = -EAGAIN;
		goto out_ret;
	}
	
	current->flags &= ~PF_NPROC_EXCEEDED;
	
	bprm = alloc_bprm(fd, filename);
	if (IS_ERR(bprm)) {
		retval = PTR_ERR(bprm);
		goto out_ret;
	}

	retval = count(argv, MAX_ARG_STRINGS);
	if (retval == 0)
		pr_warn_once("process '%s' launched '%s' with NULL argv: empty string added\n", current->comm, bprm->filename);
	if (retval < 0)
		goto out_free;
	bprm->argc = retval;

	retval = count(envp, MAX_ARG_STRINGS);
	if (retval < 0)
		goto out_free;
	bprm->envc = retval;

	retval = bprm_stack_limits(bprm);
	if (retval < 0)
		goto out_free;

	retval = copy_string_kernel(bprm->filename, bprm);
	if (retval < 0)
		goto out_free;
	bprm->exec = bprm->p;

	retval = copy_strings(bprm->envc, envp, bprm);
	if (retval < 0)
		goto out_free;
	
	retval = copy_strings(bprm->argc, argv, bprm);
	if (retval < 0)
		goto out_free;
	
	if (bprm->argc == 0) {
		retval = copy_string_kernel("", bprm);
		if (retval < 0)
			goto out_free;
		bprm->argc = 1;
	}

	retval = bprm_execve(bprm, fd, filename, flags);
out_free:
	free_bprm(bprm);
out_ret:
	putname(filename);
	return retval;
}
```

Declared in <code>/include/linux/binfmts.h</code>, the <code>linux_binprm</code> structure is used to hold the arguments that are used when loading binaries.

## A Simple User-Space Program: <code>execve_cat.c</code>

```c
#include <unistd.h>
#include <stdio.h>

int main(int argc, char *argv[])
{
    char *var[3];

    if (argc < 2) {
        printf("Please type a file name.\n");
        return -1;
    }

    var[0] = "/bin/cat";
    var[1] = argv[1];
    var[2] = 0;
    execve(var[0], var, 0);

    return 0;
}
```

The above program from Du (2019)[^7] asks the <code>execve()</code> function to execute the following shell command:

```console
cat <filename>
```

We can compile the program and run it with the root privilege:

```console
$ gcc execve_cat.c -o execcat
$ sudo chown root execcat
$ sudo chmod 4755 execcat
$ ./execcat /etc/passwd
...
nobody:*:-2:-2:Unprivileged User:/var/empty:/usr/bin/false
root:*:0:0:System Administrator:/var/root:/bin/sh
daemon:*:1:1:System Services:/var/root:/usr/bin/false
...
```

## Concurrent Server

The server program <code>unp_server.c</code> modified from SFR(2004) first creates a TCP socket, binds it with the wildcard address <code>INADDR_ANY</code> and port number defined by the macro <code>SERV_PORT</code>, and blocks in the call to <code>accept()</code>, waiting for a client connection to complete. For each client, the server uses <code>fork()</code> to spawn a child. The child calls <code>str_echo</code> to handle the new client.

```c
/* str_echo: performs server processing for each client.
 *           It reads data from the client and echoes it
 *           back to the client.
 */
void str_echo(int sockfd)
{
    ssize_t n;
    char buf[MAXLINE];
    while ((n = read(sockfd, buf, MAXLINE)) > 0)
        Writen(sockfd, buf, n);
    if (n < 0)
        err_sys("str_echo: read error");
}

int main(int argc, char *argv[])
{
    int listenfd, connfd;
    pid_t childpid;
    socklen_t clilen;
    struct sockaddr_in cliaddr, servaddr;

    listenfd = Socket(AF_INET, SOCK_STREAM, 0);

    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(SERV_PORT);

    Bind(listenfd, (SA *) &servaddr, sizeof(servaddr));

    Listen(listenfd, LISTENQ);

    for ( ; ; ) {
        clilen = sizeof(cliaddr);
        connfd = Accept(listenfd, (SA *) &cliaddr, &clilen);

        if ((childpid = Fork()) == 0) { /* child process*/
            Close(listenfd);            /* close listening socket */
            str_echo(connfd);           /* process the request */
            exit(0);
        }
        Close(connfd);                  /* parent closes connected socket */
    }

    return 0;
}
```

The functions with capitalized names are wrapper functions that perform error checking for the actual function calls, which is a recommended technique in SFR.

```c
/* Socket wrapper functions */
int Accept(int fd, struct sockaddr *sa, socklen_t *salenptr);		/* accept() */
void Bind(int fd, const strut sockaddr *sa, socklen_t salen);		/* bind() */
void Connect(int fd, const struct sockaddr *sa, socklen_t salen);	/* connect() */
int Socket(int family, int type, int protocol);						/* socket() */
void Close(int fd);													/* close() */
pid_t Fork(void);													/* fork() */
```

The whole program code can be found [here](https://github.com/XingjianXuanyuan/CSE422S-OperatingSystemsOrganization/blob/main/Src/Studio16/unp_server.c).

## An Experiment on Program Execution after a <code>fork()</code>

In this final section, I am concerned with how a program executes after a <code>fork()</code> on my Mac; which process is scheduled first, the parent or the child? The <code>spawn_children.c</code> program is modified from LPI, which uses <code>fork()</code> to create multiple children (the number of children spawned is user-specified) with a <code>for</code> loop. The whole program code can be found [here](https://github.com/XingjianXuanyuan/CSE422S-OperatingSystemsOrganization/blob/main/Src/Studio8/spawn_children.c). After each <code>fork()</code>, both parent and child print a message containing the loop counter value and a string indicating whether the process is the parent or child. For example, if we asked the program to produce two children, we might see the following (I was compiling on a macOS/x86-64 system):

```console
$ gcc spawn_children.c -o spawn
$ ./spawn 2
0 parent
0 child
1 parent
1 child
The elapsed time is 0.000000000 seconds.
```

We can run this program to create a large number of children. The output messages tell us whether the parent or the child is the first to print its message after each <code>fork()</code>. I use a script file called [<code>fork_whos_on_first.count.awk</code>](https://github.com/bradfa/tlpi-dist/blob/master/procexec/fork_whos_on_first.count.awk) provided by the author of LPI to analyze the output messages of <code>spawn_children.c</code>:

```console
$ sudo chmod +x fork_whos_on_first.count.awk
$ ./spawn 1000 > spawn_output.txt
$ ./fork_whos_on_first.count.awk spawn_output.txt 
All done
parent    996  99.60%
child       4   0.40%

$ ./spawn 10000 > spawn_output.txt
$ ./fork_whos_on_first.count.awk spawn_output.txt
All done
parent   9983  99.83%
child      17   0.17%

$ ./spawn 100000 > spawn_output.txt
$ ./fork_whos_on_first.count.awk spawn_output.txt
All done
parent  99859  99.86%
child     141   0.14%
```

For the case of $100,000$ children created, the execution time of the <code>spawn_children.c</code> is $29.741205000$ seconds. The results show that the parent is almost always scheduled first after a <code>fork()</code>.

## References

[^1]: Randal E. Bryant and David R. O'Hallaron, *Computer Systems: A Programmer's Perspective, Third Edition*, Pearson Education, 2016.

[^2]: Robert Love, *Linux Kernel Development, Third Edition*, Pearson Education, 2010.

[^3]: David R. Butenhof, *Programming with POSIX Threads*, Addison-Wesley, 1997.

[^4]: W. Richard Stevens, Bill Fenner, and Andrew M. Rudoff, *UNIX Network Programming Volume 1, Third Edition: The Sockets Networking API*, Addison-Wesley, 2004.

[^5]: Michael Kerrisk, *The Linux Programming Interface: A Linux and UNIX System Programming Handbook*, San Francisco: No Starch Press, 2010.

[^6]: *Base Specifications POSIX.1-2017*. The Open Group, San Francisco, CA, USA, 2018. URL http://pubs.opengroup.org/onlinepubs/9699919799/functions/fork.html. IEEE Std 1003.1-2017.

[^7]: Wenliang Du, *Computer & Internet Security: A Hands-on Approach, Second Edition*, 2019.