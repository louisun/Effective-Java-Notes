# 3rd Item 5: 使用依赖注入取代硬连接资源(hardwiring resources)

**许多类依赖于一个或多个底层资源。** 例如，拼写检查器 `SpellChecker` 依赖于字典 dictionary。


将此类类实现为**静态实用工具类**并不少见

```java
// Inappropriate use of static utility - inflexible & untestable!
public class SpellChecker {
    private static final Lexicon dictionary = ...;

    private SpellChecker() {} // Noninstantiable

    public static boolean isValid(String word) { ... }
    public static List<String> suggestions(String typo) { ... }
} 
```

## 反例：单例

同样地，将它们实现为**单例**也并不少见:


```java
// Inappropriate use of singleton - inflexible & untestable!
public class SpellChecker {
    private final Lexicon dictionary = ...;

    private SpellChecker(...) {}
    public static INSTANCE = new SpellChecker(...);

    public boolean isValid(String word) { ... }
    public List<String> suggestions(String typo) { ... }
}
```


这两种方法都不令人满意，因为他们假设只有一本字典值得使用。在实际中，每种语言都有自己的字典，特殊的字典被用于特殊的词汇表。另外，使用专门的字典来进行测试也是可取的。想当然地认为一本字典就足够了，这是一厢情愿的想法。


可以通过使`dictionary`属性设置为**非final**，并添加一个方法来更改现有的字典，从而让 SpellChecker 支持多个字典，但是在并发环境中，这是笨拙的、容易出错的和不可行的。

**静态实用类和单例对于那些行为被底层资源参数化的类来说是不合适的。**



## 依赖注入


**依赖注入模式非常简单，许多程序员使用它多年而不知道它有一个名字。**说白了就是往构造函数里传东西。


虽然我们的拼写检查器的例子只有一个资源（字典），但是**依赖项注入可以使用任意数量的资源和任意依赖图**。

它保持了**不变性**，因此多个客户端可以共享依赖对象（假设客户需要相同的底层资源）。 依赖注入同样适用于构造方法，静态工厂和 builder模式。

```java
// Dependency injection provides flexibility and testability
public class SpellChecker {
    private final Lexicon dictionary;

    public SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    }

    public boolean isValid(String word) { ... }
    public List<String> suggestions(String typo) { ... }
}
```

该模式的一个有用的变体是**将资源工厂传递给构造方法**。 工厂是可以重复调用以创建类型实例的对象。 这种工厂体现了工厂方法模式。 

Java 8中引入的Supplier <T>接口非常适合代表工厂。 在输入上采用`Supplier<T>`的方法通常应该使用有界的通配符类型( bounded wildcard type)约束工厂的类型参数，以允许客户端传入工厂，创建指定类型的任何子类型。 例如，下面是一个使用客户端提供的工厂生成tile的方法：

```java
Mosaic create(Supplier<? extends Tile> tileFactory) { ... }
```


尽管依赖注入极大地提高了灵活性和可测试性，但它可能使大型项目变得混乱，这些项目通常包含数千个依赖项。使用依赖注入框架(如Dagger、GuiceSpring)可以消除这些混乱。



> 总之，不要使用单例或静态的实用类来实现一个类，该类依赖于一个或多个底层资源，这些资源的行为会影响类的行为，并且不让类直接创建这些资源。
> 
> 相反，将资源或工厂传递给构造方法(或静态工厂或builder模式)。这种称为依赖注入的实践将极大地增强类的灵活性、可重用性和可测试性。