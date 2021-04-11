---
title: "Atomic"
date: 2021-04-10T22:24:10+08:00
draft: false
tags: [”C++“, "C++11"]
summary: "What and How about a atomic variable, welcome to C++"
---

C++11引入了一堆新玩具, 其中STL封装了一个叫atomic的东西`std::atomic`.

`std::atomic`定义了唯一一种不可能产生数据争用的数据类型, 任何线程写入原子变量的行为都是明确定义的.

本篇我们来聊一聊这个东西的用法和实现细节.

### WHAT

atomic类型需要引入atomic头文件, 头文件具体位置一般在`/usr/include/c++/x.x.x/atomic`.
```c++
#include <atomic>
```
atomic头文件打开看一下, 它是这么定义的:
```c++
template<typename _Tp>
    struct atomic;
```
atomic是一种模板类型, 也就意味Atomic支持以下的写法:
```c++
std::atomic<T> variable;
```
目前(至C++20)atomic支持bool, 所有整数类型(ptr, char, int, uint16_t, uint32_t, uint64_t...), 浮点类型(float, double, C++20后加入), 部分智能指针(shared_ptr, weak_ptr, C++20), 或者自定义的trivially-copyable类型也可以, 具体的看[cpprefrence](https://en.cppreference.com/w/cpp/atomic/atomic).

你可以用以下的方法判断你的atomic是不是"ill-formed":
```c++
std::is_trivially_copyable<T>::value
std::is_copy_constructible<T>::value
std::is_move_constructible<T>::value
std::is_copy_assignable<T>::value
std::is_move_assignable<T>::value
```
任何一个false就表示你这个atomic有问题.

接下来就像一个普通变量一样用就好了.

一个atomic变量有一些原子操作, 来康康:

+ `fetch_add`
+ `fetch_sub`
+ `fetch_and`
+ `fetch_or`
+ `fetch_xor`
+ `operator++`
+ `operator--`
+ `operator+=`
+ `operator-=`
+ `operator&=`
+ `operator|=`
+ `operator^=`
+ `operator=`
+ `store`
+ `load`

这些操作都保证原子性, 其他的操作就不行, 比如说`a = a + 1`, 这个操作语义上和`a++`一样, 但是编译器不会认为这是个原子操作, 它将导致a.load()(原子), 然后在该值与最终结果a.store()(也是原子). std::memory_order_seq_cst将在这里使用.

std::mem_order是内存顺序, 有6种:

+ `memory_order_relaxed`
+ `memory_order_consume`
+ `memory_order_acquire`
+ `memory_order_release`
+ `memory_order_acq_rel`
+ `memory_order_seq_cst`

这些mem order我也没有完全弄懂(其实也没必要提, 一般用不上), 不过可以说一下第一个内存顺序, 因为确实可能会发生问题, 比如以下来自refrence的代码:
``` c++
// Thread 1:
r1 = y.load(std::memory_order_relaxed); // A
x.store(r1, std::memory_order_relaxed); // B
// Thread 2:
r2 = x.load(std::memory_order_relaxed); // C 
y.store(42, std::memory_order_relaxed); // D
```
可以看到relaxed下可能变成`x=y=42`. 确实, 这里每个操作都是原子的, 但是顺序呢就有点问题, 你永远不知道是`x=y=42`还是`x=y,y=42`, 反正就是不要这么写.

### HOW

来康康这个东西是怎么实现的. 我们首先引入一个概念--Cache, Cache都知道, 但是Cache这个东西是怎么和atomic关联起来的.

首先我们要明白, 真正意义上的"无锁"是很难做到的, 很多所谓的无锁只是把锁藏到程序员看不到的地方, 比如说CPU内部.

先说个和Cache没关系的实现方式, **总线锁**(Bus Lock).

总线锁在原子操作执行的时候会直接锁住总线, 阻止其他线程执行内存操作, 这不就一致了. 但是问题来了, 别的线程又不要修改这个变量, 那锁了内存岂不是很蠢. 

接下来引出我们的带明星**缓存锁**(Cacheline Lock).

缓存锁仅锁住在高速缓存中的变量, 其他的地址访问依然可以照常. 当CPU-1尝试对一个地址原子递增, 它会将cacheline标记为locked, 并设置为Exclusive, 直到执行完成操作, 然后将cacheline设置为unlock. 当有其他的CPU, 比如CPU-2也想执行一波原子递增, 它会向CPU-1发送Invalidate消息, 然后等CPU-1发送回ACK Invalidate再执行原子操作.

当然具体实现没有我这里说的那么简单, 但是也大差不差, 有兴趣的话可以了解下一个叫MESI协议(又称Illinois protocol)的协议, 这个协议用来保证多核缓存一致性.

对于某些硬件架构也有不同的实现, 比如aarch64就不是上述任何一种实现. 也有软实现的, 比如CAS(compare and swap).

**相关资料**
+ Atomic[https://en.cppreference.com/w/cpp/atomic/atomic](https://en.cppreference.com/w/cpp/atomic/atomic)
+ memory order[https://en.cppreference.com/w/cpp/atomic/memory_order](https://en.cppreference.com/w/cpp/atomic/memory_order)
+ How [https://zhuanlan.zhihu.com/p/115355303](https://zhuanlan.zhihu.com/p/115355303)