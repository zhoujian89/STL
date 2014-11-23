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
如果用户没有调用该方法来设置异常处理函数，那么就不做任何异常处理，仅仅是想标准错误流输出一句out of memory并退出程序（对于使用new和C++特性的情况而言，则是抛出一个std::bad_alloc()异常）， 因为该函数指针的缺省值为0，此时对应的异常处理是 THROW_BAD_ALLO。SGI STL 内存配置失败的异常处理机制就是这样，提供一个默认的处理方法，也留有一个用户自定义处理异常的接口。

__内置轻量级内存池的实现__

__内存池在多线程下的互斥访问__

