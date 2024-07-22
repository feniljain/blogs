# JITs from systems perspective

NOTE: This tutorial assumes you are working on a linux x86_64 machine. Tho it's not hard to replicate the same results using listed resources in blog.

## Linux calling convention

Let's try to write a simple hello world program in asm which we print "hello world :)" to stdout using write syscall, we then finally exit the program with code zero to indicate no errors. Calling functions in asm happens using `call` instruction, and calling syscalls happens using `syscall` instruction, linux on x86_64 expects data in this format before calling syscall instruction:

- rax contains syscall number, for our case write is 1 and exit is 60
- rdi contains first argument
- rsi contains second argument
- rdx contains third argument
- r10 contains fourth argument
- r8 contains fifth argument
- r9 contains sixth argument

You can find more details about it here: https://stackoverflow.com/questions/2535989/what-are-the-calling-conventions-for-unix-linux-system-calls-and-user-space-f

And this is a nice small collective table for different syscalls and what args to pass in what registers from chromium project:

https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md#x86_64-64_bit

We see `write` has to be passed:

rax: 1
rdi: 1 (for stdout)
rsi: buffer string address (pointing to our string of "hello world :)")
rdx: 14 cause that's the length of our string

These are all the arguments to be passed. As a small exercise think what needs to be passsed for exit syscall.

Cool let's now look at the final program directly, it should be easy to understand:

```asm
global _start

section .text

_start:
    mov rax, 1
    mov rdi, 1
    mov rsi, msg
    mov rdx, 14
    syscall

    mov rax, 60
    mov rdi, 0
    syscall

section .rodata:
    msg: db "hello world :)", 14
```

We will run this using nasm:
```sh
nasm -f elf64 -o hello.o hello.s && ld -o hello hello.o && ./hello
```

First we convert our asm program into object format of elf64 because that's what linux expects on x86_64 systems, then we link it and finally execute it.
We have covered linux calling convention somewhat, there are a lot of other things too like how to pass variadic args, etc. but we don't need them in our case.
Next let's talk a bit about virtual memory and get some intuition around where code is located, and how does it relate to VM (virtual memory).

## Virtual Memory

Let's write a small C program which again prints a simple "hello world :)"

```C
#include <stdio.h>

int main () {
    printf("Hello world :)\n");
    return 0;
}
```

let's compile it and also run it just to make sure we didn't insert greek semicolon in there :p

```sh
gcc -g hello_world.c && ./a.out
```

That should give out a nice small "Hello world :)" output. Now let's fire up gdb on our final executable `a.out` using:

```sh
gdb a.out
```

and then in gdb's console let's set a breakpoint just before printing happens using:
```sh
break hello_world.c:4
```

next we run the program using well: `run`, our program should have halted at the set breakpoint. Now let's enter this command:
```sh
info proc mappings
```

You should get an output similar to:
```sh
(gdb) info proc mappings
process 7353
Mapped address spaces:

          Start Addr           End Addr       Size     Offset  Perms  objfile
      0x555555554000     0x555555555000     0x1000        0x0  r--p   /home/fenil/Projects/cpp-projects/hello-world/a.out
      0x555555555000     0x555555556000     0x1000     0x1000  r-xp   /home/fenil/Projects/cpp-projects/hello-world/a.out
      0x555555556000     0x555555557000     0x1000     0x2000  r--p   /home/fenil/Projects/cpp-projects/hello-world/a.out
      0x555555557000     0x555555558000     0x1000     0x2000  r--p   /home/fenil/Projects/cpp-projects/hello-world/a.out
      0x555555558000     0x555555559000     0x1000     0x3000  rw-p   /home/fenil/Projects/cpp-projects/hello-world/a.out
      0x7ffff7c00000     0x7ffff7c28000    0x28000        0x0  r--p   /usr/lib/x86_64-linux-gnu/libc.so.6
      0x7ffff7c28000     0x7ffff7dbd000   0x195000    0x28000  r-xp   /usr/lib/x86_64-linux-gnu/libc.so.6
      0x7ffff7dbd000     0x7ffff7e15000    0x58000   0x1bd000  r--p   /usr/lib/x86_64-linux-gnu/libc.so.6
      0x7ffff7e15000     0x7ffff7e16000     0x1000   0x215000  ---p   /usr/lib/x86_64-linux-gnu/libc.so.6
      0x7ffff7e16000     0x7ffff7e1a000     0x4000   0x215000  r--p   /usr/lib/x86_64-linux-gnu/libc.so.6
      0x7ffff7e1a000     0x7ffff7e1c000     0x2000   0x219000  rw-p   /usr/lib/x86_64-linux-gnu/libc.so.6
      0x7ffff7e1c000     0x7ffff7e29000     0xd000        0x0  rw-p
      0x7ffff7f9d000     0x7ffff7fa0000     0x3000        0x0  rw-p
      0x7ffff7fbb000     0x7ffff7fbd000     0x2000        0x0  rw-p
      0x7ffff7fbd000     0x7ffff7fc1000     0x4000        0x0  r--p   [vvar]
      0x7ffff7fc1000     0x7ffff7fc3000     0x2000        0x0  r-xp   [vdso]
      0x7ffff7fc3000     0x7ffff7fc5000     0x2000        0x0  r--p   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
      0x7ffff7fc5000     0x7ffff7fef000    0x2a000     0x2000  r-xp   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
      0x7ffff7fef000     0x7ffff7ffa000     0xb000    0x2c000  r--p   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
      0x7ffff7ffb000     0x7ffff7ffd000     0x2000    0x37000  r--p   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
      0x7ffff7ffd000     0x7ffff7fff000     0x2000    0x39000  rw-p   /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
      0x7ffffffdd000     0x7ffffffff000    0x22000        0x0  rw-p   [stack]
  0xffffffffff600000 0xffffffffff601000     0x1000        0x0  --xp   [vsyscall]
```

