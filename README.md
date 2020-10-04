# Malloc Internals  
本文是[MallocInternals](https://sourceware.org/glibc/wiki/MallocInternals)的译文。  

## `malloc`概览  
GNU C运行库，也就是glibc，提供一些便捷的函数来管理应用被分配到的内存。glibc的malloc由ptmalloc（pthreads malloc）衍生而来，而后者又由dlmalloc（Doug Lea malloc）衍生而来。本malloc是一款采用heap style（堆风格）的分配器。与采用bitmap（位图）、数组或region（每个region内含有多个同样大小的内存block）（1）的实现不同，它管理一个或多个连续的内存区域（即heap或“堆”），而每个heap被分割成不同大小的内存chunk并组织起来。在以前，每个进程（）只有一个heap，而如今glibc的malloc允许一个进程有多个heap，每个都在其持有的地址空间内增长*。  
因此，在本文中我们有以下通用的术语：  
  
**Arena（分配区）**  
    它是一个被一个或多个线程共享的数据结构，其包括一个多个堆，也包括由链表组织的空闲的内存chunk。每个线程会被指定从特定的分配区的free lists*分配到内存.  
**Heap（堆）**  
    一块连续的内存区域，被分割成chunk以供被分配。每个heap属于且仅属于一个heap。  
**Chunk（内存块）**  
    一小段可以通过`malloc()`分配给用户程序使用，又可以通过`free()`返还给malloc的内存。它还可能与相邻的chunk合并成更大的内存块。注意，chunk是一个包装（wrapper），它包装了一个返回给用户程序使用的内存块。每个chunk只能存在于一个heap内，以此类推其也只能属于某一特定的heap。  
**Memory（内存）**  
    进程可用的地址空间的一部分，它通常从物理内存或交换空间中来。  
值得一提的是，本文中，我们提到“内存”为泛指——然而，glibc的malloc中存在与Linux或其他操作系统内核交互的代码，意味着“内存”是由操作系统的资源通过映射得到的，这种资源可以被返还给操作系统。这种“物理内存”和“虚拟内存”的区别与本文讨论的内容无关，除非特地提及。  
  
*译注：在内存分配器算法语境下的region有特别意义？？补充论文引用，还没看论文就先不自作主张翻译*  
*译注：传统的计算机教学中，通常程序把.bss和stack之间的内存全部称为堆，但实际上这部分内存是由操作系统分批分配给进程的，且多次申请得到的内存区域未必连续。*  

**什么是chunk？**  
glibc的malloc是面向chunk设计的（chunk-oriented）。它把一大块内存（heap）分割成大小不一的大量chunk。每个chunk都带有一些元数据，这些元数据包括该chunk可供用户程序使用的内存的大小（通过chunk头部的size字段得到）和相邻chunk的地址。当一个chunk被分配给用户程序使用时，唯一被记录的元数据就是chunk的大小（）；当一个chunk被通过`free()`释放时，其中原用于存放用户数据的部分被重用于记载额外的arena相关信息，比如链表指针，以便大小合适的chunk能快速被找到并被重用。同样地，被释放的chunk的最后一个字（）被用作chunk大小数值的副本（其3个最低有效位被置为0，而头部的size字段则将这三个位用于元数据）。  
在malloc库中，一个chunk指针，或一个`mchunkptr`并不指向chunk的开始，而是指向其上一个chunk的最后一个字（也就是其尾部的size副本字段）。换句话说，除非上一个块是空闲块，否则mchunkptr的第一个字段是不可用的。  
由于所有的块都是8的倍数个字节，chunk size字段的3个最低有效位被用作flag。这三个flag分别定义如下：  
  
**A(0x04)**  
    用于指示其来自哪个arena。在malloc中，main arena直接使用本进程的heap（），其他arena使用调用`mmap()`得到的heap。为了将一个chunk映射到其隶属于的heap，malloc需要知道这个chunk是以上两种情况中的哪一种。如果该位为1，那么当前chunk则来自`mmap()`得到的内存，且隶属于的heap的位置可以通过本chunk的地址计算得到。  
**M(0x02)**  
    用于指示该chunk是否直接来自调用`mmap()`，且不属于任何一个heap。  
**P(0x01)**  
    用于指示之前的chunk是否已被分配。如果该位为1，则前一个相邻的chunk正被用户使用，因此当前chunk的`prev_size`字段不可用。注意，某些chunk即使空闲仍会将此位置1，比如fastbins（见下文）中的chunk。因此此位真正的意义是分辨前一个相邻chunk是否可以进行合并——如果此位置1，那么前相邻块要么已被分配，要么处于malloc优化的部分中。  
  
为了确保chunk的有效负荷区足以容纳malloc所需的额外信息，一个chunk的最小规模为`4 * sizeof(void *)`（除非`size_t`和`void *`不一样大）（）。并且，若编译平台ABI要求额外的内存对齐，最小规模可能还会更大。`prev_size`字段并没有计入chunk的大小中，因为当chunk较小时`fd_nextsize`和`bk_nextsize`指针不会被启用，而chunk较大时其尾部则有充足空间供记载额外信息。  
  
![Image of struct malloc_chunk]
(MallocInternalImages/struct_malloc_chunk.png)  
  
由于chunk在内存中彼此相邻，如果用户知道某个heap中第一个chunk（地址最低的那个）的地址，用户可以使用chunk中的size信息递增地址来遍历该heap中所有chunk，尽管这种办法难以察觉到何时到达heap的最后一个chunk。  
从`mmap()`得到的heap总是被对齐到2的幂的地址上。因此，当一个chunk属于`mmap()`得到的heap中时（换句话说，它的A位flag被置为1），可以基于该chunk的地址计算得到所属heap的`heap_info`字段的地址。  
？？图片

*译注：`free()`的API只提供内存地址而未提供内存大小，因此malloc需要额外的簿记信息来管理内存*  
*译注：原文为word，准确地说应是“用于表示chunk大小的整形”的大小，默认情况下可简单粗暴地理解为sizeof(size_t)。但个其实是属于malloc可以自定义的部分，比如malloc可以被调成使用4字节大小的size_t却使用8字节大小的指针*  
*译注：即紧接着数据段向高地址扩展的部分*  
*译注：所以准确地应为`2*sizeof(size_t) + 2*sizeof(pointer_t)`*  
  
Arenas与Heaps  
为了有效地应付多线程程序，glibc的malloc允许同时使用多个内存区域。因此，不同的线程可以访问不同的内存区域而不干扰彼此。这些内存区域统称为“arena”。malloc中存在一个“main arena”，它对应进程最初的heap。malloc使用一个静态变量指向这个区域，且每个arena都有一个`next`指针指向其后新增的arena。  
  
由于线程冲突的压力与日俱增，除main arena外的arena都是通过`mmap()`获取以降低压力。（在64位系统上）arena的最大数量是CPU个数的8倍（除非用户通过`mallopt`特别指定），这意味着使用大量线程的进程仍然会有竞争arena的使用权，但换取的好处是内存碎片化的程度更小。  
  
每个arena结构体都有一个互斥锁来控制对arena的访问。当然，部分arena操作是不需要获取锁的。比如，访问fastbins可以通过原子操作完成，不需要对整个arena上锁。线程间对互斥锁的竞争是创建多个arena的初衷，被分配给不同arena线程不需等待彼此。如果遇到竞争，线程可以自动切换到未上锁的arena。  
  
每个arena从一个或多个heap中获取内存。Main arena使用进程的初始heap（从.bss段结束处开始的heap）。额外创建的arena在当前持有的heap已经消耗殆尽时，会通过`mmap()`（从操作系统处）获得更多的内存来扩充它们的heap。每个arena都会跟踪一个名为top的特别chunk，它通常是最大的可用chunk，且指向最近（从操作系统处）获得的内存。  
  
从操作系统处通过`mmap()`获得初始内存的arena会将这块内存作为它初始的heap，并从中向用户分配内存：  
？？图片  
  
每个arena中的chunks要么被分配给用户程序使用，要么处于被释放的状态。Arena不会跟踪被用户程序使用中的chunks的情况。基于大小和使用历史，被释放的chunk被存放在多种不同的列**表中，以便未来高效地满足内存请求。这种列表被称为“bins”，以下对各种bin进行介绍：  
**Fast**  
    Small chunks（）被存放在其对应size的bin中——*意思是fastbins和smallbins（见下文）都是把相同size的small chunks放到同一条链表里。（malloc定义small的范围是[0, 512 bytes]）。它们之间的第一个区别是，Fastbins内的chunks的来源是刚被`free()`且大小在[0, 128 bytes]的chunks（）；而smallbins的chunks来自于unsorted bin（见下文）。*。加入到某一fastbin中的chunk不会与其相邻的chunk合并——fastbin的操作逻辑被最简化以保持快速访问（bin如其名）。Fastbins内的chunks会在需要时被移动到其他bin里去（什么时候？）。Fastbins的chunks以单向链表的形式存放，因为同一个fastbin内的chunks的大小相同，且不会从空闲链表中间取出chunks——*即fastbins像栈一样遵循FILO的逻辑，每次都把刚释放的对应size的chunk作为新的头结点，而分配时也是把某fastbin的第一个chunk分配给用户（如果有的话），这样可以获得更好的缓存本地性（Locality），因为刚释放的内存很可能还在CPU缓存里。*  
**Unsorted**  
    当一个超出`MAX_FAST_SIZE`（该宏定义了fastbins能响应的最大内存请求）的chunks被`free()`时，它会首先被放在unsorted bin内，且进程内的unsorted bin只有一个。malloc会给unsorted bin内的chunks一次被重用机会，若这些chunks未能满足下一次内存请求（*unsorted bin响应的请求超过small sizes，但又不需要直接调用`mmap()`*），则将unsorted bin内的chunks放到它们相应size的bin里面去，这一步称为sort（见下文smallbins中的叙述）。  
**Small**  
    普通size的bins被分为“small”和“large” bins，前者中每一个bin内chunks的大小一致，后者则不一致（见下文）。当一个chunk被添加进small或large bin前，它们首先会尽可能地与相邻的空闲chunk（*回忆前文，空闲chunk的定义是该chunk的后一个相邻chunk的size字段最低有效位为0*）合并，直到不再与空闲chunk相邻。因此，添加进任意一个small或large bin的chunk只会与1)用户程序正在使用的chunks，或2)fast chunks，或3)unsorted chunks相邻。small和large bins都以双向循环链表的形式组织，但是smallbins使用LRU顺序分配内存，如同一个队列，总是在把来自于unsorted bin的chunk添加为第一个chunk，而取链表尾节点（也就是最早被释放并添加进来的chunk）分配给用户。  
