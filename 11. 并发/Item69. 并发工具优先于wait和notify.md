# Item69. 并发工具优先于wait和notify





Java 1.5 版本开始，平台开始提供更加高级的「并发工具」，能够完成以前必须在 `await` 和 `notify` 上手写代码来完成的各项工作。正确地使用 `wait  ` 和 `notify` 比较困难，就应该用更高级的并发工具来代替。 



`java.util.concurrent` （JUC）中更高级的工具分 3 类：

1. Executor 框架
2. 并发集合 Concurrent Collection
3. 同步器 `Synchronizer`



上一章讲过 Executor 框架了，下面简单阐述「并发集合」和「同步器」。

## 并发集合

并发集合为标准集合接口（如 `List`、`Queue`、`Map`）提供了**高性能的并发实现**。为了提供高并发性，这些实现「在内部自己管理同步」（见 Item 67）。因此并发集合中不可能排除并发活动；将它锁定并没有什么用，只会将程序的速度变慢。



这意味着客户**无法「原子地」对并发集合进行方法调用**。因为这些集合接口已经通过「**依赖状态的修改操作**」进行了扩展，**将几个基本操作合并到了单个原子操作中**。例如 `ConcurrentMap` 扩展了 `Map`，并添加了几个方法，包括 `putIfAbsent(key, value)`，即键不存在会插入一个映射，并返回与键关联的前一个值，如果键存在就不会 `put` 这个键值对，只返回原来的值，如果没有这样的值返回 `null`。



下面的方法模拟了 `String.intern` 的行为:



```java
// Concurrent canonicalizing map atop ConcurrentMap - not optimal
private static final ConcurrentMap<String, String> map = new ConcurrentHashMap<>();

public static String intern(String s) {
    String previousValue = map.putIfAbsent(s, s);
    return previousValue == null ? s : previousValue;
}
```



还可以做的更好， `ConcurrentHashMap` 对获取操作进行了优化，只有当 get 表明有必要的时候，才值得先调用 get，再调用 `pugIfAbsent`： 

```java
// Concurrent canonicalizing map atop ConcurrentMap - faster!
public static String intern(String s) {
    String result = map.get(s);
    if (result == null) {
        result = map.putIfAbsent(s, s);
        if (result == null)
            result = s;
    }
    return result;
}
```





`ConcurrentHashMap` 除了提供卓越的并发性能，速度也非常快。除非迫不得已应该先使用 `ConcurrentHashMap`，而不是 `Collections.synchronizedMap` 或者 `HashTable`。只要使用并发 Map 替换老式的同步 Map，就可以极大地提升并发应用程序的性功能。更一般地，**应该优先使用并发集合，而不是使用外部同步的集合。**



有些集合接口已经通过「阻塞操作」进行了扩展，或一直等待（或阻塞）到可以执行为止，例如 `BlockingQueue` 扩展了 `Queue` 即可，添加了包括 `take` 在内的几个方法，从队列中删除并返回头元素，如果队列为空则等待。这样就允许阻塞队列用于「工作队列」，也称为「生产者-消费者队列」，大多数 `ExecutorService` 实现（包括 `ThreadPoolExecutor`）都适用 `BlockingQueue`。



## 同步器 Synchronizer



同步器 Synchronizer 是一「**使线程能够等待另一个线程**」的对象，允许它们「**协调动作**」。最常用的同步器是 `CountDownLatch` 和 `Semaphore`。较不常用的是 `CyclicBarrier` 和 `Exchanger`。



`CountdownLatch`：倒计数锁存器是「一次性」的“障碍”，允许一个或多个线程「**等待**」一个或多个其他线程，来做某些事情。`CountDownLatch` 唯一的构造器有一个 `int` 类型的参数，意思是允许所有在等待的线程被处理之前，必须在「锁存器」上调用 `countDown`方法的次数。



要在这个简单的基本类型之上构建一些有用的东西，做起来是相当的容易。例如，假设想要构建一个简单的框架，**用来给一个动作的并发执行定时**。这个框架中包含单个方法，这个方法带有**一个执行该动作的 executor，一个并发级别（表示要并发执行该动作的次数），以及表示该动作的 `runnable`**。

所有的工作线程自身都准备好，要在 `timer` 线程启动时钟之前运行该动作（为了实现准确的定时，这是必需的）。**当最后一个工作线程准备好运行该动作时， `timer` 线程就“发起头炮”，同时允许工作线程执行该动作。一旦最后一个工作线程执行完该动作， `timer` 线程就立即停止计时。**直接在`wait`和 `notify` 之上实现这个逻辑至少来说会很混乱，而在 `CountDownlatch` 之上实现则相当简单：



