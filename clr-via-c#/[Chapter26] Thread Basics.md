这章主要介绍了线程的一些基本概念、线程带来的额外开销以及线程的调度。

# 进程和线程
* 进程：进程是资源的集合，用于隔离资源。
* 线程：线程是 CPU 的抽象，可以认为是 logical CPU。

# 线程的Overload
线程的概率带来了许多 overload，包括线程本身需要的资源、线程切换带来的负担、以及垃圾回收带来的负担等。
1. 线程本身需要的资源：
    * thread kernel object：包含了 thread context
    * thread environment block:消耗约 1page 内存
    * user-mode stack:存放 local variables and arguments
    * kernel-mode stack:保存传给 kernel 方法的参数
    * DLL thread-attach and thread-detach notifications

2. 线程切换带来的负担
    
    一次线程切换（大约30ms会执行一次）需要执行如下操作：
    1. 将CPU register中的值保存到线程的 thread kernel object中。
    2. 调度下一个线程。如果线程在另一个进程中，OS 还需要切换虚拟地址空间。
    3. 将下一个线程的 thread context 写入 CPU register 中。

    线程切换之后，data 和 code 又要重新从 RAM 中读取，而不是从 CPU cache 中，又慢了许多！

3. 垃圾回收带来的负担

    当CLR执行垃圾回收时，必须 suspend 所有线程，遍历它们的stacks，最后将他们又唤醒。因此线程越多，负担越大。

# 线程调度与线程优先级
Windows 被称为 preemptive multithread operating system。Windows 中的每个线程都有一个线程优先级（0 - 31），0为最低，31为最高。
OS 在调度线程的时候，会根据线程的优先级采用如下调度规则：
1. 相同优先级的线程采用 round-robin fashion，即轮流循环调度同一优先级的所有线程。
2. 高优先级线程随时抢占低优先级的线程。也就是说，当 OS 发现一个更高优先级的线程 schedulable 的时候，它会立即 suspend 当前线程，把 CPU 时间让给高优先级线程。

由于线程有32个优先级，且是抢占式的，开发者可能很难确定应该创建什么优先级的线程。因此 CLR 通过抽象进程优先级和 relative 线程优先级的方式，让开发者更容易指定线程优先级。其中，
* 进程优先级：Idle，Below Normal，Normal，Above Normal，High，Realtime
* relative 线程优先级：Idle，Lowest，Below Normal，Normal，Above Normal，Highest，Time-Critical

Note：线程的 Idle 和 Time-Critical 优先级是 CLR 保留的，用户无法指定。

特殊线程：
* zero-page thread: 它是一个0优先级线程，当没有任何线程可以被调度的时候，它会被调度，用于清理 RAM 的 free page。
* CLR’s finalizer thread: 它是一个 Time-Critical 优先级的线程，用于做垃圾回收。

# 前台线程和后台线程

前台线程和后台线程的区别是：当进程中的前台线程结束后，CLR 会强制结束所有该进程的后台线程。

有几点 notes：
* 前台线程和后台线程可以**随时**相互转换
* 线程池线程默认是后台线程。
* created by native code and enter managed excution environment 的线程是后台线程。