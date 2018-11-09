# Item5: 避免创建不必要的对象

最好能够重用对象。

如果对象是不可变的（immutable)，它就始终可以被重用。

## string 的例子

极端反面例子：下面的语句每次执行都会创建新的实例，但这是不必要的。

```java
String s = new String("stringette); // DON'T DO THIS!
```

特别是不能放在循环中。



改进：`string s = "stringette`


这个版本只用了一个 String 实例， **而且，对于所有同一台虚拟机中运行的代码，只要包含相同的字符串字面常量，该对象就会被重用**


## 静态工厂

对于提供了静态工厂方法、和构造器的不变类，通常可以使用静态工厂而不是构造器，以避免创建不必要的对象 。例如静态工厂方法`Boolean.valueOf(String)` 几乎总是优先于构造器 `Boolean(String)`。



## 重用已知不会被修改的可变对象

除了重用不可变对象外，也可以重用已知不会被修改对象。

比如 Date 对象可变，但值算出来后就不再变化。

```java
class Person(){
  private final Date birthDate;
  private static final Date BOOM_START;
  private static final Date BOOM_END;

  static {
    Calendar gmtCal = Calendar.getInstance(TimeZone.getTimeZone("GMT"));
    gmtCal.set(1946, Calendar.JANUARY, 1, 0, 0 , 0);
    BOOM_START = gmtCal.getTime();
    gmtCal.set(1965, Calendar.JANUARY, 1, 0, 0 , 0);
    BOOM_END = gmtCal.getTime();
  }

  public boolean isBabyBoomer(){
    return birthDate.compareTo(BOOM_START) >= 0 &&
           birthDate.compareTo(BOOM_END) < 0;
  }
}

```

用一个静态初始化块来，避免每次创造相同的 Date 实例，防止频繁被调用。


> 如果使用 static 修饰初始化块，就称为静态初始化块。在类的声明中，可以包含多个初始化块，虚拟机加载类的时候，会依次执行这些代码块。
> 需要特别注意：静态初始化块只在类加载时执行，且只会执行一次，同时静态初始化块只能给静态变量赋值，不能初始化普通的成员变量。


## 适配器 或 视图的情形

适配器是这样一个对象：把功能委托给后备对象，为后备独享提供一个可代替的接口。由于适配器除了后备对象之外，没有其他信息，所以针对给某个给定对象的特定适配器，不需要有多个实例。

比如 Map 接口的 keySet 方法返回 Map 对象的 Set 视图，其中包含 Map 的所有键，粗看起来每次调用 keySet 都应该创建一个 Set 实例。但是对一个给定得到 Map 对象，每次调用 keySet 返回同样的 Set 实例。虽然被返回的 Set 实例一般是可改变的，但所有返回的对象在功能上是等同的：当其中一个返回对象发生变化的时候，所有其他的返回对象也要发生变化，因为它们由同一个 Map 实例支撑。虽然创建 keySet 视图对象的多个实例并无害处，但也没有必要。


## 自动装箱

Java 1.5 后，有种创造多余对象的新方法，称为自动装箱（autoboxing)，允许将基本类型和装箱基本类型（Boxed Primitive Type）混用，按需要自动装箱和拆箱。虽然差别模糊起来，但并未完全消除。

```java
// 超级慢
public static void main(String args[]){
    Long sum = 0L;
    for(long i = 0; i < Integer.MAX_VALUE; i++){
        sum ++ i;
    }
    System.out.println(sum);
}
```


为什么这么慢？因为 Sum 被声明为 Long 而不是 long，意味着程序创造了 2^31 个多余的的实例（每次往Long sum 中增加 long 时构造一个实例）。

不要认为这里个例子暗示“创造对象的代价非常昂贵，应该尽可能避免创造对象”，相反，小对象的创建和回收是非常廉价的，通过创建附加的对象，提升程序的清晰性、简洁性、功能性，通常是好事。

通过维护自己的对象池 object pool 来避免创建对象不是一种好的做法，除非对象池里面的对象是非常重量级的：比如数据库连接池（连接很昂贵），而且数据库限制一定数量的连接。

一般而言，维护自己的对象池必定把代码弄得很乱，同时增加内存占用（footprint），还会增加性能。





