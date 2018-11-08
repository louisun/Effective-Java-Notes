# Item3: 用私有构造器或枚举类型强化Singleton属性

Singleton 指仅被实例化一次的类，通常代表本质上唯一的系统组件。

## 方法一：公有静态 final 成员（类自身）

被称为：公有域方法（public final field)

私有构造器仅被调用一次

```java
// Singleton with public final field
public class Elvis {
    // 共有的静态域
    public static final Elvis INSTANCE = new Elvis();
    // 私有构造器
    private Elvis() { }

    public void leaveTheBuilding() {
        System.out.println("Whoa baby, I'm outta here!");
    }

    // This code would normally appear outside the class!
    public static void main(String[] args) {
        Elvis elvis = Elvis.INSTANCE;
        elvis.leaveTheBuilding();
    }
}
```

## 方法二：公有静态工厂方法返回私有静态域

静态工厂都会返回同一个对象的引用

```java
// Singleton with static factory
public class Elvis {
    // 私有静态域
    private static final Elvis INSTANCE = new Elvis();
    // 私有构造器
    private Elvis() { }
    // 静态工厂方法
    public static Elvis getInstance() { return INSTANCE; }

    public void leaveTheBuilding() {
        System.out.println("Whoa baby, I'm outta here!");
    }

    // This code would normally appear outside the class!
    public static void main(String[] args) {
        Elvis elvis = Elvis.getInstance();
        elvis.leaveTheBuilding();
    }
}
```

这种（组成类的成员的）声明很清楚地表示了这个类是一个 Signleton，公有的静态域是 final 的。

**工厂方法的优势：**

1. 提供了灵活性，在不改变 API 的前提下，可以改变类是否应该为 Singleton 的想法；
2. 与泛型有关 [(Item 30)]()




---

为了使上面任一方法实现的 Singleton 类变成是 Serializable 的，仅仅在声明中添加 “implements Serializable” 的不够的。为了维护并保证 Singleton，必须声明所有实例域是 transient 瞬时的，并提供一个 `readResolve()` 方法 [(Item 89)]()。否则，每次反序列化一个序列化实例时，都会创建一个新的实例，在我们这个例子中就会返回“假冒的Elvis”，防止这种情况，Elvis 类中还要添加 `readResolve` 方法。



```java
// readResolve method to preserve singleton property
private Object readResolve(){
    // return the one true Elvis and let the garbage collector
    // take care of the Elvis impersonator.
    return INSTANCE;
}
```

## 方法三：包含单个元素的枚举类型

Java 1.5 开始，这种方法与公有方法相近，但是更加简洁，无偿提供了序列化机制，绝对防止多次序列化。

这种方法看起来虽然有点不自然，但单元素的枚举类型已成为实现 Singleton 的最佳方法。


```java
// Enum singleton - the preferred approach
public enum Elvis {
    INSTANCE;

    public void leaveTheBuilding() {
        System.out.println("Whoa baby, I'm outta here!");
    }

    // This code would normally appear outside the class!
    public static void main(String[] args) {
        Elvis elvis = Elvis.INSTANCE;
        elvis.leaveTheBuilding();
    }
}

```

> 注意如果单例必须继承自一个不是 Enum 的父类，就不能使用这个方法，虽然可以声明 enum 来实现接口


