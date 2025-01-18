---
title: shellman 一个 Linux 堆溢出利用例题的分析
date: 2016-07-21 16:43:00
tags: []
categories: [Tech Notes]

---

## ALARM signal 自动退出程序
首先把玩一下，发现有个 alarm 事件令程序在一分钟后退出，这里在二进制修改一下时间值去掉这个小玩意，并不会影响后续堆溢出的分析。

时间值在 main 函数上作为立即数传给 alarm，用 IDA 或别的反汇编工具可以看到
一时间没有好用的二进制编辑，发现用 vi 也非常方便
https://www.zhihu.com/question/22281280/answer/34778466

![ALARM signal 时长](/img/shellman_分析/alarm.png)

之后的动态分析会用这个修改过的 shellman_noalarm 就不会一分钟后退出，看了网上一些介绍 gdb 可以屏蔽 ALARM signal， 也是可行方法吧。

<!-- truncate -->
## 用 IDA 了解关键功能和资料
** 大多数小笔记注释在 IDA 的数据库文件上

#### 新增条目
新增 shellcode，分配内存后会按输入的 size 准确地接收输入并填充

![新 shellcode 的 size](/img/shellman_分析/new_size.png)


在新增 shellcode 的这个函数里，可以发现总 shellcode 的 counter 全局变量和一个全局数组用来储存每条 record 的资料。

![全局 struct 数组](/img/shellman_分析/new_struct.png)


这个全局数组是这次利用的关键之一，从 IDA 上看还可以知道
数组起始地址：0x6016C0
新增条目时循环256 次检查有没有空位，意味着是个 size = 256 的数组
按新增条目的代码来看，结构是

```cpp
struct record {
	QWORD is_allocated;
	QWORD length;
	QWORD malloc_ptr;
};

```

![全局 struct 数组](/img/shellman_分析/new_struct2.png)


#### 删除条目
按输入的 index 从全局数组中找到 record，检查是否已分配状态，如果是就进行一段重设和 free 掉内存

#### 编辑条目
漏洞出现在这个功能里面。
获取到条目的数据地址和准备要编辑的长度后，不做任何内存重新分配直接在原地址上按新的数据长度读入数据，这里造成可以覆盖下一个 chunk 的 metadata 进而利用漏洞。

## 漏洞利用

这是典型的 unsafe_unlink 情况，参考了
https://github.com/shellphish/how2heap/blob/master/unsafe_unlink.c
http://winesap.logdown.com/posts/258859-0ctf-2015-freenote-write-up


新增3项条目，都是128 bytes 大小，因为小过128 的话 malloc 不会从 top chunk 上分割，分配出来的 chunk 就不是连续的

#### 欺骗第二项让其和假的 chunk 进行合并
编辑第一项时覆盖第二项的 metadata ，包括精心设计的 prev_size 和 inuse 标志位

按照 libc 的 P->fd->bk == P && P->bk->fd ==P 特性的检查，

P->fd == P – 24,, P->bk == P-16 (bk 和 fd 分别是第四个和第三个结构位置)

为了满足这个检测，需要一个指向假 chunk 的已知位置，所以前面说那个全局数组是这种堆溢出利用关键之处。

全局数组[0].malloc_ptr 已经指向伪造 chunk 的首地址，那按上面说的布置一下内容，把 P- 24 的值放到伪 chunk 的 bk 处，P-16 的值放到伪 chunk 的 fd 处，要覆盖的第二项里的 prev_size 本来是0x80 + 0x10 (内存大小和 metadata 大小)，现在缩小一下把 prev_size 覆盖成0x80 让第二项以为前面有0x80 bytes 空闲块。

删掉第二项触发 free –> 触发 unlink


P->fd->bk  = P->bk   // 双向链表的 node 删除，但这里P->fd->bk  就是项目一在数组上存着的地址，就是全局数组[0].malloc_ptr 的值，现在被替换成 &全局数组[0].malloc_ptr – 3 了，我们依然可以编辑项目一的内容，程序会从&全局数组[0].malloc_ptr – 3 处开始读写。

至此我们可以控制关键的全局数组的内容，达至任意地址读写。


#### 替换 free 的地址变成 system
从 IDA 或别的工具可得 free 的 GOT 地址在 0x601600, 程序跑起来后实际 free 函数地址会填充进来

![free 在 GOT 上的地址](/img/shellman_分析/free_GOT.png)

用 gdb 把程序跑起来后，用断点获取 system 地址
```
gdb-peda$ x 0x601600
0x601600 <free@got.plt>:    0x00007ffff7a91a70
gdb-peda$ b system
Breakpoint 3 at 0x7ffff7a53380
```

同一个libc 两个函数的差值是一样的

\>>> hex(0x00007ffff7a91a70 - 0x7ffff7a53380)
'0x3e6f0'

利用任意地址读写（更改数组项目的 ptr 指向，再 list 出内容或编辑回去 ），可以读出 free@GOT 的地址值，然后利用和 system 的差值固定这一特性计算 system 地址，再写回去覆盖 free 的函数地址值，至此完成函数地址替换。

#### 触发free（system）拿 shell
一时没想到 free 也是带参数的，只需要把 system 的字符串参数放在原来要 free 的地址上，其实就是 system (addr) 了，也就是说直接把”/bin/sh” 放到一个正常的条目里，这时 free 掉这个条目，就会执行 system(“/bin/sh”);


## 全局数组的变化
一开始分配了3个0x80 的情况
![一开始分配了3个0x80 的情况](/img/shellman_分析/gdb1.png)

第二项条目在被覆盖 metadata 前的状态
![第二项条目在被覆盖 metadata 前的状态](/img/shellman_分析/gdb2.png)

执行覆盖后，第二项的inuse 标志位和 prev_size 被改，而第一项里的假 chunk 也准备好了
![被覆盖后](/img/shellman_分析/gdb3.png)

执行第二项的free 触发 unlink ， 第一项的地址变成在全局数组稍前的位置（编辑它就可以控制全局数组）
![触发 unlink 后](/img/shellman_分析/gdb4.png)

![触发 unlink 后](/img/shellman_分析/gdb5.png)

此时控制全局数组让第二项变成 free@GOT, 进而读取函数地址
![全局数组的情况](/img/shellman_分析/gdb6.png)

list 功能查看 free 地址，计算system 地址可以用编辑功能替换
![使用任意读查看 free 的地址](/img/shellman_分析/gdb7.png)


最后把”/bin/sh” 放在一个正常的条目里，然后 delete 掉就会触发 system(“/bin/sh”) 
![执行 system(“/bin/sh”) 后](/img/shellman_分析/gdb8.png)


## 参考
https://www.zhihu.com/question/22281280/answer/34778466
http://www.slideshare.net/AngelBoy1/heap-exploitation-51891400 
http://drops.wooyun.org/tips/16610 
http://drops.wooyun.org/tips/7326 
https://github.com/shellphish/how2heap/blob/master/unsafe_unlink.c
http://winesap.logdown.com/posts/258859-0ctf-2015-freenote-write-up


