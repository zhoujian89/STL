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
  在STL中，Memory Allocator 处于最底层的位置，为一切的 Container 提供存储服务，是一切其他组件的基石。对于一般使用 STL 的用户而言，Allocator 是不可见的，如果需要对 STL 进行扩展，如编写自定义的容器，就需要调用 Allocator 的内存分配函数进行空间配置。本文涉及到的 SGI STL 源代码文件有 alloc.h, stl_config.h, stl_alloc.h, stl_threads.h 这4个。

在C++中，一个对象的内存配置和释放一般都包含两个步骤，对于内存的配置，首先是调用operator new来配置内存，然后调用对象的类的构造函数进行初始化；而对于内存释放，首先是调用析构函数，然后调用 operator delete进行释放。 
Allocator 的作用相当于operator new 和operator delete的功能，只是它考虑得更加细致周全。SGI STL 中考虑到了内存分配失败的异常处理，内置轻量级内存池（主要用于处理小块内存的分配，应对内存碎片问题）实现， 多线程中的内存分配处理（主要是针对内存池的互斥访问）等，本文就主要分析 SGI STL 中在这三个方面是如何处理的。在介绍着三个方面之前，我们先来看看 Allocator的标准接口。
其中simple_alloc , debug_alloc , allocator 和 __allocator 的实现都比较简单，都是对其他适配器的一个简单封装（因为实际上还是调用其他配置器的方法，如 _Alloc::allocate ）。而真正内容比较充实的是 __malloc_alloc_template 和 __default_alloc_template 这两个配置器，这两个配置器就是 SGI STL 配置器的精华所在。其中 __malloc_alloc_template 是SGI STL 的第一层配置器，只是对系统的 malloc , realloc 函数的一个简单封装，并考虑到了分配失败后的异常处理。而 __default_alloc_template 是SGI STL 的第二层配置器，在第一层配置器的基础上还考虑了内存碎片的问题，通过内置一个轻量级的内存池。下文将先介绍第一级配置器的异常处理机制，然后介绍第二级配置器的内存池实现，及在多线程环境下内存池互斥访问的机制。

__内存分配失败的异常处理__

__内置轻量级内存池的实现__

__内存池在多线程下的互斥访问__

