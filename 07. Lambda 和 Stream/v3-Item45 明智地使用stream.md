# v3-Item45 明智地使用stream

Java 8 添加了 stream API，减轻批量（串行或并行）操作任务的难度。这个 API 提供了 2 个关键抽象：

1. stream，代表有限或无限的数据元素
2. stream pipeline（流管道），代表在这些元素上多阶段的操作

stream 中的元素可以来自任何地方，通常会来自集合，数组，文件，正则表达式模式匹配器，伪随机数生成器等。stream 中的数据元素可以是「对象引用」或者「基本类型」。支持 3 种基本类型：`int`、`long`、`double`。



## 介绍

stream pipleline 由 source stream 后跟零个或多个中间操作和一个终端操作组成。 每个中间操作以某种方式转换流，例如将每个元素映射到该元素的函数，或过滤掉不满足某些条件的所有元素。中间操作都将一个流转换为另一个流，其元素类型可以与输入流「**相同**」或「**不同**」。 终端操作对从最后的中间操作产生的流执行最终计算，例如将其元素存储到集合中，返回某个元素或打印其所有元素。

stream pipeline 是惰性求值的：在调用「**终端操作**」之前不会开始求值，并且对终端操作计算无用的数据元素不会被计算。 正是这种惰性求值使能够用无限的 stream。 请注意，没有终端操作的流管道是静默无操作，所以不要忘记包含一个终端操作。



stream API 是很顺畅的：它旨在允许将构成管道的所有调用链接到单个表达式中。 实际上，多个 pipeline 可以链接在一起形成单一的表达式。



默认 stream pipelines 是串行运行的，使 pipeline 并行执行就像在 pipeline 中的任何 stream 上调用**并行方法**一样简单，但很少这样做。



stream API 具有足够的通用性，**实际上任何计算都可以使用流来执行**，但可以并不意味着应该这样做。 如果使用得当，流可以使程序**更短更清晰**; 如果使用不当，可能会使程序**难以阅读和维护**。 关于何时使用 stream 没有硬性规定，但有如下启示：



## stream 应用的例子

考虑以下程序，该程序从字典文件中读取单词并打印其大小符合用户指定的最小值的所有变位词（anagram）组。如果两个单词由长度相通，不同顺序的相同字母组成，则它们是变位词。程序从用户指定的字典文件中读取每个单词并将单词放入map对象中。map对象的键是按照字母排序的单词，因此『staple』的键是『aelpst』，『petals』的键也是『aelpst』：这两个单词就是同位词，所有的同位词共享相同的依字母顺序排列的形式（或称之为alphagram）。map对象的值是包含共享字母顺序形式的所有单词的列表。 处理完字典文件后，每个列表都是一个完整的同位词组。然后程序遍历map对象的`values()`的视图并打印每个大小符合阈值的列表：



```java
// Prints all large anagram groups in a dictionary iteratively
public class Anagrams {
    public static void main(String[] args) throws IOException {
        File dictionary = new File(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
        Map<String, Set<String>> groups = new HashMap<>();
        try (Scanner s = new Scanner(dictionary)) {
            while (s.hasNext()) {
                String word = s.next();
                // computeIfAbsent 返回 value (若不存在，通过lambda表达式生成并返回value)
                groups.computeIfAbsent(alphabetize(word),
                                      (unused) -> new TreeSet<>()) // 这是一个 Set
                    				  .add(word);
            }
        }
        for (Set<String> group : groups.values())
        if (group.size() >= minGroupSize)
            System.out.println(group.size() + ": " + group);
    }
    
    private static String alphabetize(String s) {
        char[] a = s.toCharArray();
        Arrays.sort(a);
        return new String(a);
    }
}
```



这个程序中的一个步骤值得注意。将每个单词插入到 map 中(以粗体显示)中使用了`computeIfAbsent`方法，该方法是在Java 8中添加的。这个方法在map中查找一个键：如果键存在，该方法只返回与其关联的值。如果没有，该方法通过将给定的函数对象应用于键来计算值，将该值与键关联，并返回计算值。`computeIfAbsent`方法简化了将多个值与每个键关联的map的实现。



现在考虑以下程序，它解决了同样的问题，但**大量过度使用了流**。 请注意，整个程序（打开字典文件的代码除外）包含在单个表达式中。 在单独的表达式中打开字典文件的唯一原因是允许使用 `try-with-resources` 语句，该语句确保关闭字典文件：

```java
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.stream.Stream;

import static java.util.stream.Collectors.groupingBy;

// Overuse of streams - don't do this! 
public class StreamAnagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
		// try-with-resources
        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(
                    groupingBy(word -> word.chars().sorted()
                            .collect(StringBuilder::new,
                                    (sb, c) -> sb.append((char) c),
                                    StringBuilder::append).toString()))
                    .values().stream()
                    .filter(group -> group.size() >= minGroupSize)
                    .map(group -> group.size() + ": " + group)
                    .forEach(System.out::println);
        }
    }
}
```



