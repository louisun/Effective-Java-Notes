# Item9: 覆盖 equals 时总要覆盖 hashCode

在每个类中，在重写 `equals` 方法的时侯，一定要重写 `hashcode` 方法。如果不这样做，你的类违反了hashCode的通用约定，这会阻止它在`HashMap`和`HashSet`这样的集合中正常工作。根据 Object 规范，以下是具体约定。



1. **当在一个应用程序执行过程中，如果在equals方法比较中没有修改任何信息，在一个对象上重复调用hashCode方法时，它必须始终返回相同的值。**从一个应用程序到另一个应用程序的每一次执行返回的值可以是不一致的。

2. **如果两个对象根据equals(Object)方法比较是相等的，那么在两个对象上调用hashCode就必须产生的结果是相同的整数。**

3. 如果两**个对象根据equals(Object)方法比较并不相等，则不要求在每个对象上调用hashCode都必须产生不同的结果。** 但是，程序员应该意识到，为不相等的对象生成不同的结果可能会提高散列表（hash tables）的性能。

**当无法重写hashCode时，所违反第二个关键条款是：相等的对象必须具有相等的哈希码（ hash codes）。**



根据类的equals方法，两个不同的实例可能在逻辑上是相同的，但是对于Object 类的hashCode方法，它们只是两个没有什么共同之处的对象。因此， Object 类的hashCode方法返回两个看似随机的数字，而不是按约定要求的两个相等的数字。

```java
Map<PhoneNumber, String> m = new HashMap<>();

m.put(new PhoneNumber(707, 867, 5309), "Jenny");
```

你可能期望`m.get(new PhoneNumber(707, 867, 5309))`方法返回`Jenny`字符串，但实际上，返回了 `null`。注意，这里涉及到两个`PhoneNumber`实例：一个实例插入到 HashMap 中，另一个作为判断相等的实例用来检索。**`PhoneNumber`类没有重写 hashCode 方法导致两个相等的实例返回了不同的哈希码，违反了 hashCode 约定**。put 方法把`PhoneNumber`实例保存在了一个哈希桶（ hash bucket）中，但get方法却是从不同的哈希桶中去查找，即使恰好两个实例放在同一个哈希桶中，get 方法几乎肯定也会返回 null。因为HashMap 做了优化，缓存了与每一项（entry）相关的哈希码，如果哈希码不匹配，则不会检查对象是否相等了。

解决这个问题很简单，只需要为`PhoneNumber`类重写一个合适的 hashCode 方法。



## 编写 hashCode 



一个好的 hash 方法趋向于为不相等的实例生成不相等的哈希码。这也正是 hashCode 约定中第三条的表达。理想情况下，hash 方法为集合中不相等的实例均匀地分配int 范围内的哈希码。实现这种理想情况可能是困难的。 幸运的是，要获得一个合理的近似的方式并不难。 以下是一个简单的配方：

1. 声明一个 `int` 类型的变量 `result`，并将其初始化为对象中**第一个重要属性`c`的哈希码**

2. 对于对象中剩余的重要属性 `f`，

   1. 比较属性`f`与属性`c`的 `int` 类型的哈希码：
      1. 如果这个属性是基本类型的，使用`Type.hashCode(f)`方法计算，其中`Type`类是对应属性 `f` 基本类型的包装类。
      2. 如果该属性是一个对象引用，并且该类的equals方法通过递归调用equals来比较该属性，**并递归地调用hashCode方法。** 如果需要更复杂的比较，则计算此字段的“范式（“canonical representation）”，并在范式上调用hashCode。 如果该字段的值为空，则使用0（也可以使用其他常数，但通常来使用0表示）。
      3. 如果属性`f`是一个数组，把它看作每个重要的元素都是一个独立的属性。 也就是说，通过**递归地应用这些规则计算每个重要元素的哈希码**，并且将每个步骤`2.1.2`的值合并。 如果数组没有重要的元素，则使用一个常量，最好不要为 0。如果所有元素都很重要，则使用`Arrays.hashCode`方法。
   2.  将步骤2.1中属性c计算出的哈希码合并为如下结果：`result = 31 * result + c;`

