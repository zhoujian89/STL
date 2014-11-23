Standard Template Library
===
##<a name="index"/>目录
* [Memory Allocation](#distribution)
* [Function Objects](#title)
* [Containers](#text)
* [Iterators](#text)
* [Algorithm](#text)
* [Adaptors](#text)

##<a name="distribution"/>Memory Allocation
  在STL中，Memory Allocator 处于最底层的位置，为一切的 Container 提供存储服务，是一切其他组件的基石。对于一般使用 STL 的用户而言，Allocator 是不可见的，如果需要对 STL 进行扩展，如编写自定义的容器，就需要调用 Allocator 的内存分配函数进行空间分配。
  
在C++中，一个对象的内存配置和释放一般都包含两个步骤，对于内存的配置，首先是调用operator new来配置内存，然后调用对象的类的构造函数进行初始化；而对于内存释放，首先是调用析构函数，然后调用 operator delete进行释放。Allocator 的作用相当于operator new 和operator delete的功能，只是它考虑得更加细致周全。

在Allocator的标准接口中，真正内容比较充实的是 __malloc_alloc_template 和 __default_alloc_template 这两个配置器，这两个配置器就是 SGI STL 配置器的精华所在。其中 __malloc_alloc_template 是SGI STL 的第一层配置器，只是对系统的 malloc , realloc 函数的一个简单封装，并考虑到了分配失败后的异常处理。而 __default_alloc_template 是SGISTL的第二层配置器，在第一层配置器基础上还考虑内存碎片的问题，通过内置一轻量级的内存池。

SGI STL 中考虑到了内存分配失败的异常处理，内置轻量级内存池（主要用于处理小块内存的分配，应对内存碎片问题）实现， 多线程中的内存分配处理（主要是针对内存池的互斥访问）等。本文就主要分析 SGI STL 中在这三个方面是如何处理的。

__内存分配失败的异常处理__

内存分配失败一般是由于out-of-memory(oom)，SGI STL 本身并不会去处理oom问题，而只是提供一个 private 的函数指针成员和一个 public 的设置该函数指针的方法，让用户来自定义异常处理逻辑
如果用户没有调用该方法来设置异常处理函数，那么就不做任何异常处理，仅仅是想标准错误流输出一句out of memory并退出程序（对于使用new和C++特性的情况而言，则是抛出一个std::bad_alloc()异常，因为该函数指针的缺省值为0，此时对应的异常处理是 THROW_BAD_ALLO。SGI STL 内存配置失败的异常处理机制就是这样，提供一个默认的处理方法，也留有一个用户自定义处理异常的接口。

__内置轻量级内存池的实现__

第一级配置器 __malloc_alloc_template 仅仅只是对 malloc 的一层封装，没有考虑可能出现的内存碎片化问题。内存碎片化问题在大量申请小块内存是可能非常严重，最终导致碎片化的空闲内存无法充分利用。SGI 于是在第二级配置器 __default_alloc_template 中 内置了一个轻量级的内存池。 对于小内存块的申请，从内置的内存池中分配。然后维护一些空闲内存块的链表（简记为空闲链表，free list），小块内存使用完后都回收到空闲链表中，这样如果新来一个小内存块申请，如果对应的空闲链表不为空，就可以从空闲链表中分配空间给用户。具体而言SGI默认最大的小块内存大小为128bytes，并设置了128/8=16 个free list，每个list 分别维护大小为 8, 16, 24, …, 128bytes 的空间内存块（均为8的整数倍），如果用户申请的空间大小不足8的倍数，则向上取整。

其中 _S_refill 和 _S_chunk_alloc 这两个函数是该内存池机制的核心。 __default_alloc_template 对外提供的 public 的接口有 allocate, deallocate 和 reallocate 这三个，其中涉及内存分配的 allocate 和 reallocate 的逻辑思路是，首先看申请的size（已round up）对应的free list是否为空，如果为空，则调用 _S_refill 来分配，否则直接从对应的free list中分配。而 deallocate 的逻辑是直接将空间插入到相应free list的最前面。

函数 _S_refill 的逻辑是，先调用 _S_chunk_alloc 从内存池中分配20块小内存（而不是用户申请的1块），将这20块中的第一块返回给用户，而将剩下的19块依次链接，构建一个free list。这样下次再申请同样大小的内存就不用再从内存池中取了。有了 _S_refill ，用户申请空间时，就不是直接从内存池中取了，而是从 free list 中取。因此 allocate 和 reallocate 在相应的free list为空时都只需直接调用 _S_refill 就行了。

这里默认是依次申请20块，但如果内存池空间不足以分配20块时，会尽量分配足够多的块，这些处理都在 _S_chunk_alloc 函数中。该函数的处理逻辑如下：

1) 能够分配20块
从内存池分配20块出来，改变 _S_start_free 的值，返回分配出来的内存的起始地址

2) 不足以分配20块，但至少能分配一块
分配经量多的块数，改变 _S_start_free 的值，返回分配出来的内存的起始地址

3) 一块也分配不了
首先计算新内存池大小 size_t __bytes_to_get = 2 * __total_bytes + _S_round_up(_S_heap_size >> 4)

将现在内存池中剩余空间插入到适当的free list中

调用 malloc 来获取一大片空间作为新的内存池：

– 如果分配成功，则调整 _S_end_free 和 _S_heap_size 的值，并重新调用自身，从新的内存池中给用户分配空间； – 否则，分配失败，考虑从比当前申请的空间大的free list中分配空间，如果无法找不到这样的非空free list，则调用第一级配置器的allocate，看oom机制能否解决问题


__内存池在多线程下的互斥访问__

