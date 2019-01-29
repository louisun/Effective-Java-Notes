# Item21. （过时）表示策略：Lambda！函数接口过时了



有些语言支持函数指针、代理、`lambda` 表达式，或者支持类似的机制，**允许程序把“调用特殊函数的能力”储存起来并传递这种能力。**这种机制通常用于允许函数的调用者通过传入第二个函数，来指定自己的行为。

什么是函数对象？实际上这是在`JDK8`之前没有 Java 不支持`lamda`表达式，方法参数不能传递一个方法只能通过传递对象的方式“曲线救国”，例如`Arrays.sort(T[] a, Comparator<? super T> c)`方法，第一个参数传递数组，根据传入第二个自定义的比较类中的比较方法进行排序。



**如果能传入函数指针、Lambda表达式等，那就自然不用传递一个类。**



## 函数对象

```java
class StringLengthComparator {
    public int compare(String s1, String s2) {
        return s1.length() - s2.length();
    }
}
```



如果类仅仅导出这样一个方法，它的实例实际上就等同于指向该方法的指针，这样的实例被称为函数对象。



这个类导出一个带两个字符串参数的方法。指向`StringLengthComparator`对象的引用可以被当做是一个指向该比较器的“函数指针”，可以在任意一对字符串上被调用。换句话说，`StringLengthComparator`实例是用于字符串比较操作的**具体策略**。作为典型的具体策略类，`StringLengthComparator`类是无状态的：它没有域，所以，这个类的所有实例在功能上是相互等价的。因此，它作为一个`Singleton`是非常合适的，可以节省不必要的对象创建开销：

```java
public class StringLengthComparator {
    private StringLengthComparator() {}
     
    public static final StringLengthComparator INSTANCE = new StringLengthComparator();
     
    public int compare(String s1, String s2) {
        return s1.length() - s2.length();
    }
}
```



为了把`StringLengthComparator`实例传递给方法，需要适当的参数类型。使用`StringLengthComparator`并不好，因为客户端无法传递任何其他的比较策略。相反，我们需要定义一个`Comparator`接口，并修改`StringLengthComparator`来实现这个接口。换句话说，我们在设计具体的策略类时，还需要定义一个策略接口：

```java
// Strategy interface
public interface Comparator<T> {
    public int compare(T t1, T t2);
}
```



`Comparator` 接口的这个定义碰巧也出现在`java.util`包中，但是这并不神奇，我们自己也可以定义它。`Comparator`接口时范型的，因此它适合作为除字符串之外其他对象的比较器。它的`compare`两个参数类型为`T`，而不是`String`。只要声明前面所示的`StringLengthComparator`类要这么做，就可以用它实现`Comparator<String>`接口。

具体的策略类往往使用匿名类声明，下面的语句根据长度对一个字符串数组进行排序：

```java
Arrays.sort(stringArray, new Comparator<T>() {
    @Override
    public int compare(String s1, String s2) {
        // TODO Auto-generated method stub
        return s1.length() - s2.length();
    }
});
```



但是注意，以这种方式使用匿名类时，将会在每次执行调用的时候创建一个新的实例。如果它被重复执行，考虑将函数对象存储到一个私有的静态 final 域里，并重用它。这样做的另一种好处是，可以为这个函数对象取一个有意义的域名。

因为策略接口被用作所有具体策略实例的类型，所以并不需要为了导出具体策略，而把策略类做成公有的。相反“宿主类”还可以导出公有的静态域，其类型为策略接口，具体的策略类可以是宿主类的私有前套类。下面的例子使用静态成员类，而不是匿名类，以便允许具体的策略类实现第二个接口 `Serializable`



```java
/**
 * 用函数对象表示策略
 *
 */
public class Host {
    private static class StrLenCmp implements Comparator<String>, Serializable {
 
        private static final long serialVersionUID = -5797980299250787300L;
 
        public int compare(String s1, String s2) {
            return s1.length() - s2.length();
        }
    }
     
    // Return comparator is serializable
    public static final Comparator<String> STRING_LENGTH_COMPARATOR = new StrLenCmp();
     
    public static void main(String[] args) {
        System.out.println(STRING_LENGTH_COMPARATOR.compare("aaaaaa", "aaaaa"));
    }
}
```



String 类利用这种模式，通过它的`STRING_LENGTH_COMPARATOR`域，导出一个不区分大小写的字符串比较器。



总而言之，函数指针的主要用途就是实现策略模式。在JDK8之前，为了在 java中实现这种模式，要声明一个接口来表示该策略，并且为每一个策略声明一个实现了该接口的类。当一个具体策略只被使用一次时，通常使用匿名类来声明和实例化这个具体策略类。当一个具体策略类时设计用来重复使用的时候，它的类通常就要被实现为私有的静态成员类，并通过公有的静态final域被导出，其类型为该策略接口。

## Lambda 表达式

从 JDK8 开始 Java 已经支持了lambda表达式.

**`lambda` 表达式不过是一个匿名方法实现而已**。

广义上来讲JDK8中lambda表达式有两个部分组成：一是 lambda 表达式本身，二是**函数式接口**。**函数式接口实际上就是指只包含一个抽象方法的接口**，比如 Runnable 接口只包含 run 抽象方法。而 lambda 表达式本身实际上则是对抽象方法的实现。



首先lambda表达式的语法格式如下所示：

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2018-11-27-20-39-46_r34.png)



例如：

```java
(n）->System.out.println(n);

// 相当于如下 demo 方法
public void demo(String n){

   System.out.println(n);

}
 
// 上面 lambda 表达式对应的接口
 public interface LambdaDemo {
     void demo(String n);
 }

 // lambda 表达式的使用
 public class App {
     public static void main( String[] args )  {
         //实例化LambdaDemo类，同时也 lambda 表达式实现了 demo 方法
         LambdaDemo lambdaDemo = (n) -> System.out.println(n);     
         lambdaDemo.demo("hello lambda");
     }
 }


```



上面的 `sort ` 字符串可以这么写：

```java
Comparator<String> sortByName = (String s1, String s2) -> (s1.compareTo(s2));  
Arrays.sort(players, sortByName);  
 
// 或者一步到位，但是 comparator 不能复用
Arrays.sort(players, (String s1, String s2) -> (s1.compareTo(s2)));  
```

 