# Item35. 注解优先于命名模式

> 命名模式：即约定「方法」、「变量」、「类名」要如何命名

Java 1.5 之前，一般用 **命名模式（naming pattern）**表明：有些程序元素需要通过某种工具或框架进行特殊的处理。利用，用 JUnit 测试框架原本要求用户一定要用「test」作为 测试方法的名称开头。



第 1个缺点：文字拼写可能出错导致失败，且没有任何提示。



第 2 个缺点：无法确保它们只作用于相应的程序元素上，如某个类称作「`testSafetyMechanisms`」，是希望 JUnit 自动测试它的所有方法，而不管它们叫什么名称，JUnit 还是不会出错，但也同样不会执行测试。



第 3 个缺点：没有提供将**参数值**与**程序元素**关联起来的好方法。例如，假设想要支持一种测试类别，它只在抛出特殊异常时才会成功。异常类型本质上是测试的一个参数。你可以利用某种具体的命名模式，**将异常类型名称编码到测试方法名称中，但是这样的代码会很不雅观，也很脆弱**。编译器不知道要去检验准备命名异常的字符串是否真正命名成功。如果命名的类不存在，或者不是一个异常，你也要到试着运行测试时才会发现。



## 注解 Annotation 解决了所有上面的问题



JUnit 从第 4 版开始采用注解。



```java
// Test.java 文件
// Marker annotation type declaration
import java.lang.annotation.*;

/**
 * Indicates that the annotated method is a test method.
 * Use only on parameterless static methods.
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Test {
}
```



`Test` 注解类型的声明本身使用`Retention`和`Target`注解进行标记。 **注解类型声明上的这种注解称为元注解。** 



`@Retention(RetentionPolicy.RUNTIME)`元注解指示`Test`注解应该在**运行时保留**。 没有它，测试工具就不会看到`Test`注解。



`@Target.get(ElementType.METHOD)`元注解表明 `Test` 注解只对方法声明合法：它不能应用于类声明，属性声明或其他程序元素。



在 `@Test` 注解声明之前的注释说：“**仅在无参静态方法中使用** Use only parameterless static method”。如果编译器可以强制执行此操作是最好的，但它不能，除非编写注解处理器来执行此操作。 有关此主题的更多信息，请参阅`javax.annotation.processing`文档。 **在缺少这种注解处理器的情况下**，如果将 `Test` 注解放在实例方法声明或带有一个或多个参数的方法上，那么测试程序仍然会编译，**并将其留给测试工具在运行时来处理这个问题 。**

以下是Test注解在实践中的应用。 它被称为标记注解，因为它没有参数，只是“标记”注解元素。

```java
public class Sample {
    @Test
    public static void m1() { }        // Test should pass
    
    public static void m2() { }
    
    @Test public static void m3() {    // Test should fail
        throw new RuntimeException("Boom");
    }
    public static void m4() { }  // Not a test
    
    @Test public void m5() { }   // INVALID USE: nonstatic method
    
    public static void m6() { }
    
    @Test public static void m7() {    // Test should fail
        throw new RuntimeException("Crash");
    }
    public static void m8() { }
}
```



`Sample`类有七个静态方法，其中四个被标注为 `Test` 。 其中两个，m3 和 m7 引发异常，两个m1和m5不引发异常。 但是没有引发异常的注解方法之一是实例方法，因此它不是注释的有效用法。 总之，`Sample`包含四个测试：一个会通过，两个会失败，一个是无效的。 未使用 `Test` 注解标注的四种方法将被测试工具忽略。



## 注解是如何对测试工具生效的（反射）

Test 注解对Sample类的语义没有直接影响。 他们只提供信息供相关程序使用。 更一般地说，注**解不会改变注解代码的语义，但可以通过诸如这个简单的测试运行器等工具对其进行特殊处理：**

```java
import java.lang.reflect.*;

public class RunTests {
    // 扫描 Test.java 类中 @Test 注解的方法，反射调用
    public static void main(String[] args) throws Exception {
        int tests = 0;
        int passed = 0;
        Class<?> testClass = Class.forName(args[0]);
        for (Method m : testClass.getDeclaredMethods()) {
            if (m.isAnnotationPresent(Test.class)) {
                tests++;
                try {
                    m.invoke(null);
                    passed++;
                } catch (InvocationTargetException wrappedExc) {
                    Throwable exc = wrappedExc.getCause();
                    System.out.println(m + " failed: " + exc);
                } catch (Exception exc) {
                    System.out.println("Invalid @Test: " + m);
                }
            }
        }
        System.out.printf("Passed: %d, Failed: %d%n",
                passed, tests - passed);
    }
}
```





测试运行器工具在命令行上接受完全限定的类名，并通过调用`Method.invoke`来反射地运行所有类标记有Test注解的方法。 `isAnnotationPresent`方法告诉工具要运行哪些方法。 如果测试方法引发异常，则反射机制将其封装在`InvocationTargetException`中。 该工具捕获此异常并打印包含由test方法抛出的原始异常的故障报告，该方法是使用`getCause`方法从`InvocationTargetException`中提取的。



