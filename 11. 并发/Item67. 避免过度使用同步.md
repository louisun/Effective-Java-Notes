# Item67. 避免过度使用同步

过度使用同步可能会导致性能降低、死锁，甚至不确定的行为。



为了避免活性失败和安全性失败，在一个被同步的方法或者代码块中，**永远不要放弃对客户端的控制**。换句话说，**在一个被同步的区域内部，不要调用设计成要被覆盖的方法，或者是由客户端以函数对象的形式提供的方法**。从包含该同步区域的类的角度来看这样的方法是外来的。这个类不知道该方法会做什么事情，也无法控制它。根据外来方法的作用，从同步区域中调用它会导致异常、死锁或者数据损坏。



下面列子实现了一个「可以观察到的集合包装」，允许客户端将元素添加到集合中是预定通知。这就是「观察者模式」。简洁起见，删除元素时没有提供通知。这个类是基于 `ForwardingSet` 实现的。



```java
public class ForwardingSet<E> implements Set<E> {
    private final Set<E> s;
    public ForwardingSet(Set<E> s) { this.s = s; }

    public void clear()               { s.clear();            }
    public boolean contains(Object o) { return s.contains(o); }
    public boolean isEmpty()          { return s.isEmpty();   }
    public int size()                 { return s.size();      }
    public Iterator<E> iterator()     { return s.iterator();  }
    public boolean add(E e)           { return s.add(e);      }
    public boolean remove(Object o)   { return s.remove(o);   }
    public boolean containsAll(Collection<?> c)
    { return s.containsAll(c); }
    public boolean addAll(Collection<? extends E> c)
    { return s.addAll(c);      }
    public boolean removeAll(Collection<?> c)
    { return s.removeAll(c);   }
    public boolean retainAll(Collection<?> c)
    { return s.retainAll(c);   }
    public Object[] toArray()          { return s.toArray();  }
    public <T> T[] toArray(T[] a)      { return s.toArray(a); }
    @Override public boolean equals(Object o)
    { return s.equals(o);  }
    @Override public int hashCode()    { return s.hashCode(); }
    @Override public String toString() { return s.toString(); }
}

```



```java
public interface SetObserver<E> {
    // Invoked when an element is added to the observable set
    void added(ObservableSet<E> set, E element);
}
```



```java
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;

// Broken - invokes alien method from synchronized block!
public class ObservableSet<E> extends ForwardingSet<E> {
    public ObservableSet(Set<E> set) {
        super(set);
    }

    private final List<SetObserver<E>> observers = new ArrayList<>();

    public void addObserver(SetObserver<E> observer) {
        synchronized (observers) {
            observers.add(observer);
        }
    }

    public boolean removeObserver(SetObserver<E> observer) {
        synchronized (observers) {
            return observers.remove(observer);
        }
    }

    private void notifyElementAdded(E element) {
        synchronized (observers) {
            for (SetObserver<E> observer : observers)
                // observer.added 是外来方法的调用
                observer.added(this, element);
        }
    }


    @Override
    public boolean add(E element) {
        boolean added = super.add(element);
        if (added)
            notifyElementAdded(element);
        return added;
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        boolean result = false;
        for (E element : c)
            result |= add(element); // Calls notifyElementAdded
        return result;
    }

}
```



下面打印 `0~99` 的数字：

```java
public class Test1 {
    public static void main(String[] args) {
        ObservableSet<Integer> set =
                new ObservableSet<>(new HashSet<>());

        set.addObserver((s, e) -> System.out.println(e));

        for (int i = 0; i < 100; i++)
            set.add(i);
    }
}
```



当尝试更复杂的例子：



```java
public class Test2 {
    public static void main(String[] args) {
        ObservableSet<Integer> set =
                new ObservableSet<>(new HashSet<>());
	
        // 这个观察者的 added 方法可以将自身从观察者集合中删除
        set.addObserver(new SetObserver<>() {
            public void added(ObservableSet<Integer> s, Integer e) {
                System.out.println(e);
                if (e == 23)
                    s.removeObserver(this);
            }
        });

        for (int i = 0; i < 100; i++)
            set.add(i);
    }
}
```



预计可能会打印 `0~23`的数字并正常终止，实际是打印 `0~23`后抛出 `ConcurrentModificationException`。由于当 `notifyElementAdded` 调用观察者的 `added` 方法时，正处于遍历 `observers` 列表的过程中，试图从列表中删除一个元素是非法的。



`notifyElementAdded`  方法中的迭代是在一个「同步代码块」中，可以防止并发的修改，但是无法防止「迭代线程本身回调到可观察的集合中」，也无法防止它修改它的 `observers` 列表。



现在尝试一个奇特的例子：一个试图取消预订的观察者，但是「不是直接调用 `removeObserver`」，而是用另一个线程的服务来完成，这个观察则使用了一个 `executor service`。



 

```java
public class Test3 {
    public static void main(String[] args) {
        ObservableSet<Integer> set =
                new ObservableSet<>(new HashSet<>());

		// Observer that uses a background thread needlessly
        set.addObserver(new SetObserver<>() {
            public void added(ObservableSet<Integer> s, Integer e) {
                System.out.println(e);
                if (e == 23) {
                    // 用另一个线程池去 removeObserver
                    ExecutorService exec =
                            Executors.newSingleThreadExecutor();
                    try {
                        exec.submit(() -> s.removeObserver(this)).get();
                    } catch (ExecutionException | InterruptedException ex) {
                        throw new AssertionError(ex);
                    } finally {
                        exec.shutdown();
                    }
                }
            }
        });

        for (int i = 0; i < 100; i++)
            set.add(i);
    }
}
```



