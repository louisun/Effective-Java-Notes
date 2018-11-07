# Item1. 静态工厂方法代替构造器

```java
public static Boolean valueOf(boolean b){
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```

## 优点

### 1. 静态工厂不像构造器，静态工厂有名称

比如：构造器 `BigInteger(int, int, Random)` 返回的 `BigInteger` 可能是素数，如果用名为 `BigInteger.probablePrime` 的静态工厂方法来表示显然更为清楚

###  2. 静态工厂不需要每次调用的时候都创建一个新对象

允许不可变类 [(Item 17)]() 使用预先构造好的实例，或者当实例构造好的时候缓存下来，避免重复构造不必要的对象。

比如上面的 `public static Boolean valueOf(boolean b)` 返回了构造好的实例。

此能力允许类严格控制实例的产生和销毁，就是所谓的 *instance-controlled*。

写这种 *instance-controlled* 类的原因：

- 单例（singleton）模式[(Item 3)]() 
- 不可实例化模式 [(Item 4)]()
- 允许不可变值的类 [(Item 17)]() ：不允许两个相等的实例 `a.equals(b)  // only of a == b`,  `Enums 类型`[(Item34)]() 提供了这个保证。

### 3. 静态工厂可以返回原类型任何子类对象

能够灵活地选择返回对象的类型。

一种应用是一个 API 可以不使类 `public` 而返回这个对象。这种隐藏实现类的方法可以得到紧凑的 API。

*interfaced-based* （基于接口的）框架 [(Item 20)]() 常常使用这个技术，接口为静态工厂方法提供了自然的返回类型。

Java 8 之前，接口不能有静态方法，一个接口的静态工厂方法（叫做 `Type`）放在一个叫做 `Types`的不可实例化的 *companion class* 中 [(Item 4)]()。举个例子：Java Collection 框架有其接口的 45 种工具实现，提供了不可修改的集合，可同步的集合等。几乎所有的这些实现 **通过静态工厂方法** 暴露在不可实例化的 `java.util.Collections` 类中。程序员能通过接口 API 准确地知道返回的对象是什么，而不需要读文档。

使用静态工厂方法，需要通过接口而不是事先类来指明返回对象，是一个 good practice [(Item 64)]()。

Java 8 之后，接口也可以有静态方法了，所以没有必要为接口提供一个不可实例化的 *companion class* 了。 很多共有的静态成员可以放在接口自己那里，而不用放到另一个类的里面的了。但是将在静态方法后面的大量的实现代码放在独立的 package-private 类任是有必要的。Java 9 允许私有的静态方法，但是 static fields 和 static member class 依然要 public。



### 4. 返回对象的类可以根据参数的变化而变化



任何声明的返回类型的子类型都是宽容的。

`EnumSet` 类 [(Item 36)]() 没有公有构造器，只有静态工厂，在 OpenJDK 实现中，如果 enum type 大小是 64 以下，静态工厂返回 `RegularEnumSet` 实例， 如果是 65 以上，返回 `JumboEnumSet`，背后是 `long` 数组实现。

### 5. 当写类内方法的时候，返回对象的类型可以不存在

像 Java JDBC （Java Database Connectivity API）。

静态工厂是 service provider framework 的基础，将客户端和实现解耦。

3 个关键的组件：

- `servcie interface`：服务接口（即实现）：`Connection`
-  **provider registration API**：提供商注册 API（提供商用来注册实现）：`DriverManager.registerDriver`
-  **service access API**：静态工厂，服务接入 API（客户端使用其获取 service 实例）：`DriverManager.getConnection`
- (Optional): **service provider interface**：描述一个产生 `service interface `实例的工厂对象：`Driver` 

“服务接入 API” 允许客户端明确标准来选择哪一种实现，如果是空的标准，API 会返回的默认实现的实例，或者允许客户端轮询所有可行的实现。**这个 Service Access API （服务接入API）**就是灵活的静态工厂，也是服务提供商框架的基础。

可选的第四个组件是 *service provider interface*，**描述一个产生服务接口 *service interface* 实例的工厂对象**。如果缺少一个 *service provider interface*，实现必须用反射来实例化 [(Item 65)]()。

在 JDBC 中，`Connection` 就扮演了服务接口（**servcie interface**）的作用，`DriverManager.registerDriver`是一个 **provider registration API**，`DriverManager.getConnection` 是一个 **service access API**，`Driver` 是 **service provider interface**。

有很多服务提供商框架，比如 Bridge pattern。依赖注入框架 [(Item 5)]() 可以被视为强大的 service providers。Java 6 平台包括了通用目的的 `java.util.ServiceLoader`，一般不需要也不应该自己去写 service provider framework。JDBC 用的不是 `ServiceLoader`，因为比它更早。


```java
// Service provider framwork sketch

// Service Inerface
public interface Service {
    ... // Service-spcific methods go here
}

// Service provider interface
public interface Provider{
    // 产生一个 Service 实例
    Service newService();
}

// Noninstantiable class for service registration and access
public class Services{
    private Services(){}

    // Maps service names to service
    private static final Map<String, Provider> providers = new ConcurrentHashMap<String, Provider>();

    public static void registerDefaultProvider(Provider p){
        registerDefaultProvider(DEFAULT_PROVIDER_NAME, p);
    }

    public static registerDefaultProvider(String name, Provider p){
        provider.put(name, p);
    }

    // Service access API
    public static Service newInstance(){
        return newInstance(DEFAULT_PROVIDER_NAME)
    }

    public static Service newInstance(String name){
        Provider p = providers.get(name);
        if(p == null){
            throw new IllegalArgumentException(
                "No provider regietered with name: " + name);
            )
        }
        return p.newService();
    }
}
```


## 缺点



###  1. 只有一个静态工厂方法，没有公共或保护的构造函数的类，无法被子类化



比如无法子类化 Collections Framwork 中的任何方便的实现类。这其实鼓励我们用组合的方法而不用继承[(Item 18)]()，而且必需要是不可变的类型[(Item 17)]()。



```java
//这是一个父类Demo
public class SunDemo {
	//私有的构造器
    private SunDemo(){
    }
    //通过静态工厂方法得到对象
    public static SunDemo value(){
        return new SunDemo();
    }
    //父类的方法
    public void say(){
        System.out.println("SunDemo say()");
    }
}

//子类Demo

public class Demo extends SunDemo{
}

// error：Implicit super constructor SunDemo() is not visible for default constructor. Must define an explicit constructor

// 说明SunDemo没有公共的构造器，它不允许被继承。就是说你的类使用了静态工厂方法提供对象的实例化，没有提供public的构造器，那么这个类就不允许被继承。我们如果想要在Demo类中使用SunDemo的方法就得使用复合。

public class Demo {

    SunDemo d = null;

    @Test
    public void DemoTest(){

        d=SunDemo.value();
        d.say();

    }
}
```



###  2. 静态工厂方法很难被找到



在文档中不像构造方法那么直观，很难搞清楚如何用静态工厂而非构造器，来实例化一个类。 Javadoc 工具有时候可以提醒你。通用静态工厂命名规范：



![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2018-11-06-23-22-53_r8.png)











