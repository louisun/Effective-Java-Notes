# Item6. 消除过期的对象引用

内存泄露什么时候容易发生？


## 1. 自己管理逻辑上的内存池

是否能发现下面的代码会导致内存泄露？

```java
package effectivejava.chapter2.item7;
import java.util.*;

// Can you spot the "memory leak"? 
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        return elements[--size];
    }

    /**
     * Ensure space for at least one more element, roughly
     * doubling the capacity each time the array needs to grow.
     */
    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }

//    // Corrected version of pop method
//    public Object pop() {
//        if (size == 0)
//            throw new EmptyStackException();
//        Object result = elements[--size];
//        elements[size] = null; // Eliminate obsolete reference
//        return result;
//    }

    public static void main(String[] args) {
        Stack stack = new Stack();
        for (String arg : args)
            stack.push(arg);

        while (true)
            System.err.println(stack.pop());
    }
}

```

内存泄露：过期对象没有回收。

在支持垃圾回收的语言中，内存泄露是隐蔽的（这类内存泄露称为“无意识地对象保持”），对性能造成潜在重大影响。


上面 `pop()` 方法实际并没有将弹出的元素从栈里面删除，怎么删除？直接将这个对象赋值为 `null`。

```java
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null; // Eliminate obsolete reference
    return result;
}
```

清空过期错误的另一个好处：以后又被错误地解除引用会立即抛出 NullPointerException。

并不是说每一个对象都要小心翼翼，不用它了就清除它赋值为 null，也就是说清空对象引用是一种例外（特殊情况），而不是一种规范行为。

什么时候应该清空引用？

Stack 类由于自己管理内存，存储池包含了 elements 数组（对象引用单元，而不是对象本身）的元素，它自定义了已分配元素区域和自由区域，但垃圾回收期不知道这一点，对它来说所有元素都是同等有效的。所以程序员应该把情况告诉垃圾回收期，即数组变成非活动元素的一部分，手工清空这些元素。

> 一般而言，只要是类自己管理内存，就应该警惕内存泄露问题。

## 2. 缓存

一旦把对象放到缓存中，它容易被遗忘掉。

## 3. 监听器和其他回调

加入实现了一个 API，客户端在 API 中注册回调，缺没有显示注册，除非采取行动，它们会聚集。确保回调立即被当做垃圾回收的最佳方法是只保存它们的弱引用，例如，只将它们保存为 WeakHashMap 中的键。