3. 返回 result 值



   步骤2.2中的乘法计算结果取决于属性的顺序，如果类中具有多个相似属性，则产生更好的散列函数。 例如，如果乘法计算从一个String散列函数中被省略，则所有的字符将具有相同的散列码。 之所以选择31，因为它是一个奇数的素数。 如果它是偶数，并且乘法溢出，信息将会丢失，因为乘以2相当于移位。 使用素数的好处不太明显，但习惯上都是这么做的。 31的一个很好的特性，是在一些体系结构中乘法可以被替换为移位和减法以获得更好的性能：`31 * i ==（i << 5） - i`。 现代JVM可以自动进行这种优化。


```java
// Typical hashCode method

@Override public int hashCode() {

    int result = Short.hashCode(areaCode);

    result = 31 * result + Short.hashCode(prefix);

    result = 31 * result + Short.hashCode(lineNum);

    return result;

}
```



因为这个方法返回一个简单的确定性计算的结果，它的唯一的输入是`PhoneNumber`实例中的三个重要的属性，所以显然相等的`PhoneNumber`实例具有相同的哈希码。 实际上，这个方法是`PhoneNumber`的一个非常好的hashCode实现，与Java平台类库中的实现一样。 它很简单，速度相当快，并且合理地将不相同的电话号码分散到不同的哈希桶中。



#### 一行 hashCode：用Objects.hash(任意参数) 

`Objects`类有一个静态方法，**它接受任意数量的对象**并为它们返回一个哈希码。 这个名为hash的方法可以让你编写一行hashCode方法，其质量与根据这个项目中的上面编写的方法相当。 不幸的是，它们的运行速度更慢，因为它们需要创建数组以传递可变数量的参数，以及如果任何参数是基本类型，则进行装箱和取消装箱。 这种哈希函数的风格建议仅在性能不重要的情况下使用。 以下是使用这种技术编写的`PhoneNumber`的哈希函数：

```java
// One-line hashCode method - mediocre performance

@Override public int hashCode() {
   return Objects.hash(lineNum, prefix, areaCode);
}
```



### 缓存哈希码

如果一个类是不可变的，并且计算哈希码的代价很大，那么可以考虑在对象中缓存哈希码，而不是在每次请求时重新计算哈希码。 **如果你认为这种类型的大多数对象将被用作哈希键，那么应该在创建实例时计算哈希码。 否则，可以选择在首次调用`hashCode`时延迟初始化（lazily initialize）哈希码。** 需要注意确保类在存在延迟初始化属性的情况下保持线程安全。 `PhoneNumber`类不适合这种情况，但只是为了展示它是如何完成的。 请注意，属性hashCode的初始值（在本例中为0）不应该是通常创建的实例的哈希码：

```java
// hashCode method with lazily initialized cached hash code

private int hashCode; // Automatically initialized to 0

@Override public int hashCode() {
    int result = hashCode;
    if (result == 0) {
        result = Short.hashCode(areaCode);
        result = 31 * result + Short.hashCode(prefix);
        result = 31 * result + Short.hashCode(lineNum);
        hashCode = result;
    }
    return result;
}
```

1. **不要试图从哈希码计算中排除重要的属性来提高性能。**在Java 2之前，String 类哈希函数在整个字符串中最多使用16个字符，从第一个字符开始，在整个字符串中均匀地选取。 对于大量的带有层次名称的集合（如URL），此功能正好显示了前面描述的病态行为。

2. **不要为hashCode返回的值提供详细的规范，因此客户端不能合理地依赖它； 你可以改变它的灵活性。** Java类库中的许多类（例如String和Integer）都将hashCode方法返回的确切值指定为实例值的函数。 这不是一个好主意，而是一个我们不得不忍受的错误：它妨碍了在未来版本中改进哈希函数的能力。 如果未指定细节并在散列函数中发现缺陷，或者发现了更好的哈希函数，则可以在后续版本中对其进行更改。



## 总结



总之，每次重写equals方法时都必须重写hashCode方法，否则程序将无法正常运行。你的hashCode方法必须遵从Object类指定的常规约定，并且必须执行合理的工作，将不相等的哈希码分配给不相等的实例。如果使用第51页的配方，这很容易实现。如条目 10所述，AutoValue框架为手动编写equals和hashCode方法提供了一个很好的选择，IDE也提供了一些这样的功能。