如果你发现这段代码难以阅读，不要担心；你不是一个人。它更短，但是可读性也更差，尤其是对于那些不擅长使用流的程序员来说。过度使用流使程序难于阅读和维护。

幸运的是，有一个折中的办法。下面的程序解决了同样的问题，使用流而不过度使用它们。其结果是一个比原来更短更清晰的程序：



```java
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.Arrays;
import java.util.stream.Stream;

import static java.util.stream.Collectors.groupingBy;

// Tasteful use of streams enhances clarity and conciseness 
public class HybridAnagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);

        try (Stream<String> words = Files.lines(dictionary)) {
            // words 流没有中间操作，直接终结操作 collect
            words.collect(groupingBy(word -> alphabetize(word)))
                    .values().stream() // 打开新的流 List<String>
                    .filter(group -> group.size() >= minGroupSize) // 过滤操作
                    .forEach(g -> System.out.println(g.size() + ": " + g)); // 终结操作
        }
    }

    private static String alphabetize(String s) {
        char[] a = s.toCharArray();
        Arrays.sort(a);
        return new String(a);
    }
}

```

即使以前很少接触流，这个程序也不难理解。它在一个`try-with-resources`块中打开字典文件，获得一个由文件中的所有行组成的流。流变量命名为`words`，表示流中的每个元素都是一个单词。此流上的管道没有中间操作；它的终结操作将所有单词收集到个map对象中，按照字母排列的形式对单词进行分组。这与之前两个版本的程序构造的map完全相同。然后在map的`values()`视图上打开一个新的流`List<String>`。当然，这个流中的元素是同位词组。对流进行过滤，以便忽略大小小于`minGroupSize`的所有组，最后由终结操作`forEach`打印剩下的同位词组。

请注意，仔细选择lambda参数名称。 上面程序中参数`g`应该真正命名为`group`，但是生成的代码行对于本书来说太宽了。

> **在没有显式类型的情况下，仔细命名lambda参数对于流管道的可读性至关重要**。

> 另请注意，单词字母化是在单独的`alphabetize`方法中完成的。 这通过提供操作名称并将实现细节保留在主程序之外来增强可读性。 **使用辅助方法对于流管道中的可读性比在迭代代码中更为重要**，因为管道缺少显式类型信息和命名临时变量。

字母顺序方法可以使用流重新实现，但基于流的字母顺序方法本来不太清楚，更难以正确编写，并且可能更慢。 这些缺陷是由于Java缺乏对原始字符流的支持（这并不意味着Java应该支持char流；这样做是不可行的）。 要演示使用流处理char值的危害，请考虑以下代码：

```java
"Hello world!".chars().forEach(System.out::print);
```

你可能希望它打印`Hello world!`，但如果运行它，发现它打印`721011081081113211911111410810033`。这是因为`“Hello world!”.chars()`**返回的流的元素不是char值，而是int值**，因此调用了`print`的int重载。无可否认，一个名为chars的方法返回一个int值流是令人困惑的。可以通过强制调用正确的重载来修复该程序:



```java
"Hello world!".chars().forEach(x -> System.out.print((char) x));
```

但理想情况下，**应该避免使用流来处理char值**。

当开始使用流时，你可能会感到想要将所有循环语句转换为流方式的冲动，但请抵制这种冲动。尽管这是可能的，但可能会损害代码库的可读性和可维护性。 通常，使用流和迭代的某种组合可以最好地完成中等复杂的任务，如上面的`Anagrams`程序所示。 因此，**重构现有代码以使用流，并仅在有意义的情况下在新代码中使用它们**。



## stream 的限制

如本项目中的程序所示，流管道使用函数对象(通常为lambdas或方法引用)表示重复计算，而迭代代码使用代码块表示重复计算。从代码块中可以做一些从函数对象中不能做的事情:

- 从代码块中，可以读取或修改范围内的任何局部变量; 从lambda中，只能读取最终或有效的最终变量，并且无法修改任何局部变量。
- 从代码块中，可以从封闭方法返回，中断或继续封闭循环，或抛出声明此方法的任何已检查异常; 从一个lambda你不能做这些事情。



## stream 容易做的事

如果使用这些技术最好地表达计算，那么它可能不是流的良好匹配。 相反，流可以很容易地做一些事情：

- 统一**转换**元素序列

- **过滤**元素序列
- 使用**单个操作**组合元素序列（例如添加、连接或计算最小值）

- 将元素序列**累积到一个集合中**，可能通过一些公共属性将它们分组

- 在元素序列中**搜索满足某些条件的元素**

