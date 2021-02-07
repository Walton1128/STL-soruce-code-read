
C++标准库将对其中各实体（比如std::vector<>，std::deque<>...）的内存管理封装进了被称为allocator的类模板中。一个allocator主要包含如下几个函数：

- allocate()：被用来分配未初始化的内存。
- construct()：在由allocate()分配的未初始化内存中构造对象。
- destroy()：析构由construct()构造的对象。
- deallocate()：释放由allocate()分配的内存。

在后续内容中将会对这些函数的使用有所涉及。

标准库提供了多种allocator类模板。比如：

1. new_allocator，该allocator只是简单的封装了::operator new和::operator delete，也是目前默认的allocator。
2. malloc_allocator，该allocator也只是简单的封装了malloc和free。
3. debug_allocator，该allocator封装了任意一种其它类型的allocator(称之为A)。它会不断地向A申请相比于之前稍微更大一些的内存，并用额外的内存存储size信息。在deallocate中，会用一个assert()来检查被存储的size信息和将要被释放的内存的size是否一致。
4. __pool_alloc，一个高性能的内存池allocator。

以及__mt_alloc，bitmap_allocator，throw_allocator等适用于不同场景的allocator。本文将重点介绍\_\_pool_alloc。



## \_\_poll_alloc原理简介

\_\_pool_alloc使用的是一个单池，该单池会被\_\_pool_alloc的不同实例所共享。\_\_pool_aloc适用于为size小于128 byte的类型进行的内存申请，如果某个类型的size不是8的整数倍，那么为其申请的内存会被向上取整到8的倍数。为大于128 bytes的类型执行的内存申请将被直接转发给::operator new。所有的已分配的内存最终都会被回收，并用一个数组进行管理。

\_\_pool_alloc的父类\_\_pool_alloc_base是一个“**非**”模板类，上述数组就是其所包含的一个static的、长度为16的数组。被回收的内存会被以单链表的形式挂载到这个数组中，根据被回收内存的大小，会为其在数组中选择合适的位置，并在该位置存储一个指向链表起点的指针。（整体而言是hash桶的组织方式）

<img src="https://github.com/Walton1128/STL-soruce-code-read/blob/main/Figures/image-20210206174947789.png" style="zoom:85%;" />

为更直观的阐述其工作机制，引入如下代码：

```c++
1  #include <iostream>
2  #include <memory>
3  #include<ext/pool_allocator.h>
4  using namespace std;

5  struct S_8
6  {
7    double a[1];
8  };

9  struct S_16
10 {
11    double a[2];
12  };

13  struct S_40
14  {
15     double a[5];
16  };

17  struct S_120
18  {
19     double a[15];
20  };


21  int main(){
22    __gnu_cxx::__pool_alloc<S_8> S_8_allo;
23    __gnu_cxx::__pool_alloc<S_16> S_16_allo;
24    __gnu_cxx::__pool_alloc<S_120> S_120_allo;
25    __gnu_cxx::__pool_alloc<S_40> S_40_allo;

26   auto p_S_16 = S_16_allo.allocate(1); 
27   auto p_S_16_1 = S_16_allo.allocate(1); 

28   std::cout << (char*) p_S_16_1 - (char *) p_S_16 << std::endl;  // should output 16 

29   auto p_S_8 = S_8_allo.allocate(1);
30   std::cout << (char *) p_S_8 - (char*) p_S_16 << std::endl;     //should output 320

31   auto p_S_120 = S_120_allo.allocate(1);
32   std::cout << (char *) p_S_120 - (char*) p_S_16 << std::endl;  // should output 480 

33   auto p_S_120_1 = S_120_allo.allocate(1);
34   std::cout << (char *) p_S_120_1 - (char*) p_S_16 << std::endl;   // a unpredictale value

35   auto p_S_40 = S_40_allo.allocate(1);
36   std::cout << (char *) p_S_40 - (char*) p_S_16 << std::endl;   // shoud output 600

37  S_16_allo.deallocate(p_S_16, 1);
38  S_16_allo.deallocate(p_S_16_1, 1);
39  S_8_allo.deallocate(p_S_8, 1);
40  S_120_allo.deallocate(p_S_120, 1);
41  S_120_allo.deallocate(p_S_120_1, 1);
42  S_40_allo.deallocate(p_S_40, 1);

43  return 0;
}
```

代码中引入struct S_8，struct S_16，struct S_40和struct S_120，只是单纯的为了引入大小为8，16，40和120 byte的类型。后续图例中的\_S_heap_size记录的是当前为内存池所分配的总的内存的大小，_S_start_free记录的是由内存池所分配的内存块中未被利用、也未被归入到某个链表中的内存的起点，_S_end_free所记录的则是这块内存的终点，它们都是父类\_\_pool_alloc_base的static成员。

## 内存分配

