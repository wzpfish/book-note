# CLR 中的线程池

由于创建、销毁线程会带来许多开销，拥有大量线程会消耗大量的资源，因此 CLR 有一套机制来维护自己的线程池。

如下图所示，CLR 线程池中的线程都有一个自己的任务队列，线程池本身还有一个全局任务队列。

![Image of Thread Pool](https://raw.githubusercontent.com/wzpfish/book-note/master/clr-via-c%23/img/[ch27]thread_pool.png)

当需要执行一个简单的 Compute-Bound 操作时，可以用如下函数：
```
static Boolean QueueUserWorkItem(WaitCallback callBack);
static Boolean QueueUserWorkItem(WaitCallback callBack, Object state);
```
该函数将一个 work item 塞到线程池的全局队列里，当线程池里的某个线程取到这个 work item 时，就会调用它的 callBack 函数。**Note，如果 callBack 抛出未被捕获的异常，则 CLR 会强制终止该线程所在的进程。**

`Timer` 类也会将任务塞到全局任务队列中。

如果用 `TaskScheduler.Default` 来调度任务，那么当一个 non-worker thread 调度一个任务，该任务会被放入全局任务队列；当一个 worker thread 调度一个任务，该任务会被放入 thread local queue. 当一个 worker thread 准备执行一个任务时，它按照如下顺序找一个任务：
1. 以 LIFO 的顺序从 tread local queue 中拿一个任务（无锁）。
2. 如果 thread local queue 中没有任务了，则从其它 thread 的 local queue 的队尾 steal 一个任务（加锁）。
3. 如果所有 thread local queue 都空了，则以 FIFO 的顺序从线程池的 global queue 中拿一个 task 来执行。

关于 `TaskScheduler` 多说几句。默认情况下，所有应用都用 tread pool task scheduler，也即上面说的 `TaskScheduler.Default`，调度方式也是上面说的。还有一种典型的是 synchronization context task scheduler，即 `TaskScheduler.FromCurrentSynchronizationContext`。所有用这种 scheduler 的 task 会被 `post` 到当前的 synchronization context 中。

# Execution Context

每个线程都保存着一个 execution context data structure，这个 structure 里面包含了 security settings，host settings 以及 logical call context 等等内容。

默认情况下，当一个线程请另一个线程（helper thread）帮忙时（无论是通过`QueueUserWorkItem`、`Task.Run`还是`new Thread`），CLR 都会默认 copy 当前线程的 execution context 到另一个线程中，这个过程称为 **flow**。

由于 flow 有大量的 copy 操作会影响 performance，因此 C# 提供了一个 `ExecutionContext` 类，如下：
```
public sealed class ExecutionContext : IDisposable, ISerializable {
    [SecurityCritical] public static AsyncFlowControl SuppressFlow();
    public static void RestoreFlow();
    public static Boolean IsFlowSuppressed();
    // Less commonly used methods are not shown
}
```
通过调用 `SuppressFlow` 方法，可以禁止 execution context flow，禁止后之后，helper thread 中的 execution context 是该线程上一次使用的东西。

在 server application 中，这能大大提高性能。但是需要注意，任何与 security settings、host settings 以及 logical callcontext 相关的东西都不能去用！比如下面例子：
```
CallContext.LogicalSetData("Name", "wzpfish");

// this will output "wzpfish".
ThreadPool.QueueUserWorkItem(
    state => Console.WriteLine($"QueueUserWorkItem, Name={CallContext.LogicalGetData("Name")}")
);

// this will output empty.
ExecutionContext.SuppressFlow();
ThreadPool.QueueUserWorkItem(
    state => Console.WriteLine($"QueueUserWorkItem, Name={CallContext.LogicalGetData("Name")}")
);

ExecutionContext.RestoreFlow();
```