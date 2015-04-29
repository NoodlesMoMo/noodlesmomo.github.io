---
title: 调用C库中的socket，背后会发生什么
author: Noodles
layout: post
permalink: /2015/04/socket-trap-kernel
categories:
  - network
tags:
  - Linux glibc
  - socket
  - kernel
  
---

------------------------------------------------------------

  前两天见到最多的一个大结构体就是`sk_buff`，数据包从本地发到网络，途径协议栈时候，其中的数据指针何时被改变，走到哪一步哪些数据有效，在LOCAL_OUT HOOK点挂上钩子，ip_hdr()返回的指针值和tcp_hdr()返回的指针地址一样，搞得七荤八素，真是有点自取其辱。
  当时就在想，用户在调用socket()时候，是如何将socket描述符与这个结构体发生关系的。协议栈工作在内核态，为了安全也为了方便，对用户态以套接口（socket）的接口形式提供给应用程序。
  书上说，标准库通过系统调用的形式陷入内核态，来获取内核服务，Linux是通过80号软中断陷入内核的。隐藏在标准库背后的细节，我想搞明白。
   
### 弱符号
  
 > 若两个或两个以上全局符号（函数或变量名）名字一样，而其中之一声明为weak symbol（弱符号），则这些全局符号不会引发重定义错误。链接器会忽略弱符号，去使用普通的全局符号来解析所有对这些符号的引用，但当普通的全局符号不可用时，链接器会使用弱符号。当有函数或变量名可能被用户覆盖时，该函数或变量名可以声明为一个弱符号。弱符号也称为weak alias（弱别名）。
 
  glibc中大量的函数使用了weak alias属性。
  

<!--more-->


  {% highlight c %}
#include <errno.h>
#include <sys/socket.h>

/* Create a new socket of type TYPE in domain DOMAIN, using
   protocol PROTOCOL.  If PROTOCOL is zero, one is chosen automatically.
   Returns a file descriptor for the new socket, or -1 for errors.  */
int
__socket (domain, type, protocol)
	 int domain;
	 int type;
	 int protocol;
{
  __set_errno (ENOSYS);
  return -1;
}

weak_alias (__socket, socket)
stub_warning (socket)
#include <stub-tag.h>
	{% endhighlight %}  

  i386平台下真正的socket函数实际定义在glibc源码目录下的 sysdeps/unix/sysv/linux/i386目录下的socket.S：
	
{% highlight c %}
#include <sysdep-cancel.h>
#include <socketcall.h>
#include <tls.h>

#define P(a, b) P2(a, b)
#define P2(a, b) a##b

	.text
/* The socket-oriented system calls are handled unusally in Linux.
   They are all gated through the single `socketcall' system call number.
   `socketcall' takes two arguments: the first is the subcode, specifying
   which socket function is being called; and the second is a pointer to
   the arguments to the specific function.

   The .S files for the other calls just #define socket and #include this.  */

#ifndef __socket
# ifndef NO_WEAK_ALIAS
#  define __socket P(__,socket)
# else
#  define __socket socket
# endif
#endif

.globl __socket
	cfi_startproc
ENTRY (__socket)
#if defined NEED_CANCELLATION && defined CENABLE
	SINGLE_THREAD_P
	jne 1f
#endif

	/* Save registers.  */
	movl %ebx, %edx
	cfi_register (3, 2)

	movl $SYS_ify(socketcall), %eax	/* System call number in %eax.  */

	/* Use ## so `socket' is a separate token that might be #define'd.  */
	movl $P(SOCKOP_,socket), %ebx	/* Subcode is first arg to syscall.  */
	lea 4(%esp), %ecx		/* Address of args is 2nd arg.  */

        /* Do the system call trap.  */
	ENTER_KERNEL

	/* Restore registers.  */
	movl %edx, %ebx
	cfi_restore (3)

	/* %eax is < 0 if there was an error.  */
	cmpl $-125, %eax
	jae SYSCALL_ERROR_LABEL

	/* Successful; return the syscall's value.  */
L(pseudo_end):
	ret


#if defined NEED_CANCELLATION && defined CENABLE
	/* We need one more register.  */
