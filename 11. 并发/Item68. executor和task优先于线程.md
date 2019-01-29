# Item68. executor和task优先于线程



Java 1.5 版提供了 `java.util.concurrent`（JUC），这个包中包含了一个 `Executor` 框架。这是一个很灵活的基于接口的「**任务执行工具**」。



```java
// 单线程执行器
ExecutorService executor = Executors.newSingleThreadExecutor();

// 执行提交一个 runnable 
executor.execute(runnable);

// 终止
executor.shutdown();
```



Executor Service 还可以做更多的事情，比如等待完成一项特殊任务，等待一个任务集合中的任何任务或所有任务完成（利用 `invokeAny` 或 `invokeAll` 方法），可以在任务完成时逐个获取这些任务的结果（利用 `ExecutorCompletionService`）。



如果想让多个线程来处理这个工作队列的请求，只要调用一个不同的静态工厂，其创建了一种不同的 Executor Service，称为「**线程池 thread pool**」，可以用固定或可变数目的线程创建一个线程池。`java.util.concurrent.Executors`类包含了静态工厂，能提供所需的大多数 executor。然后如果想来点特别的，可以直接用 `ThreadPoolExecutor` 类，其可允许控制线程池操作的几乎每个方面。



为特殊的应用程序选择 Executor Service 是很有技巧的。

- 如果编写的是**小程序**，或者是轻载的服务器，使用 `Executors.newCachedThreadPool` 通常是个不错的选择，因为**它不需要配置，并且一般情况下能够正确地完成工作**。

- 但是对于**大负载的服务器**来说，缓存的线程池就不是很好的选择了！在缓存的线程池中，**被提交的任务没有排成队列，而是直接交给线程执行。如果没有线程可用，就创建一个新的线程。**如果服务器负载得太重，以致它所有的 CPU 都完全被占用了，当有更多的任务时，就会创建更多的线程，这样只会使情况变得更糟。因此，在大负载的产品服务器中，最好使用 `Executors.newFixedThreadpool`，它为你提供了一个包含**固定线程数目的线程池**，或者为了最大限度地控制它，就直接使用 `ThreadPoolExecutor` 类。



你不仅应该尽量不要编写自己的工作队列，而且还应该**尽量不直接使用线程**。现在关**键的抽象不再是 Thread 了**，它以前可是既充当工作单元，又是执行机制。**现在「工作单元」和「执行机制」是分开的**。

**现在关键的抽象是工作单元，称作任务（task）**。任务有两种： `Runnable` 及其近亲 `Callable`（它与 `Runnable` 类似，但它会返回值）。**执行任务的通用机制是 Executor Service**。如果你从任务的角度来看问题，并让一个 Executor Service 替你执行任务，在「选择适当的执行策略」方面就获得了极大的灵活性。从本质上讲， **Executor Famework** 所做的工作是「**执行**」，犹如 Collections Framework 所做的工作是聚集（aggregation）一样。



**Executor Framework** 也有一个可以代替 `java.util.Timer` 的东西，即 `ScheduledThreadPoolExecutor`。虽然 `timer` 使用起来更加容易，但是「**被调度的线程池 executor**」更加**灵活。** `timer` 只用一个线程来执行任务，这在面对长期运行的任务时，会影响到定时的准确性。如果 `timer` 唯一的线程抛出未被捕获的异常， `timer` 就会停止执行。「**被调度的线程池 executor **」支持多个线程，并且优雅地从抛出未受检异常的任务中恢复。



更多内容参阅《Java 并发编程实战》一书。



