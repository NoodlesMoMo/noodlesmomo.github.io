---
title: Linux中IP校验和计算源码分析
author: Noodles
layout: post
permalink: /2015/04/ip-checksum/
categories:
  - 语言
tags:
  - Linux
  - TCP/IP stack
  - IP checksum analyse
  
---

### Linux 内核中关于IP校验和计算的源码实现分析

------------------------------------------------------------

  前段时间简单学习了一下Netfilter框架，尝试写一个内核模块，将本地发出的特定数据包改造蹂躏一下，这种感觉挺爽。
  这其中涉及到校验和的计算，开始阅读内核中网络协议栈的东西。其中的收获很多，太多的Hacking技巧值得学习和记录。
  内核的东西，只要你去想，总是会感到它的实现之美。

  
  首先贴一张IP报文协议图，内核的相关代码与此图密不可分。
  <center><img src="/images/study/ip_checksum/ip.jpg"></img></center>
  

<!--more-->

  
  内核中ip头部结构定义如下：
  
  {% highlight c %}
  
	  struct iphdr {
	#if defined(__LITTLE_ENDIAN_BITFIELD)
		__u8	ihl:4,
			version:4;
	#elif defined (__BIG_ENDIAN_BITFIELD)
		__u8	version:4,					// 4位版本号
			ihl:4;							// 4位首部长度，！以4字节为单位！！！
	#else
	#error	"Please fix <asm/byteorder.h>"
	#endif
		__u8	tos;						// 8位服务类型
		__be16	tot_len;					// 16位总长度
		__be16	id;							// 16位标识
		__be16	frag_off;					// 3位标志 + 13位片偏移
		__u8	ttl;						// 8位TTL值
		__u8	protocol;					// 8位协议号
		__sum16	check;						// 16位校验和
		__be32	saddr;						// 32位源IP
		__be32	daddr;						// 32位目的IP
		/*The options start here. */
	};
	
	{% endhighlight %}
	  
  TCP/IP协议IP校验和计算方法：
  
  **发送数据**
  
	1. 先将IP头部中校验和字段置0
	2. 将首部以16位为单位累（如果总字节数为奇数，最后一个字节单独相加），累加结果保存在一个32位数值中
	3. 累加完毕，将结果中的高16位累加到低16位上，重复这一过程直到高16位全为0

  **接收数据**
  
    接收方将IPv4分组头部中的校验和字段与其他字段一视同仁，也按16位累加。如果最后结果为`0xffff`，则校验正确。
	
  **几点补充**
  
    1. IP协议头部-首部长度-字段长度单位为4字节。所以多数IPv4报文此字段值为5，代表IP报文长度为20字节。
	2. IPv4报文首部校验和仅计算IP首部，不包括携带的数据。
	3. TCP和UDP数据包，其头部也包含16位校验和，校验算法与IPv4分组头部完全一致，但参与校验的数据不同。L4层参与校验的除了首部还有用户数据部分。
	
  Linux中，校验和计算多数使用了内联汇编，因为它需要足够的快，这是有必要的：IP头部可能很短，参与计算的
  数据就几十个，但如果是TCP和UDP校验和计算，传送一个高清电影，参与计算的字节数起码上G。另外，Linux没有
  按16bit进行计算，而是采用32bit或者64bit计算，这样处理，还是为了高效。
  
  内核里完成IP校验和计算函数为 `ip_fast_csum` ,它在源码树的arch/x86/um/asm/Checksum.h中，具体实现和注释为：

    {% highlight c %}
	/**
	 * ip_fast_csum - Compute the IPv4 header checksum efficiently.
	 * iph: ipv4 header
	 * ihl: length of header / 4
	 */
	static inline __sum16 ip_fast_csum(const void *iph, unsigned int ihl)
	{
		unsigned int sum;

		asm(	"  movl (%1), %0\n"		; // 先将 IP 首部开始的4个字节放到sum中
			"  subl $4, %2\n"			; // ihl = ihl-4
			"  jbe 2f\n"				; // 如果ihl <= 0，则前跳到2处结束
			"  addl 4(%1), %0\n"		; // sum 加上IP头部第2个32位值
			"  adcl 8(%1), %0\n"		; // 带进位将sum 加上IP头部第3个32位值
			"  adcl 12(%1), %0\n"		; // 带进位将sum 加上IP头部第4个32位值
			"1: adcl 16(%1), %0\n"		; // 带进位将sum 加上IP头部第5个32位值
			"  lea 4(%1), %1\n"			; // iph指针向下偏移4字节
			"  decl %2\n"				; // ihl自减(--ihl)
			"  jne	1b\n"				; // 如果iph不为0则前跳到1处循环，也即以后每次都只将接下来的32位累加到sum中
			"  adcl $0, %0\n"			; // 累加完了IP头部所有字节之后，再将进位累加到sum中(当然，进位标志有可能为0)
			"  movl %0, %2\n"			; // ipl = sum
			"  shrl $16, %0\n"			; // 因为IP校验和是16位的，所以此处为： sum = sum >> 16;
			"  addw %w2, %w0\n"			; // w表示计算16bit值，%w2表示将iph的低16位(也即原sum的低16位)与上一步中计算的sum高16位相加
			"  adcl $0, %0\n"			; // 期间可能产生的进位也因这步加到sum中
			"  notl %0\n"				; // 对sum取反(sum ~= sum)
			"2:"
		/* Since the input registers which are loaded with iph and ipl
		   are modified, we must also specify them as outputs, or gcc
		   will assume they contain their original values. */
		: "=r" (sum), "=r" (iph), "=r" (ihl)
		: "1" (iph), "2" (ihl)
		: "memory");
		return (__force __sum16)sum;
	　}

    {% endhighlight %}
	
  代码比较有意思的地方是：内核对IPv4头部的前5个字节采用4个显得比较笨的add指令？完全可以从开始中取出
  长度字段的值之后就进入循环。
  内核这样实现就有他实现的道理，我认为其中重要的原因就是多数IP数据包选项为0，也就是头部大部分是20个
  字节的，内核将20个字节逐条指令相加，这样可使处理器形成流水线操作，代码命中率比较高。如果IP头部长度
  不是通常的20个，那再进入循环中去处理。
  
  
  

