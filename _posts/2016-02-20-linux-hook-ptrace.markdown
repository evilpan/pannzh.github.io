---
layout: post
title:  "Linux Hook 笔记"
date:   2016-02-21 18:42:00
comments: true
categories: Tech
---

相信很多人对"Hook"都不会陌生,其中文翻译为"钩子".在编程中,
钩子表示一个可以允许编程者插入自定义程序的地方,通常是打包好的程序中提供的接口.
比如,我们想要提供一段代码来分析程序中某段逻辑路径被执行的频率,或者想要在其中
插入更多功能时就会用到钩子. 钩子都是以固定的目的提供给用户的,并且一般都有文档说明.
通过Hook,我们可以暂停系统调用,或者通过改变系统调用的参数来改变正常的输出结果,
甚至可以中止一个当前运行中的进程并且将控制权转移到自己手上.

## 基本概念

操作系统通过一系列称为系统调用的方法来提供各种服务.他们提供了标准的API来访问下面的
硬件设备和底层服务,比如文件系统. 以32位系统为例,当进程运行系统调用前,会先把系统调用号放到寄存器
`%eax`中,并且将该系统调用的参数依次放入寄存器`%ebx, %ecx, %edx 以及 %esi 和 %edi`中.
以write系统调用为例:

    write(2, "Hello", 5);

在32位系统中会转换成:

    movl   $1, %eax
    movl   $2, %ebx
    movl   $hello,%ecx
    movl   $5, %edx
    int    $0x80


其中`1`为write的系统调用号, 所有的系统调用号码定义在`unistd.h`文件中. $hello表示字符串
"Hello"的地址; 32位Linux系统通过0x80中断来进行系统调用.

如果是64位系统则有所不同, 用户层应用层用整数寄存器` %rdi, %rsi, %rdx, %rcx, %r8 以及 %r9`来传参,
而**内核接口**用`%rdi, %rsi, %rdx, %r10, %r8 以及 %r10`来传参. 并且用`syscall`指令而不是80中断
来进行系统调用. 相同之处是都用寄存器`%rax`来保存调用号和返回值.


更多关于32位和64位汇编指令的区别可以参考[stack overflow的总结][x64-asm],
因为我当前环境是64位Linux,所以下文的操作都以64位系统为例.


## 进程追踪

上面说到钩子一般由程序提供,那么操作系统内核作为一个程序,是否有提供相应的钩子呢?
答案是肯定的, `ptrace`(Process Trace)系统调用就提供了这样的功能. ptrace提供了许多
方法来观察和控制其他进程的执行, 并且可以检查和修改其内核镜像和寄存器. 通常用来
作为调试器(如gdb)或用来跟踪各种其他系统调用.


那么,ptrace在程序运行的哪个阶段起作用呢? 答案是在执行系统调用之前. 内核会先检查是否
进程正在被追踪, 如果是的话, 内核会停止进程并将控制权转移给追踪进程, 因此其可以查看和
修改被追踪进程的寄存器. 举例说明: 

```
#include <stdio.h>
#include <unistd.h>
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/reg.h>   /* For constants ORIG_RAX etc */
int main()
{   pid_t child;
    long orig_rax;
    child = fork();
    if(child == 0) {
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        execl("/bin/ls", "ls", NULL);
    }
    else { wait(NULL);
        orig_rax = ptrace(PTRACE_PEEKUSER,
                          child, 8 * ORIG_RAX,
                          NULL);
        printf("The child made a "
               "system call %ld\n", orig_rax);
        ptrace(PTRACE_CONT, child, NULL, NULL);
    }
    return 0;
}
```

程序编译运行后输出:

    The child made a system call 59

以及`ls`的结果. 系统调用号59是`__NR_execve`, 由子进程调用的`execl`产生.

在上面的例子中我们可以看见, 父进程fork了一个子进程,并且在子进程中进行系统调用. 
在执行调用前,子进程运行了ptrace,并设置第一个参数为`PTRACE_TRACEME`, 这告诉内核
当前进程正在被追踪. 因此当子进程运行到execl时, 会把控制权转回父进程. 父进程用wait
函数(系统调用)来等待内核通知. 然后就可以查看系统调用的参数以及做其他事情.

当系统调用出现的时候, 内核会保存原始的rax寄存器值(其中包含系统调用号), 我们可以
从子进程的`USER`段读取这个值, 这里是使用ptrace并且设置第一个参数为`PTRACE_PEEKUSER`.

当我们检查完了系统调用之后, 可以调用ptrace并设置参数`PTRACE_CONT`让子进程继续运行. 
值得一提的是, 这里的child为子进程的进程ID, 由fork函数返回.

## 寄存器读写

ptrace函数通过四个参数来调用, 其原型为:

    long ptrace(enum __ptrace_request request,
                pid_t pid,
                void *addr,
                void *data);