这一次没有遇到「异常」，而是遇到「死锁」，后台线程调用 `s.removeObserver`，但是它无法获得锁，因为主线程已经有锁了，而且主线程在等待后台线程来完成对观察者的删除。



> 这个例子是刻意用作示范的，实际上「观察者」没有理由使用「后台线程」。但这个问题是真实的，从同步区域中调用「**外来方法**」，在真实的系统中已经造成了许多死锁，例如 GUI 工具箱。



在前面这两个例子中（异常和死锁），我们都还算幸运的。调用**外来方法**（added）时**同步区域（observers）所保护的资源处于一致的状态**。假设当同步区域所保护的约束条件暂时无效时，你要从同步区域中调用一个外来方法。由于Java程序设计语言中的**锁是可重入的**（reentrant），**这种调用不会死锁**。就像在第一个例子中一样，它会产生一个异常，**因为调用线程已经有这个锁了，因此当该线程试图再次获得该锁时会成功，尽管概念上不相关的另一项操作正在该锁所保护的数据上进行着**。这种失败的后果可能是**灾难性**的。从本质上来说，这个锁没有尽到它的职责。可重入的锁简化了多线程的面向对象程序的构造，但是它们可能会将**活性失败**（liveness failure）变成**安全性失败**（safety failure）。



幸运的是，通过将「**外来方法的调用移出同步代码块**」来解决这个问题通常并不太困难。对于 `notifyElementAdded`方法，这还涉及给 observers 列表拍张「**快照**」，然后没有锁也可以安全地遍历这个列表了。经过这一修改，前两个例子运行起来便再也不会出现异常或者死锁了。



```java
// Alien method moved outside of synchronized block - open calls
private void notifyElementAdded(E element) {
    List<SetObserver<E>> snapshot = null;
    // 同步拷贝一份 observers 列表供后续遍历
    synchronized(observers) {
        snapshot = new ArrayList<>(observers);
    }
    // 将外来方法的调用移出同步代码块
    for (SetObserver<E> observer : snapshot)
        observer.added(this, element);
}
```



还有一种更好的方法：**并发集合：`CopyOnWriteArrayList`**，对于哪些列表不怎么变化的集合是很好的，否则性能会受很大影响。对于观察者列表正适用：几乎不改动，经常被遍历。



`CopyOnWriteArrayList` 不需要改动下面这些方法，也不需要加 `synchronized` 关键字显示地同步，其内部会进行同步。

```java
private final List<SetObserver<E>> observers =
    new CopyOnWriteArrayList<>();

public void addObserver(SetObserver<E> observer) {
    observers.add(observer);
}

public boolean removeObserver(SetObserver<E> observer) {
    return observers.remove(observer);
}

private void notifyElementAdded(E element) {
    for (SetObserver<E> observer : observers)
        observer.added(this, element);
}
```





在「**同步区域之外**」被调用的**外来方法**被称为「**开放调用**」，除了可以避免死锁，开放调用还可以极大地增加并发性。外来方法的运行时间可能会任意长，如果在同步区域内调用外来方法，其他线程对受保护资源的访问就会遭到不必要的拒绝。





通常应该在同步区域做尽可能少的工作。获得锁，检查共享数据，根据需要转换数据，然后放掉锁。如果你必须要执行某个**很耗时的动作**，则应该设法把这个动作移到**同步区域的外面**，而不违背 Item 66 中的指导方针。





上部分是关于「**正确性**」的，下面讨论一下「**性能**」。虽然同步的成本已经岁版本更新下降了很多，但永远也不要过度使用同步。在这个多核的时代，过度同步的实际成本不是说获取锁所花费的 CPU 实际，而是指「**失去了并行的机会**」，以及因为要确保每个核都有一个「**一致的内存视图**」而导致的延迟。过度同步的另一项潜在的开销在于它会限制虚拟机优化代码执行的能力。



如果一个可变类要并发使用，应该使这个类变成线程安全的。**通过内部同步，可以获得明显比外部锁定整个对象更高的并发性，否则就不要在内部同步**，让客户端在必要的时候从外部同步。Java 平台早期很多违背了很多指导方针，比如 `StringBuffer`实例几乎总是被用于单个线程中，单证还行的却是内部同步，现在它基本已经被 `StringBuilder`代替，它是非同步版本。

**当你不确定的时候，就不要同步你的类，而是建立文档，注明它不是线程安全的**。



如果在内部同步了类，就可以使用不同的方法来实现高并发性，比如分段锁（lock splitting）、分离锁（lock striping）和非阻塞（nonblocking）并发控制。



如果方法修改了静态域，那么你也必须同步对这个域的访问，即使它往往只用于单个线程。客户要在这种方法上执行外部同步是不可能的，因为不可能保证其他不相关的客户也会执行外部同步，比如之前那个 `generateSerialNumber`方法。





简而言之，为了避免死锁和数据破坏，千万不要从同步区域内部调用外来方法。更为一般地讲，要尽量限制同步区域内部的工作量。当你在设计一个可变类的时候，要考虑一下它们是否应该自己完成同步操作。在现在这个多核的时代，这比永远不要过度同步来得更重要。只有当你有足够的理由一定要在内部同步类的时候，才应该这么做，同时还应该将这个决定清楚地写到文档中 Item 70。



