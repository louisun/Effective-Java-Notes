# Item44 为所有导出的API元素写文档注释



Javadoc 提供特殊格式的文档注释，根据源代码自动产生 API 文档。



为了正确地编写API文档，必须在每个被导出的「**类、接口、构造器、方法和域声明**」之前增个文档注释。如果类是可序列化的，也应该对它的序列化形式编写文档。如果没有文档注释， Javadoc 所能够做的也就是重新生成该声明，作为受影响的API元素的唯文档。使用没有文档注释的API是非常痛苦的，也很容易出错。为了编写出可维护的代码，还应该为那些没有被导出的类、接口、构造器、方法和域编写文档注释。



文档注释应该简介描述与使用者的「约定」，方法做了什么，前提条件、后置条件。



文档注释还要说明副作用（系统状态中可观察到的变化，比如方法启动了后台线程，文档中要说明这一点）。



文档注释要描述类或方法的「**线程安全性**」



方法注释每个参数都要有  `@param` 标签，以及一个 `@return` 标签，对抛出的每个异常要有 `@throws` 标签。`@param` 和 `@return` 后面的文字应该是一个「**名词短语**」，描述这个参数或返回值「**所表示的值**」，`@throws`标签后应该包含「`if`」，紧接着是一个名词短语，表明什么情况下异常会抛出。



![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-15-14-55-46_r19.png)









文档注释中可以使用 HTML 标签。



上面 `@trhow` 中有个 `@code`，作用是：

1. 以代码字体 code font 呈现
2. 限制 HTML 标记和嵌套的 JavaDoc 标签在代码段中进行处理（转义）





## 简要概述

文档注释中的第一句话是：所属元素的「**概要描述**」，为避免混淆，同一个类或接口的两个成员或构造器，不应该有同样的概要描述，特别是重载的情形。



对方法、构造器，概要描述是完整的动词短语（包含任何对象），描述了方法执行的动作。

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-15-15-01-21_r97.png)



对于类、接口、域，概要描述应该是一个「名词短语」，描述类或接口的实例，或域本身所代表的事物。

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-15-15-01-31_r31.png)



## 泛型、枚举、注解的文档注释



对泛型、枚举、注解的注释要特别小心。泛型要说名所有的类型参数：



![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-15-15-02-14_r44.png)



枚举类型的文档注释，要注释说明常量，以及类型，还有公有的方法。如果文档注释很短，可以在一行上：

```java
/**
 * An instrument section of a symphony orchestra.
 */
public enum OrchestraSection {
    /** Woodwinds, such as flute, clarinet, and oboe. */
    WOODWIND,

    /** Brass instruments, such as french horn and trumpet. */
    BRASS,

    /** Percussion instruments, such as timpani and cymbals. */
    PERCUSSION,

    /** Stringed instruments, such as violin and cello. */
    STRING;
}

```





为注解类型编写文档注释，要说明所有成员，以及类型本身。



```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTest {
    /**
     * The exception that the annotated test method must throw
     * in order to pass. (The test is permitted to throw any
     * subtype of the type described by this class object.)
     */
    Class<? extends Throwable> value();
}
```