如果使用这些技术最好地表达计算，那么使用流是这些场景很好的候选者。



## 同时访问多阶段的元素

对于流来说，很难做到的一件事是同时访问管道的多个阶段中的相应元素：**一旦将值映射到其他值，原始值就会丢失。**一种解决方案是将每个值映射到一个包含原始值和新值的 `pair` 对象，但这不是一个令人满意的解决方案，尤其是在管道的多个阶段需要一对对象时更是如此。生成的代码既混乱又冗长，破坏了流的主要用途。当它适用时，**一个更好的解决方案是在需要访问早期阶段值时「转换映射」。**

例如，让我们编写一个程序来打印前20个梅森素数(Mersenne primes)。 梅森素数是一个`2p-1`形式的数字。如果`p`是素数，相应的梅森数可能是素数; 如果是这样的话，那就是梅森素数。 作为我们管道中的初始流，我们需要所有素数。 这里有一个返回该（无限）流的方法。 我们假设使用静态导入来轻松访问`BigInteger`的静态成员：



```java
static Stream<BigInteger> primes() {
    return Stream.iterate(TWO, BigInteger::nextProbablePrime);
}
```



方法的名称（primes）是一个复数名词，描述了流的元素。 强烈建议所有返回流的方法使用此命名约定，因为它增强了流管道的可读性。 该方法使用静态工厂`Stream.iterate`，它接受两个参数：流中的第一个元素，以及从前一个元素生成流中的下一个元素的函数。 这是打印前20个梅森素数的程序：



```java
public static void main(String[] args) {
    primes().map(p -> TWO.pow(p.intValueExact()).subtract(ONE))
        .filter(mersenne -> mersenne.isProbablePrime(50))
        .limit(20)
        .forEach(System.out::println);
}
```



这个程序是上面的梅森描述的直接编码：它从素数开始，计算相应的梅森数，过滤掉除素数之外的所有数字（幻数50控制概率素性测试the magic number 50 controls the probabilistic primality test），将得到的流限制为20个元素， 并打印出来。

**现在假设我们想在每个梅森素数前面加上它的指数(p)，这个值只出现在初始流中，因此在终结操作中不可访问，而终结操作将输出结果。幸运的是通过反转第一个中间操作中发生的映射，可以很容易地计算出Mersenne数的指数。 指数是二进制表示中的位数，因此该终结操作会生成所需的结果：**

```java
.forEach(mp -> System.out.println(mp.bitLength() + ": " + mp));
```



## 用流还是迭代看情况

有很多任务不清楚是使用流还是迭代。例如，考虑初始化一副新牌的任务。假设`Card`是一个不可变的值类，它封装了`Rank`和`Suit`，它们都是枚举类型。这个任务代表任何需要计算可以从两个集合中选择的所有元素对。数学家们称它为两个集合的笛卡尔积。下面是一个迭代实现，它有一个嵌套的for-each循环，你应该非常熟悉:

```java
// Iterative Cartesian product computation
private static List<Card> newDeck() {
    List<Card> result = new ArrayList<>();
    for (Suit suit : Suit.values())
        for (Rank rank : Rank.values())
            result.add(new Card(suit, rank));
    return result;
}
```



下面是一个基于流的实现，它使用了中间操作flatMap方法。这个操作将一个流中的每个元素映射到一个流，然后将所有这些新流连接到一个流(或展平它们)。注意，这个实现包含一个嵌套的lambda表达式`rank -> new Card(suit, rank))`:



```java
   // Stream-based Cartesian product computation
   private static List<Card> newDeck() {
       return Stream.of(Suit.values())
           	   // 扁平放到流里
               .flatMap(suit ->
                       Stream.of(Rank.values()).map(rank -> new Card(suit, rank)))
               .collect(toList());
   }
```



`newDeck`的两个版本中哪一个更好？ 它归结为个人偏好和你的编程的环境。 第一个版本更简单，也许感觉更自然。 大部分Java程序员将能够理解和维护它，但是一些程序员会对第二个（基于流的）版本感觉更舒服。 如果对流和函数式编程有相当的精通，那么它会更简洁，也不会太难理解。 如果不确定自己喜欢哪个版本，则迭代版本可能是更安全的选择。 如果你更喜欢流的版本，并且相信其他使用该代码的程序员会与你共享你的偏好，那么应该使用它。

总之，有些任务最好使用流来完成，有些任务最好使用迭代来完成。将这两种方法结合起来，可以最好地完成许多任务。对于选择使用哪种方法进行任务，没有硬性规定，但是有一些有用的启发式方法。在许多情况下，使用哪种方法将是清楚的；在某些情况下，则不会很清楚。**如果不确定一个任务是通过流还是迭代更好地完成，那么尝试这两种方法，看看哪一种效果更好。**

d