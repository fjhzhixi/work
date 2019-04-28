# lab4做了什么

## 支持缺页中断处理

## 完成基本的系统调用

我认为系统调用的实质就是用户态的进程想使用一些内核态下的功能,比如分配内存空间,创建进程等等,但是由于操作系统要保障**安全性**,所以这些接口不能向用户程序开放,所以采用一个折中的方法,即**系统调用**,用户将一些必要的信息以**参数的形式**传递给内核,内核判断你的操作是否**合法**,合法则执行并将结果返回给用户,否则拒绝执行

## 实现fork

`fork`就是**以父进程为模板**创建一个与其**高度相似**的子进程,其关键的一点特性是**父子进程中的`fork`返回值不同**,这也是区分父子进程的方法

## 实现进程间通信

通信本质上就是**在进程间传递一个数值**, 通过**共享内存**来实现, 共享的是所有进程都存在的内核中的2G空间中的一个位置

(搞懂再写)

# 提前准备的知识

## MIPS下C与汇编的参数传递

首先介绍两个关于MIPS汇编的宏定义 

```C
//include/asm/asm.h
// 用来定义全局叶子汇编函数 : 即在函数内部不需要调用其他函数
#define LEAF(symbol)                                    \
                .globl  symbol;                         \
                .align  2;                              \
                .type   symbol,@function;               \
                .ent    symbol,0;                       \
symbol:         .frame  sp,0,ra
// 用来定义全局非叶子汇编函数 : 即在函数内部还调用其他函数 
// 所以在该宏定义结构中已经做了对栈指针的处理
#define NESTED(symbol, framesize, rpc)                  \
                .globl  symbol;                         \
                .align  2;                              \
                .type   symbol,@function;               \
                .ent    symbol,0;                       \
symbol:         .frame  sp, framesize, rpc
// 函数结尾
#define END(function)                                   \
                .end    function;                       \
                .size   function,.-function
```

**注意我们一般而言只有在接近底层时才有使用到汇编来实现,所以大部分汇编功能都能通过叶子函数完成**

1. C函数之间的调用 : 

   所有的参数传递以及返回值得维护由编译器隐式的实现,不需要考虑

2. MIPS汇编之间的调用

   * 调用者需要在代码中显式的在栈中保存受保护寄存器,返回值寄存器等,然后显式的向`$a0-$a3`中写入参数值,然后调用,返回时要恢复现场

   * 被调用者需要显式的获取参数

   * 参数较多时通过栈传递

   其实就是**调用者与被调用者遵守并显式的实现统一的调用规则**

3. **C调用MIPS汇编函数** : 最常见的一种情况

   * 调用者(即C函数) : 
     1. 在栈上创建一个容纳参数的空间,从`sp`指向的位置开始,第一个参数(即C源码中最左侧的参数)位于最低地址处,**每个参数至少占据一个字的大小空间**
     2. **为任何一个调用都至少分配16字节的栈参数空间,即使没有这么多参数**
     3. **实际上优先通过的是寄存器传递参数,即参数结构的前16个字节(即4个字)保存在`$a0-$a3`的寄存器结构中,而栈中的前16个字节的内容未定义,但是其结构必须保存**
   * 被调用者(MIPS函数)
     1. 可以选择盲目的将`$a0-$a3`的写入栈中,也可以不写(取决于该汇编函数的功能)
     2. **前4个参数从寄存器获得,之后的参数从栈中获得**

4. MIPS汇编函数调用C函数

## 异常处理流程



# 走进lab4

## 系统调用的基本流程

### 宏观上看一个系统调用的过程

1. 调用一个封装好的用户空间的库函数
2. 调用用户空间的`syscall_*` 函数
3. 调用`msyscall`，用于陷入内核态
4. 陷入内核，内核取得信息，执行对应的内核空间的系统调用函数（`sys_*`）
5. 执行系统调用，并返回用户态，同时将返回值传递回用户态
6. 从库函数返回，回到用户程序调用处

### 代码细节

1. 用户空间的行为 : 

   1. 调用`syscall_*`函数,等待返回值(如果有的话)

      **用户空间不负责任何实质的处理,只是准备好参数之后陷入内核态即可**

      **所有的系统调用都是通过`mysyscall`陷入内核,通过第一个参数作为系统调用号来区别不同的系统调用种类**

      ````c
      //user/syscall_lib.c
      void syscall_yield(void)
      {
      	msyscall(SYS_yield,0,0,0,0,0);
      }
      void syscall_env_destroy(u_int envid)
      {
      	msyscall(SYS_env_destroy,envid,0,0,0,0);
      }
      int syscall_set_pgfault_handler(u_int envid, u_int func, u_int xstacktop)
      {
      	return msyscall(SYS_set_pgfault_handler,envid,func,xstacktop,0,0);
      }
      int syscall_mem_alloc(u_int envid, u_int va, u_int perm)
      {
      	return msyscall(SYS_mem_alloc,envid,va,perm,0,0);
      }
      int syscall_mem_map(u_int srcid, u_int srcva, u_int dstid, u_int dstva, u_int perm)
      {
      	return msyscall(SYS_mem_map,srcid,srcva,dstid,dstva,perm);
      }
      ````

      即用户准备好必要的信息,然后**包装一层**在将第一个参数设置为**系统调用号**,然后内核根据系统调用号的值(**偏移**)决定**真正执行功能的函数的入口地址**

      这些系统调用号以一个注册表的形式定义在头文件中 :

      ```c
      //include/unistd.h
      #ifndef UNISTD_H
      #define UNISTD_H
      
      #define __SYSCALL_BASE 9527
      #define __NR_SYSCALLS 20
      #define SYS_putchar 		((__SYSCALL_BASE ) + (0 ) ) 
      #define SYS_getenvid 		((__SYSCALL_BASE ) + (1 ) )
      #define SYS_yield			((__SYSCALL_BASE ) + (2 ) )
      #define SYS_env_destroy		((__SYSCALL_BASE ) + (3 ) )
      #define SYS_set_pgfault_handler	((__SYSCALL_BASE ) + (4 ) )
      #define SYS_mem_alloc		((__SYSCALL_BASE ) + (5 ) )
      #define SYS_mem_map			((__SYSCALL_BASE ) + (6 ) )
      #define SYS_mem_unmap		((__SYSCALL_BASE ) + (7 ) )
      #define SYS_env_alloc		((__SYSCALL_BASE ) + (8 ) )
      #define SYS_set_env_status	((__SYSCALL_BASE ) + (9 ) )
      #define SYS_set_trapframe		((__SYSCALL_BASE ) + (10 ) )
      #define SYS_panic			((__SYSCALL_BASE ) + (11 ) )
      #define SYS_ipc_can_send		((__SYSCALL_BASE ) + (12 ) )
      #define SYS_ipc_recv		((__SYSCALL_BASE ) + (13 ) )
      #define SYS_cgetc			((__SYSCALL_BASE ) + (14 ) )
      #endif
      ```

   2. 调用`msyscall`陷入内核态

      ```c
      //user/syscall_wrap.S
      LEAF(msyscall)	//说明该函数是一个全局叶子函数,不需要调用其他函数
      sw	a0,0(sp)
      sw	a1,4(sp)
      sw	a2,8(sp)
      sw	a3,12(sp)
      move	v0, a0
      syscall
      jr	ra
      END(msyscall)
      ```

      





# 问题

1. fork.c中的duppage传递的父进程号不一定是0吧
2. 