We could also get this by getting process ID of current process (in gdb run `inferior`) and running:

```sh
cat /proc/<pid>/maps
```

This is virtual address space of our program, I am not going to cover what exactly that is cause that would take few more blogs on it's own. Main point for us is seeing where our current breakpoint lies, to get what address we are on, let's run `disassemble` command:

```sh
(gdb) disassemble
Dump of assembler code for function main:
   0x0000555555555149 <+0>:     endbr64
   0x000055555555514d <+4>:     push   rbp
   0x000055555555514e <+5>:     mov    rbp,rsp
=> 0x0000555555555151 <+8>:     lea    rax,[rip+0xeac]        # 0x555555556004
   0x0000555555555158 <+15>:    mov    rdi,rax
   0x000055555555515b <+18>:    call   0x555555555050 <puts@plt>
   0x0000555555555160 <+23>:    mov    eax,0x0
   0x0000555555555165 <+28>:    pop    rbp
   0x0000555555555166 <+29>:    ret
End of assembler dump.
```

We are currently on `0x0000555555555151`, let's check which range does it lie in VM output.
It lies in second row i.e. between `0x555555555000` and `0x555555556000`, this has a size of 0x1000, which is 4096 in decimal, i.e. 4KB, that is also the size of a page. Looking at the perms we have `r-x`, `r` is read and `x` is for executable. That means this page is executable and that's why it also contains our code.

What if we could write arbitary code to a memory region which is size as 4KB, mark it as executable and call it as a function? Doing all this in C should allow us to execute arbitary code, cause if there's a way to write a page while program is running, we could just keep re-writing it if we want to change what needs to be executed.

This is what JIT is from systems perspective, we simple allocate a memory region, write our code in it, change it's permissions to be executable and then call it as a function :)

Cool, let's try to do just this in next section.

## Actually building it

We need to write actual platform specific asm in the page, so let's convert our earlier written hello world asm program to behave like a function?

```asm
# ssize_t write(int fd, const void *buf, size_t count);

mov rax, 1 # write syscall number
mov rsi, rdi # second arg to write syscall
mov rdi, 1 # first arg to write syscall
mov rdx, 18 # third arg to write syscall
syscall

ret
```

Here we move 1 to rax, write's syscall number, next we move rdi to rsi, this is because we would have received our string in that register as an argument while calling this function as userspace function, convention for this looks like this:

- rdi contains first argument
- rsi contains second argument
- rdx contains third argument
- rcx contains fourth argument
- r8 contains fifth argument
- r9 contains sixth argument

So let's say our funciton is called `func`, we could call it like this: `func(msg)` where msg will be a string containing our hello world, and it would be passed in rdi register. We take that address directly and move it to `rsi` register as second argument to write syscall.

Cool let's convert this into hex codes/binary to be hard-coded in the program so that we can write it to a memory. Run this command to get hex representation:

```sh
as base_hello_world.asm && objdump -j .text -M intel -D a.out
```


- Understanding how to get hex code repr
- Kernel APIs used and understanding flags, etc.
- Call it simply as a function, and leverage registers to pass args

## Making a change (understanding pain of just copying hex codes here)

- Make reader understand need for dynamic assembler