其中第一个参数决定了ptrace的行为以及其他参数的含义, request的值可以是下列值中的一个:

    PTRACE_TRACEME, PTRACE_PEEKTEXT, PTRACE_PEEKDATA, PTRACE_PEEKUSER, PTRACE_POKETEXT, 
    PTRACE_POKEDATA, PTRACE_POKEUSER, PTRACE_GETREGS, PTRACE_GETFPREGS, PTRACE_SETREGS, 
    PTRACE_SETFPREGS, PTRACE_CONT, PTRACE_SYSCALL, PTRACE_SINGLESTEP, PTRACE_DETACH.

在系统调用追踪中, 常见的流程如下图所示:

![ptrace](http://images2015.cnblogs.com/blog/676200/201602/676200-20160220161141175-432928373.png)

### 读取系统调用参数

系统调用的参数按顺序存放在rbx,rcx...之中,因此以write系统调用为例看如何读取寄存器的值:

```
#include <sys/ptrace.h>
#include <sys/wait.h>
#include <sys/reg.h>   /* For constants ORIG_EAX etc */
#include <sys/user.h>
#include <sys/syscall.h> /* SYS_write */
int main() {
    pid_t child;
    long orig_rax;
    int status;
    int iscalling = 0;
    struct user_regs_struct regs;

    child = fork();
    if(child == 0) {
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        execl("/bin/ls", "ls", "-l", "-h", NULL);
    } else {
        while(1) {
            wait(&status);
            if(WIFEXITED(status))
                break;
            orig_rax = ptrace(PTRACE_PEEKUSER,
                              child, 8 * ORIG_RAX,
                              NULL);
            if(orig_rax == SYS_write) {
                ptrace(PTRACE_GETREGS, child, NULL, &regs);
                if(!iscalling) {
                    iscalling = 1;
                    printf("SYS_write call with %lld, %lld, %lld\n",
                            regs.rdi, regs.rsi, regs.rdx);
                }
                else {
                    printf("SYS_write call return %lld\n", regs.rax);
                    iscalling = 0;
                }
            }
            ptrace(PTRACE_SYSCALL, child, NULL, NULL);
        }
    }
    return 0;
}
```

编译运新有如下输出:

```
SYS_write call with 1, 140693012086784, 10
total 32K
SYS_write call return 10
SYS_write call with 1, 140693012086784, 45
-rwxr-xr-x 1 lxy lxy  13K Feb 21 12:19 a.out
SYS_write call return 45
SYS_write call with 1, 140693012086784, 46
-rw-r--r-- 1 lxy lxy 1.5K Feb 20 20:52 test.c
SYS_write call return 46
SYS_write call with 1, 140693012086784, 53
-rw-r--r-- 1 lxy lxy 5.0K Feb 21 12:19 trace_write.c
SYS_write call return 53
```

可以看到我们的`ls -l -h`命令中, 发生了四次write系统调用.这里读取寄存器的时候可以用之前
的`PTRACE_PEEKUSER`参数,也可以直接用`PTRACE_PEEKUSER`参数将寄存器的值读取到结构体`user_regs_struct`,
该结构体定义在`sys/user.h`中.

程序中WIFEXITED函数(宏)用来检查子进程是被ptrace暂停的还是准备退出, 可以通过`wait(2)`的man page
查看详细的内容. 其中还有个值得一提的参数是`PTRACE_SYSCALL`,其作用是使内核在子进程进入和
退出系统调用时都将其暂停, 等价于调用`PTRACE_CONT`并且在下一个`entry/exit`系统调用前暂停.

### 修改系统调用参数

假设我们现在要修改write系统调用的参数从而修改打印的内容,根据文档可知,其第二个参数为write字符串的地址,
第三个参数为字符串的字节数,因此我们可以用:

        val = ptrace(PTRACE_PEEKDATA, child, addr, NULL);

来得到字符串的内容. 值得一提的是, 由于ptrace的返回值是long型的,因此一次最多只能读取sizeof(long)个字节 的数据,可以多次读取`addr + i*sizeof(long)`然后合并得到最终的字符串内容. 在64bit系统下一次可以读取64/8=8字节的数据.

修改字符串后,可以用:

        ptrace(PTRACE_POKEDATA, child, addr, data);

来更新系统调用参数. 同样一次只能更新8字节,因此需要分多次将结果放到long型的data里,再按顺序更新到`addr + i*sizeof(long)`中.
一个读取参数字符串值的例子如下:

```
#define long_size  sizeof(long);
void getdata(pid_t child, long addr,
             char *str, int len) {   
    char *laddr;
    int i, j;
    union u {
            long val;
            char chars[long_size];
    }data;
    i = 0;
    j = len / long_size;
    laddr = str;
    while(i < j) {
        data.val = ptrace(PTRACE_PEEKDATA,
                          child, addr + i * 8,
                          NULL);
        if(data.val == -1)
            if(errno) {
                printf("READ error: %s\n", strerror(errno));
            }
        memcpy(laddr, data.chars, long_size);
        ++i;
        laddr += long_size;
    }
    j = len % long_size;
    if(j != 0) {
        data.val = ptrace(PTRACE_PEEKDATA,
                          child, addr + i * 8,
                          NULL);
        memcpy(laddr, data.chars, j);
    }
    str[len] = '\0';
}
```

值得一提的是union类型可以用来很方便地往64bit寄存器(long型)读写和转换其他类型(如char)格式的数据.

## 追踪其他程序的进程

上面举的例子都是追踪并修改声明了*PTRACE_TRACEME*的子进程的,那么我们能否追踪其他独立的正在运行的进程呢?
使用`PTRACE_ATTACH`参数就可以追踪正在运行的程序:

    ptrace(PTRACE_ATTACH, pid, NULL, NULL)

其中pid位想要追踪的进程的进程id. 当前进程会给被追踪进程发送**SIGSTOP**信号,但不要求立即停止,
一般会等待子进程完成当前调用. ATTACH之后就和操作fork出来的TRACEME子进程一样操作就好了.
如果要结束追踪,则再调用`PTRACE_DETACH`即可.

### 动态注入指令

用过gdb等调试器的人都知道,debugger工具可以给程序打断点和单步运行等. 这些功能其实也能用ptrace实现,
其原理就是ATTACH并追踪正在运行的进程, 读取其指令寄存器IR(32bit系统为%eip, 64位系统为%rip)的内容,
备份后替换成目标指令,再使其返回运行;此时被追踪进程就会执行我们替换的指令. 运行完注入的指令之后,
我们再恢复原进程的IR,从而达到改变原程序运行逻辑的目的. talk is cheap, 先写个循环打印的程序:

```
//victim.c
int main() {
    while(1) {
        printf("Hello, ptrace! [pid:%d]\n", getpid());
        sleep(2);
    }
    return 0;
}
```
程序运行后会每隔2秒会打印到终端.然后再另外编写一个程序:

```
//attach.c
int main(int argc, char *argv[]) {
    if(argc!=2) {
        printf("Usage: %s pid\n", argv[0]);
        return 1;
    }
    pid_t victim = atoi(argv[1]);
    struct user_regs_struct regs;
    /* int 0x80, int3 */
    unsigned char code[] = {0xcd,0x80,0xcc,0x00,0,0,0,0};
    char backup[8];
    ptrace(PTRACE_ATTACH, victim, NULL, NULL);
    long inst;

    wait(NULL);
    ptrace(PTRACE_GETREGS, victim, NULL, &regs);
    inst = ptrace(PTRACE_PEEKTEXT, victim, regs.rip, NULL);
    printf("Victim: EIP:0x%llx INST: 0x%lx\n", regs.rip, inst);

    /* Copy instructions into a backup variable */
    getdata(victim, regs.rip, backup, 7);
    /* Put the breakpoint */
    putdata(victim, regs.rip, code, 7);
    /* Let the process continue and execute the int 3 instruction */
    ptrace(PTRACE_CONT, victim, NULL, NULL);

    wait(NULL);
    printf("Press Enter to continue ptraced process.\n");
    getchar();
    putdata(victim, regs.rip, backup, 7);
    ptrace(PTRACE_SETREGS, victim, NULL, &regs);

    ptrace(PTRACE_CONT, victim, NULL, NULL);
    ptrace(PTRACE_DETACH, victim, NULL, NULL);
    return 0;
}
```
运行后会将一直循环输出的进程暂停, 再按回车使得进程恢复循环输出. 其中putdata和getdata在上文中已经介绍过了.
我们用之前替换寄存器内容的方法,将%rip的内容修改为`int 3`的机器码, 使得对应进程暂停执行; 
恢复寄存器状态时使用的是`PTRACE_SETREGS`参数. 值得一提的是对于不同的处理器架构, 其使用的寄存器名称
也不尽相同, 在不同的机器上允许时代码也要作相应的修改.

这里注入的代码长度只有8个字节, 而且是用shellcode的格式注入, 但实际中我们可以在目标进程中动态加载库文件(.so),
包括标准库文件(如libc.so)和我们自己编译的库文件, 从而可以通过传递函数地址和参数来进行复杂的注入,限于篇幅暂不细说.
不过需要注意的是动态链接库挂载的地址是动态确定的, 可以在`/proc/$pid/maps`文件中查看, 其中$pid为进程id.

## 参考资料

- [playing with ptrace part I][playing-ptrace1]
- [playing with ptrace part II][playing-ptrace2]
- [安卓动态调试之Hook][android-hook]

博客地址:

- [pannzh.github.io](http://pannzh.github.io)
- [有价值炮灰-博客园](http://www.cnblogs.com/pannengzhi/)

欢迎交流,文章转载请注明出处.

[x64-asm]:http://stackoverflow.com/questions/2535989/what-are-the-calling-conventions-for-unix-linux-system-calls-on-x86-64
[playing-ptrace1]:http://www.linuxjournal.com/article/6100
[playing-ptrace2]:http://www.linuxjournal.com/article/6210
[android-hook]:http://drops.wooyun.org/tips/9300