**如果尝试通过反射调用测试方法会抛出除`InvocationTargetException`之外的任何异常，则表示编译时未捕获到没有使用的 Test 注解。** 这些用法包括注解实例方法，具有一个或多个参数的方法或不可访问的方法。 测试运行器中的第二个catch块会捕获这些Test使用错误并显示相应的错误消息。 这是在`RunTests`在`Sample`上运行时打印的输出：



```java
public static void Sample.m3() failed: RuntimeException: Boom
Invalid @Test: public void Sample.m5()
public static void Sample.m7() failed: RuntimeException: Crash
Passed: 1, Failed: 3
```





## 「抛出特定异常时才成功测试」的注解

现在，让我们添加对仅在抛出特定异常时才成功的测试的支持。 我们需要为此添加一个新的注解类型：



```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTest {
    Class<? extends Throwable> value();
}

```

此注解的参数类型是`Class<? extends Throwable>`。毫无疑问，这种通配符是拗口的。 在英文中，它表示“扩展Throwable的某个类的Class对象”，它允许注解的用户指定任何异常（或错误）类型。 这个用法是一个限定类型标记的例子。 以下是注解在实践中的例子。 请注意，类名字被用作注解参数值：



```java
public class Sample2 {
    @ExceptionTest(ArithmeticException.class)
    public static void m1() {  // Test should pass
        int i = 0;
        i = i / i;
    }
    @ExceptionTest(ArithmeticException.class)
    public static void m2() {  // Should fail (wrong exception)
        int[] a = new int[0];
        int i = a[1];
    }
    @ExceptionTest(ArithmeticException.class)
    public static void m3() { }  // Should fail (no exception)
}

```

现在让我们修改测试运行器工具来处理新的注解。 这样将包括将以下代码添加到 main 方法中：



```java

Class<?> testClass = Class.forName(args[0]);
for (Method m : testClass.getDeclaredMethods()) {

    if (m.isAnnotationPresent(ExceptionTest.class)) {
        tests++;
        try {
            m.invoke(null);
            // 必须要是某个异常才通过，运行正常会反而失败
            System.out.printf("Test %s failed: no exception%n", m);
        } catch (InvocationTargetException wrappedEx) {
            Throwable exc = wrappedEx.getCause();	// 真正的异常
            // 获取特定注解中的 value
            Class<? extends Throwable> excType =
                m.getAnnotation(ExceptionTest.class).value();
            if (excType.isInstance(exc)) {
                passed++;	// 通过
            } else {
                System.out.printf(
                    "Test %s failed: expected %s, got %s%n",
                    m, excType.getName(), exc);
            }
        } catch (Exception exc) {
            System.out.println("Invalid @ExceptionTest: " + m);
        }
    }
}

System.out.printf("Passed: %d, Failed: %d%n",
                  passed, tests - passed);

```



此代码与我们用于处理Test注解的代码类似，只有一个例外：此代码提取注解参数的值并使用它来检查测试引发的异常是否属于正确的类型。 没有明确的转换，因此没有`ClassCastException`的危险。 测试程序编译的事实保证其注解参数代表有效的异常类型，但有一点需要注意：如果注解参数在编译时有效，但代表指定异常类型的类文件在运行时不再存在，则测试运行器将抛出`TypeNotPresentException`异常。



## 抛出异常数组用于测试的注解

将我们的异常测试示例进一步推进，可以设想一个测试，如果它抛出几个指定的异常中的任何一个，就会通过测试。 注解机制有一个便于支持这种用法的工具。 假设我们将`ExceptionTest`注解的参数类型更改为Class对象数组：



```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTest {
    Class<? extends Exception>[] value();
}
```



注解中数组参数的语法很灵活。 它针对单元素数组进行了优化。 所有以前的`ExceptionTest`注解仍然适用于`ExceptionTest`的新数组参数版本，并且会生成单元素数组。 **要指定一个多元素数组，请使用花括号将这些元素括起来，并用逗号分隔它们：**



```java

// Code containing an annotation with an array parameter 
@ExceptionTest({ IndexOutOfBoundsException.class,
                NullPointerException.class })
public static void doublyBad() {   // Should pass
    List<String> list = new ArrayList<>();

    // The spec permits this method to throw either
    // IndexOutOfBoundsException or NullPointerException
    list.addAll(5, null);
}
```





```java

Class<?> testClass = Class.forName(args[0]);
for (Method m : testClass.getDeclaredMethods()) {

    // Code to process annotations with array parameter (Page 185)
    if (m.isAnnotationPresent(ExceptionTest.class)) {
        tests++;
        try {
            m.invoke(null);
            System.out.printf("Test %s failed: no exception%n", m);
        } catch (Throwable wrappedExc) {
            Throwable exc = wrappedExc.getCause();
            int oldPassed = passed;
            Class<? extends Throwable>[] excTypes =
                m.getAnnotation(ExceptionTest.class).value();
            // 只要满足一种异常类型即可
            for (Class<? extends Throwable> excType : excTypes) {
                if (excType.isInstance(exc)) {
                    passed++;
                    break;
                }
            }
            if (passed == oldPassed)
                System.out.printf("Test %s failed: %s %n", m, exc);
        }
    }
}
System.out.printf("Passed: %d, Failed: %d%n",
                  passed, tests - passed);
```



