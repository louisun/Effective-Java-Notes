# Item39. 必要时进行保护性拷贝

> 保护性拷贝：defensive copy

一、对构造函数的可变参数进行保护性拷贝；
二、对可变域的访问方法，只返回可变域的保护性拷贝

---



Java 是一门安全的语言。这意味着，它对于缓冲区溢出、数组越界、非法指针以及其他的内存破坏错误都自动免疫。但即使在安全的语言中，如果不采取一点措施，还是无法与其他的类隔离开来。假设类的客户端会尽其所能来破坏这个类的约柬条件，因此你必须保护性地设计程序。



实际上，只有当有人试图破坏系统的安全性时，才可能发生这种情形：更有可能的是，对你的 API 产生误解的程序员，所导致的各种不可预期的行为，只好由类来处理。无论是哪种情况，**编写一些面对客户的不良行为时仍能保持健壮性的类，这是非常值得投入时间去做的事情。**



虽然另一个类单独不可能修改对象的内部状态，但是对象容易在无意识的情况下帮助另一个类修改该对象类的内部状态。



```java
package effectivejava.chapter8.item50;
import java.util.*;

// Broken "immutable" time period class (Pages 231-3)
public final class Period {
    private final Date start;
    private final Date end;

    /**
     * @param  start the beginning of the period
     * @param  end the end of the period; must not precede start
     * @throws IllegalArgumentException if start is after end
     * @throws NullPointerException if start or end is null
     */
    public Period(Date start, Date end) {
        if (start.compareTo(end) > 0)
            throw new IllegalArgumentException(
                    start + " after " + end);
        this.start = start;
        this.end   = end;
    }

    public Date start() {
        return start;
    }
    public Date end() {
        return end;
    }

    public String toString() {
        return start + " - " + end;
    }

}
```



看起来 `Period` 类是不可变的，比如私有静态常量，只提供了 `get` 访问方法，加了约束条件。但其实可以这样做改变这个类的内部状态：

```java
Date start = new Date();
Date end = new Date();
Period p = new Period(start, end);
end.setYear(78);
```



也就是说，这个 `end` 本来就是外部传入的，外部的 `end` 对象是可修改的。



为了保护 `Period` 实例的内部信息避免受到这种攻击，对于构造器的每个可变参数进行保护性拷贝（ defensive copy）是必要的，并且**使用备份对象作为 `Period`实例的组件，而不使用原始的对象**

```java
// Repaired constructor - makes defensive copies of parameters (Page 232)
public Period(Date start, Date end) {
    // 内部变量只是外部的拷贝
    this.start = new Date(start.getTime());
    this.end   = new Date(end.getTime());

    if (this.start.compareTo(this.end) > 0)
        throw new IllegalArgumentException(
        this.start + " after " + this.end);
}


```



注意：保护性拷贝是在检查参数有效性之前进行的，并且有效性检查是针对拷贝后的对象，而非原始对象。这样可以避免在「危险阶段」期间**从另一个线程改变类的参数。**比如：A 线程正在有效性检查，其实即将通过了，但 B 线程也是调用这个构造函数，已经通过了参数检查，并对常量 start end 进行初始化了，这样 A 继续执行下去就会报错，无法再对常量赋值。这被称为：Time-Of-Check / Time-Of-Use 攻击。



同时也请注意，在构造器中，我们没有用`Date`的 `clone`方法来进行保护性拷贝。因为 外部的`Date`是非 `final`的，可以被子类继承，不能保证 `clone`方法一定返回类为`java.util.Date`的对象：它有可能返回专门出于恶意的目的而设计的**不可信子类**的实例。例如，这样的子类可以在毎个实例被创建的时候，把指向该实例的引用记录到一个私有的静态列表中，并且允许攻击者访问这个列表。这将使得攻击者可以自由地控制所有的实例。为了阻止这种攻击，**对于参数类型可以被不可信任方子类化的参数，请不要使用 `clone`方法进行保护性拷贝**。



> 还有就是 clone 返回的是浅拷贝。



另一种攻击方式：



```java
Date start = new Date();
Date end = new Date();
Period p = new Period(start, end);
p.end.setYear(78);  // 通过返回值修改内部状态
```



只要修改访问方法，返回内部域的保护性拷贝：

```java
// Repaired accessors - make defensive copies of internal fields
// 获取内部对象，也是返回一个拷贝
public Date start() {
    return new Date(start.getTime());
}

public Date end() {
    return new Date(end.getTime());
}
```



这样类 `Period` 是真正不可变了。此处访问器和构造方法不同，在进行保护性拷贝的时候可以使用 `clone` 方法，因为我们知道内部的类就是 `Date`。





参数保护性拷贝不仅仅是针对「不可变」类，当编写方法或构造器时，如果要「允许客户提供的对象进入到内部数据结构中」，则有必要考虑一下，也就是说这个对象进入数据结构之后是否能由于外界改变而变化。如果不能这样，应该进行保护性拷贝。**比如某对象作为 Map 的 key时。** 



然后类的内部组件被返回到客户端前，对其进行保护性拷贝也是同样的道理。比如返回内部数组，可以进行保护性拷贝，也可以返回数组的不可变视图。





![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-07-23-08-29_r52.png)





![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-07-23-08-46_r98.png)





可以肯定地说，只要有可能，都应该使用「不可变对象」作为对象内部组件，这样就不必再为保护性拷贝操心。其实前面有经验的程序员会用 `Date.getTime()` 返回的 `long` 基本类型作为内部时间表示法，而不是 `Date`对象引用，这是因为 `Date` 是可变的。



简而言之，如果类具有从客户端得到或者返回到客户端的**可变组件，类就必须保护性地拷贝这些组件**。如果拷贝的成本受到限制，并且类信任它的客户端不会不恰当地修改组件，就可以在文档中指明客户端的职责是不得修改受到影响的组件，以此来代替保护性拷贝。

 











