# Item42. 慎用可变参数

Java 1.5 增加了可变参数（varargs）方法，一般乘坐「variable artiy method」（可匹配不同长度的变量的方法）。可变参数方法接受 0 个或多个指定类型的参数。



可变参数机制是先创建一个数组，数组大小为调用位置所传递的参数数量，然后将参数值传到数组中，最后将数组传递给方法。



```java
static int sum(int... args) {
    int sum = 0;
    for (int arg : args)
        sum += arg;
    return sum;
}

sum(1, 2, 3)
// 6
```



有时候要编写需要 1 个或多个「多种类型参数」的方法，而不是需要 0 个或多个。例如，要计算多个 int 参数的最小值。如果在运行时检查数组长度：

```java
// 错误地使用 vargs 来传递 1 个或多个参数
static int min(int... args) {
    if (args.length == 0)
        throw new IllegalArgumentException("Too few arguments");
    int min = args[0];
    for (int i = 1; i < args.length; i++)
        if (args[i] < min)
            min = args[i];
    return min;
}
```



上面这样解决的一个问题是，如果调用时没有传参数进去，只能在「运行时」失败，而不是编译时失败。另一个问题是，这段代码很不美观，也不能随便地进行 for-each 循环。



## 正常参数 + 可变参数



有一种更好的方法：



```java
// 一个正常参数 + 可变参数来传递 1 个或多个参数
static int min(int firstArg, int... remainingArgs) {
    int min = firstArg;
    for (int arg : remainingArgs)
        if (arg < min)
            min = arg;
    return min;
}
```



## 不必将数组作为final参数的方法改造为可变参数



比如 `Arrays.asList()` ，在 Java 1.5 之前是传入一个 final 数组，转换为一个 List 数组：

```java
Arrays.asList(myArray); // myArray 是个数组
```



但在 Java 1.5 时把其参数变成了「可变参数」，可以下面这样做：

```java
Arrays.asList(1, 2, 3); // 每个参数都是数组中的元素
```



```java
// a 中元素的类型是 T
public static <T> List<T> asList(T... a) {
    return new ArrayList<>(a);
}
```

如果一不小心，传一个「**基本类型的数组**」给 `Arrays.asList`，就会出问题：**传入的整个数组变成了参数数组中的一个元素。**

```java
int[] datas = new int[]{1,2,3,4,5};
Arrays.asList(datas); // 可变参数数组的长度是 1 ，里面放了 datas 数组
```

**这是由于基本类型不支持泛型化**，也就是说 `int` 不能被泛型化，但是 `int[]` 数组支持，因此数组中的元素类型变成了 `int[]`。要把基本类型数组改成装箱类型数组才行。



```java
Integer[] datas = new Integer[]{1,2,3,4,5};
Arrays.asList(datas)	// 可变参数类型长度是 5
```



> 教训：不必改造**具有 final 数组参数的每个方法**，只当确实是在「**数量不定的值上执行调用**」时才使用**可变参数**。



```java
ReturnType1 suspect1(Object... arsg){}

<T> ReturnType2 suspect2(T... args){}
```



这两个方法签名都很可疑，**因为都能接受任何参数列表**，改造为可变参数之前的任何编译时类型检查都会丢失，比如 `Arrayys.asList` 的情形





## 性能考虑



可变参数方法的每次调用都会导致「**进行一次数组分配和初始化**」，如果凭经验无法承受这个成本，有需要灵活性，可以考虑用「方法重载」。假设确定对某个方法 95% 的调用会有 3 个或更少的参数，就要应该声明该方法的 5 个重载，每个重载方法带 0-3 个普通参数，当参数超过 3 个时，就可以使用一个可变参数方法：



![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-15-14-15-38_r3.png)



这种方法通常不太恰当，但当真正需要它时，就会帮上忙。



`EnumSet` 类对它的静态工厂就是使用这种方法，可以最大限度地减少创建枚举集合的成本。这样有必要，因为枚举集合要为位域提供性能方面有竞争力的替代方法。



## 总结



简而言之，在定义参数数目不定的方法时，可变参数方法是一种方便的方式，但也不能过度滥用。使用不当会有混乱的结果。