1:	pushl %esi
	cfi_adjust_cfa_offset(4)

	/* Enable asynchronous cancellation.  */
	CENABLE
	movl %eax, %esi
	cfi_offset(6, -8)		/* %esi */

	/* Save registers.  */
	movl %ebx, %edx
	cfi_register (3, 2)

	movl $SYS_ify(socketcall), %eax	/* System call number in %eax.  */

	/* Use ## so `socket' is a separate token that might be #define'd.  */
	movl $P(SOCKOP_,socket), %ebx	/* Subcode is first arg to syscall.  */
	lea 8(%esp), %ecx		/* Address of args is 2nd arg.  */

        /* Do the system call trap.  */
	ENTER_KERNEL

	/* Restore registers.  */
	movl %edx, %ebx
	cfi_restore (3)

	/* Restore the cancellation.  */
	xchgl %esi, %eax
	CDISABLE

	/* Restore registers.  */
	movl %esi, %eax
	popl %esi
	cfi_restore (6)
	cfi_adjust_cfa_offset(-4)

	/* %eax is < 0 if there was an error.  */
	cmpl $-125, %eax
	jae SYSCALL_ERROR_LABEL

	/* Successful; return the syscall's value.  */
	ret
#endif
	cfi_endproc
PSEUDO_END (__socket)

#ifndef NO_WEAK_ALIAS
weak_alias (__socket, socket)
#endif

{% endhighlight %}

   这段代码我只看懂个大概。从注释看出：所有面向系统调用的socket函数（socket，bind，listen ...）均被Linux作为异常处理。
   这些函数都是通过`socketcall`系统调用号陷入内核中。`socketcall`携带两个参数：第一个是子调用号，它用来指定哪个socket类函数被调用；第二个是socket系列函数参数的指针。
   
   这里解释一下，为什么要靠寄存器来传递参数，而不靠堆栈，毕竟寄存器的个数有限。因为这里涉及从用户态到内核态的切换，就引起堆栈的切换，这是CPU保护模式的要求.
   
   其中的条件变量哪些被实际定义这其实并不妨碍代码分析。（本来打算编译确认一下，下载的glibc源码有点旧，config阶段ld程序版本不符退出）说明一下，上面代码中的以`cfi`为前缀的宏是gdb调试时候用的。`cfi=Call Frame Information`
   一下是我查找到的关于`cfi`的资料：
   
     Debugging using DWARF
     CFI: Call Frame Information 
     CFI macros are used to generate dwarf2 unwind information for better backtraces. They don't change any code. If you find yourself in the position of combining C++ with assembly language routines, you can incorporate exception handling support by using the Call Frame Information directives. These directives generate stack unwinding information, such that any routines written in assembler will integrate nicely with C++ and other high-level languages.

     Modern ABIs don't require frame pointers to be used in functions. However missing FPs bring difficulties when doing a backtrace. One solution is to provide Dwarf-2 CFI data for each such function. This can be easily done for example by GCC in it's output, but isn't that easy to write by hand for pure assembler functions.

     With the help of these .cfi_* directives one can add appropriate unwind info into his asm source without too much trouble.

     The call frame is identified by an address on the stack. We refer to this address as the Canonical Frame Address or CFA. Typically, the CFA is defined to be the value of the stack pointer at the call site in the previous frame (which may be different from its value on entry to the current frame).
   还有一些宏我直接扩展开，不再单列。
   以上汇编转换一下为：
   
{% highlight c %} 
   .globl socket
   .align 4
   
   /*保存寄存器 ebx*/
   movl %ebx, %edx
   movl $__NR_socketcall, %eax	/*系统调用号存于 %eax*/
   movl $SOCKOP_socket, %ebx 	/*子调用号存于 %ebx*/
   lea 4(%esp), %ecx 	/*socket系列参数地址保存于 %ecx*/
   
   /*陷入内核*/
   int $0x80
   
   /*恢复寄存器*/
   movl %edx, %ebx
   
   /*判断 %eax的值是否小于0，如果小于0则表明出现错误*/
   cmpl $-125, %eax
   jae SYSCALL_ERROR_LABEL
   
   /* 成功，返回 syscall 的值*/
   ret
   
{% endhighlight %}

其中,__NR_socketcall 在内核源码相关架构的unistd.h中定义为：
 `#define __NR_socketcall 102`
 SOCKOP_socket 在glibc目录下的/sysdeps/unix/sysv/linux/socketcall.h中定义为：
 `#define SOCKOP_socket 1`
 
 汇编代码将这个宏值作为socket的调用号保存在ebx中。代码lea 4(%esp),%ecx将socket()参数地址保存在ecx中。而eax中存放102号系统调用。
 当系统执行软中断指令 int $0x80 到达系统调用的总入口 system_call()函数。此函数实现
 

  
  
  