## @Repeatable 重复使用注解

从 **Java 8** 开始，还有另一种方法来执行多值注解。 **可以使用`@Repeatable`元注解来标示注解的声明，而不用使用数组参数声明注解类型，以指示注解可以重复应用于单个元素。** 



该元注解采用单个参数，**该参数是包含注解类型的类对象，其唯一参数是注解类型的数组**。 如果我们使用`ExceptionTest`注解采用这种方法，下面是注解的声明。 请注意，包含注解类型必须使用适当的保留策略和目标进行注解，否则声明将无法编译：



```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Repeatable(ExceptionTestContainer.class)	// 指示 @ExceptionTestContainer 可以重用该注解 
public @interface ExceptionTest {
    Class<? extends Throwable> value();
}

// ExceptionTestContainer.java
// Container annotation for the 复用 ExceptionTest 注解
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTestContainer {
    // 该注解唯一参数类型是「注解类型（ExceptionTets）的数组」
    ExceptionTest[] value();
}
```



下面是我们的 `doublyBad` 测试**用一个重复的注解**代替基于数组值注解的方式:



```java
// Code containing a repeated annotation (Page 186)
@ExceptionTest(IndexOutOfBoundsException.class)
@ExceptionTest(NullPointerException.class)
public static void doublyBad() {
    List<String> list = new ArrayList<>();

    // The spec permits this staticfactory to throw either
    // IndexOutOfBoundsException or NullPointerException
    list.addAll(5, null);
}
```







```java
for (Method m : testClass.getDeclaredMethods()) {
    // Processing repeatable annotations
    // 检查重复和非重复的注解，要检查「注解类型」以及「包含的注解类型」
    if (m.isAnnotationPresent(ExceptionTest.class)
        || m.isAnnotationPresent(ExceptionTestContainer.class)) {
        tests++;
        try {
            m.invoke(null);
            System.out.printf("Test %s failed: no exception%n", m);
        } catch (Throwable wrappedExc) {
            Throwable exc = wrappedExc.getCause();
            int oldPassed = passed;
            // getAnnotationsByType 可以访问重复注解和非重复注解
            ExceptionTest[] excTests =
                m.getAnnotationsByType(ExceptionTest.class);
            for (ExceptionTest excTest : excTests) {
                if (excTest.value().isInstance(exc)) {
                    passed++;
                    break;
                }
            }
            if (passed == oldPassed)
                System.out.printf("Test %s failed: %s %n", m, exc);
        }
    }
}
System.out.printf("Passed: %d, Failed: %d%n", passed, tests - passed);

```



> 这里非重复的注解应该就是指 「ExceptionTestContainer」?



处理可重复的注解需要注意。**一个 repeated annotation 注解会生成一个合成的包含注解类型的注解。(也就是说，一个可重复的注解`@ExceptionTest`会生成其所指定的`@ExceptionTestContainer`)，**但是我们 `getAnnotationsByType(ExceptionTest.class)` 会掩盖着一点，其可用于访问可重复注解类型和非重复注解。



而`isAnnotationPresent`明确指出**可重复注解`@ExeceptionTest`并不是注解类型，而是一个包含注解类型 `@ExceptionTestContainer`。**如果某个元素具有某种类型的重复注解，并且使用`isAnnotationPresent`方法检查元素是否具有该类型的注释，则会发现它没有。

**使用此方法检查注解类型的存在会因此导致程序默默忽略重复的注解。**同样，使用此方法检查**包含注解类型**将导致程序默默忽略不重复的注释。

**要使用`isAnnotationPresent`检测重复和非重复的注解，需要检查注解类型及其包含的注解类型。**



## 总结

添加了可重复的注解以提高源代码的可读性，从逻辑上将相同注解类型的多个实例应用于给定程序元素。 如果觉得它们增强了源代码的可读性，请使用它们，但请记住，在声明和处理可重复注解时存在更多的样板，并且处理可重复的注解很容易出错。

这个项目中的测试框架只是一个演示，但它清楚地表明了注解相对于命名模式的优越性，而且它仅仅描绘了你可以用它们做什么的外观。 如果编写的工具要求程序员将信息添加到源代码中，请定义适当的注解类型。**当可以使用注解代替时，没有理由使用命名模式**。

这就是说，除了特定的开发者（toolsmith）之外，大多数程序员都不需要定义注解类型。 但**所有程序员都应该使用Java提供的预定义注解类型**。 另外，请考虑使用IDE或静态分析工具提供的注解。 这些注解可以提高这些工具提供的诊断信息的质量。 但请注意，这些注解尚未标准化，因此如果切换工具或标准出现，可能额外需要做一些工作。