**Large**  
    一个largebin中会包含不同size的chunks，它们按降序排列，并且在分配时采用best-fit算法寻找最匹配的块，chunk中多出请求的内存会被切除，作为remainder chunk添加到unsorted bin中。  
  
？？图片  
  
在上图和下图中，所有指针都指向一个chunk（`mchunkptr`）。由于bins不是chunks（它们是元素为一对`forward/back`指针的数组），一个小技巧被用于为一个mchunkptr提供一个类似于chunk的对象，这个对象正好重叠了数个bins，使得`foward`或`back`指针能被正确用于指向相应的bin。  
  
由于需要找到最佳适配的large chunk，large chunks有额外的双向链表连接（回忆上文中chunk结构体定义中的`fw_nextsize/bk_nextsize`）——这一对指针把一个large bin内每一个size的第一个chunk按size从大到小连接起来。这个设计是为了便于malloc能快速地遍历一个bin来寻找第一个足够大的chunk。如果有多个相同size的large bins，那么通常最佳适配size的第二个chunk会被取出，这样可以避免对size链表的修改（即减少一对指针的修改）。出于同样的考虑，添加large chunk到bin中时会把它添加到同样size的chunk之后（如果有的话）。  
  
？？图片  
  


*译注：在典型的64位系统中，size_t和指针的大小为8个字节，先由`#define MAX_FAST_SIZE     (80 * SIZE_SZ / 4)`计算得出最大的fastbin内的chunk size为160 bytes，再要减去chunk的overhead（2个size和2个指针，见前文对chunk结构体的描述）得到最大有效负荷为128 bytes。因此，在这种情况下，fastbins响应内存请求的默认范围是[0, 128 bytes]。*  