编译并运行上述代码，当代码运行到第26行后，已经分别为这4种类型创建了各自的allocator，但是请记住它们将共享同一个内存池，此时内存池所管理的内存为零（\_S_heap_size = 0， _S_start_free = 0， S_end_free = 0）。

在第26行代码运行期间，为了响应内存分配的请求，\_\_pool_alloc会先分配一块大小为40 * sizeof(S_16) = 640 bytes的内存，并将前16个byte返回给p_S_16，然后将前20 * sizeof(S_16) = 320 byte中扣除前16 byte的内存（304 bytes）挂载到长度为16的static数组中的对应位置，其间会将这304 bytes的内存分成19个链表中前后相连的node。图示如下：

<img src="https://github.com/Walton1128/STL-soruce-code-read/blob/main/Figures/image-20210206182331049.png" style="zoom:85%;" />

当程序运行第27行代码时，由于它发现在内存池中对应于类型大小为16 byte的链表不为空，因此它只需要从这一链表中直接取用16 byte内存（链表中第一个node），并将链表起点后移一位即可，

<img src="https://github.com/Walton1128/STL-soruce-code-read/blob/main/Figures/image-20210206183459493.png" style="zoom:85%;" />

由于p_S_16和p_S_16所指的内存是连续的，因此程序第28行将会输出16。

在程序运行第29行时，由于它发现对应于8 byte的链表为空，因此会试着从\_S_start_free和\_S_end_free中找出20 * 8 byte的内存给相应链表，并将链表中第一个node的内存分配给p_S_8，然后后移链表起点，此时状态如下：

<img src="https://github.com/Walton1128/STL-soruce-code-read/blob/main/Figures/image-20210206184631922.png" style="zoom:85%;" />

如图所示，第30行代码的输出结果将是320。

在程序运行第31行代码时，虽然介于\_S_start_free和\_S_end_free之间的内存小于120 * 20 byte，但是依然大于120 byte并小于2 * 120 byte，因此\_\_pool_alloc会将其中的第一个120 byte直接赋值给p_S_120，此时的状态如下：

<img src="https://github.com/Walton1128/STL-soruce-code-read/blob/main/Figures/image-20210206191500779.png" style="zoom:85%;" />

显然第32行代码将输出480。

第33行代码试图再次申请一块大小为120 byte的内存，但是此时介于\_S_start_free和\_S_end_free之间的内存并不足以满足这一要求，因此需要开辟新的内存块。不过，在分配新的内存块之前，我们应该尽可能的利用剩余的、介于\_S_start_free和\_S_end_free之间的这一块内存。方法就是将其挂载到适合的链表中，然后再去分配新的内存块（新内存块的尺寸将是40* 120加上\_S_heap_size >> 4之后向上取整到8的倍数的值，也就是：4840 bytes）。随后采入相同的方式，将新内存块中前20*120 byte的内存挂载到120对应的链表下，然后将第一个node赋值给p_S_120_1，并后移链表的起点。此时的状态如下：

<img src="https://github.com/Walton1128/STL-soruce-code-read/blob/main/Figures/image-20210206192613729.png" style="zoom:85%;" />

由于p_S_120_1的起点在新的内存块中，因此第34行代码的结果将是一个不可预知的值。

当程序执行第35行代码时，由于40byte对应的链表的不是空的，因此会直接从链表中得到所需的内存，并将链表起点后移一位（结果是使该链表变为空），此时状态如下：

<img src="https://github.com/Walton1128/STL-soruce-code-read/blob/main/Figures/image-20210206193140157.png" style="zoom:85%;" />

由于此时p_S_40和p_S_16所指向的内存处在同一个内存块中，因此第36行代码将会输出600。

以上过程大致涵盖了\_\_pool_alloc中各种内存分配的情况，后续新的内存申请将遵照相同的形式进行。下面来看内存回收再利用的情况。

### 内存回收再利用

作为使用了内存池的allocator，\_\_pool_alloc并不会将其分配的内存释放掉，而是会将回收内内存放到与其对应的链表中。由于过程相对简单，此处只将其最终结果展示如下（120 byte对应的链表有两条箭头）：

<img src="https://github.com/Walton1128/STL-soruce-code-read/blob/main/Figures/image-20210206230032745.png" style="zoom:85%;" />



## 讨论

\_\_pool_alloc提供了一种简单、普适且高效的内存池实现机制，能够很容易地被用于std::vector一类的容器。但是该实现并没有提供主动释放内存池中内存的接口，因此这部分内存一旦被分配，就会一致持续存在，直到程序结束。这会导致对于某些只有中期某个时间点会占用大量内存的程序，其内存占用会一直保持较高的占用率，这或许是我们所希望能够尽量避免的，但是增加能存池大小的动态调整的功能，无疑会大大增加该实现的复杂度，又会降低该实现的普适性。