```java
// Simple framework for timing concurrent execution
public class ConcurrentTimer {
    private ConcurrentTimer() { } // Noninstantiable

    public static long time(Executor executor, int concurrency,
                            Runnable action) throws InterruptedException {
        CountDownLatch ready = new CountDownLatch(concurrency);
        CountDownLatch start = new CountDownLatch(1);
        CountDownLatch done  = new CountDownLatch(concurrency);

        for (int i = 0; i < concurrency; i++) {
            executor.execute(() -> {
                ready.countDown(); // 告诉 timer 已经准备好了
                try {
                    start.await(); // 在执行之前，等待「同伴」也准备好(ready倒数至0)
                    action.run();  // 执行当前 action
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    done.countDown();  // 最终告诉 timer 已经执行完了
                }
            });
        }

        ready.await();     // 等待所有工作线程准备好
        long startNanos = System.nanoTime();
        
        start.countDown(); // 在所有线程 ready 之后，告诉工作线程，你可以执行 action 了

        done.await();      // 等待所有工作线程执行完
        
        return System.nanoTime() - startNanos;
    }
}

```



还有一个细节要注意，就是 executor 必须允许创建于指定并发级别一样多的线程，否则会导致「**线程饥饿死锁**」。如果工作线程捕捉到 `InterruptedException`，就会利用习惯用法：`Thread.currentThread().interrupt()`重新断言中断，并从它的 `run` 方法中返回。这样就允许 `executor` 在必要时处理中断，事实上也理当如此。最后，对于「**间歇式的定时**」，使用应该优先使用 `System.nanTime`，而不是 `System.currentTimeMills`，因为它更准确、精确，也不受系统的实时时钟的调整所影响。



其他并发工具的详细说明请参阅《Java并发编程实战》。





## 维护老代码？使用 wait、notify 方法的标准模式



虽然你始终应该优先使用并发工具，而不是使用`wait`和 `notify`，但可能必须维护使用了`wait`和`notify`的遗留代码。**`wait`方法被用来使线程等待某个条件。它必须在同步区域内部被调用**，这个同步区域将对象锁定在了调用 `wait` 方法的对象上。下面是使用`wait`方法的标准模式：





```java
synchronized(obj){
    while(<不满足条件>){
        obj.wait(); // 暂时释放锁，需要唤醒
    }
    ... // 根据条件执行合适的动作
}
```



**始终应该使用 `wait` 循环模式来调用 `wait` 方法，永远不要在循环之外调用 `wait` 方法**，循环会在等待之前和之后测试条件。



**在等待之前测试条件，当条件已经成立时就跳过等待**，这对于「**确保活性 liveness**」是必要的，如果条件已经成立，并且在线程等待之前，`notify` （或`notifyAll`）方法已经被调用，则无法保证线程将会从等待中苏醒过来。



**在等待之后测试条件，如果条件不成立的话继续等待，这对于确保安全性（ safety）是必要的。**当条件不成立的时候，如果线程继续执行，则可能会破坏被锁保护的约束关系。**当条件不成立时，有下面一些理由可使一个线程苏醒过来：**

- 另一个线程可能已经得到了锁，并且从一个线程调用 `notify` 那一刻起，到等待线程苏醒过来的这段时间中，得到锁的线程已经改变了受保护的状态。
- 条件并不成立，但是另一个线程可能意外地或恶意地调用了 `notify`。在公有可访问的对象上等待，这些类实际上把自己暴露在了这种危险的境地中。公有可访问对象的同步方法中包含的 `wait`都会出现这样的问题。
- 通知线程在唤醒等待线程时可能会过度“大方”。例如只有某些等待线程的条件已经被满足，但是通知线程可能仍然调用 `notifyAll`。
- 在没有通知的情况下，等待线程也可能（但很少）会苏醒过来，这称为「伪唤醒」。





一个相关的话题是，为了唤醒正在等待的线程，你应该使用 `notify` 还是 `notifyAll`（回忆一下， `notify`唤醒的是单个正在等待的线程，假设有这样的线程存在，而 `notifyAll` 唤醒的则是所有正在等待的线程。**一种常见的说法是，你总是应该使用 `notifyAll` 这是合理而保守的建议。它总会产生正确的结果，因为它可以保证你将会唤醒所有需要被唤醒的线程。你可能也会唤醒其他一些线程，但是这不会影响程序的正确性。这些线程醒来之后，会检査它们正在等待的条件，如果发现条件并不满足，就会继续等待。**



**从优化的角度看，如果处于等待状态的所有线程都在等待同一个条件，而每次只有一个线程可以从这个条件中被唤醒，就应该选择 `notify`。**



即使这些条件都是真的，也许还是有理由使用 `notifyAll` 而不是 `notify`，就好像把 `wait` 调用放在一个循环中，以避免在公有可访问对象上意外或恶意的通知已于，与此类似，使用 `notifyAll` 代替 `notify` 可以避免来自不相关线程的意外或恶意的等待。否则，这样的等待会“吞并”掉一个关键的通知，使真正的接受线程无限等待下去。



简而言之，直接使用 `wait`、`notify` 就像「并发汇编语言」进行编程一样，而 `java.util.concurrent` 提供了更高级的语言。没有理由在新代码中使用 `wait` 和 `notify`，即使有，也是极少的情况。如果你在维护使用 `wait` 和 `notify` 的代码，务必确保使用标准模式从 `while` 循环内部调用 `wait`。一般情况下，应该优先使用 `notifyAll`，而不是 `notify`，如果使用 `notify`，一定要小心，确保程序的「活性 liveness」。