# Item53. 接口优先于反射机制



**核心反射机制（core reflection facility）** `java.lang.reflect` 提供了『通过程序来访问关于已装载类的信息』的能力。给定一个 `Class` 实例，可以获得 `Constructor`、`Method`、`Field`实例，分别代表了构造器、方法和域。这些对象提供了『通过程序来访问类成员名称、域类型、方法签名等信息』的能力。



而且， `Constructor`、 `Method` 和 `Field` 实例使你能够通过反射机制操作它们的**底层对等体**通过调用 `Constructor`、 `Method`和 `Field` 实例上的方法，可以**构造底层类的实例、调用底层类的方法，并访问底层类中的域。**例如， `Method.invoke` 使你可以调用任何类的任何对象上的任何方法（遵从常规的安全限制）。反射机制（ reflection）**允许一个类使用另一个类**，即使当前者被编译的时候后者还根本不存在。然而，这种能力也要付出代价：



- **丧失了编译时类型检查的好处**：包括异常检查。如果程序企图用反射的方式调用不存在或不可访问的方法，在运行时它将会失败，除非采取了特别的预防措施
- **执行反射访问所需要的代码非常笨拙和冗长**
- **性能损失**：反射方法调用比普通方法慢了许多，具体受多个因素影响，速度差异可能是2倍到50倍...



> 核心反射机制最初是为了「**基于组件的应用创建工具而设计的**」，这类工具要根据需要装载类，并且用反射功能找出它们支持哪些方法和构造器。这些工具允许用户交互式地构建出访问这些类的程序，但是所生产出来的这些应用程序能够以正常的方式访问这些类，而不是以反射的方式。反射功能只是在设计时候被用到，通**常普通应用程序在运行时不应该以反射方式访问对象。**



有一些复杂的应用程序需要用反射机制。示例中包括类浏览器、对象监视器、代码分析工具、解释型的内嵌式系统。**在 RPC （远程过程调用）系统中使用反射机制也是非常合适的，**这样可以不需要存「根编译器」，如果你对自己的应用程序是否也属于这一类应用程序而感到疑惑，它很可能不属于这一类。

如果只是「以非常有限的形式使用反射机制」，虽然也要付出少许代价，但是可以获得许多好处。**对于有些程序，它们必须用到在编译时无法获取的类，但是在编译时存在适当的接口或者超类，通过它们可以引用这个类**（见 Item 52）。如果是这种情况，就可以**以反射方式创建实例然后通过它们的接口或者超类，以正常的方式访问这些实例。**

举个例子，下面程序创建一个 `Set<String>` 实例，它的类是由第一个命令行参数指定的。改程序把其余的命令行参数插入到这个集合中，然后打印这个集合。不管第一个参数是什么，程序都会打印出余下的命令行参数，其中重复的参数会被消除掉。这些参数的打印顺序取决于第一个参数中指定的类，如果指定的是 `java.util.HashSet`，那么打印顺序是随机的，如果指定 `TreeSet`，就会按字母顺序打印出来。



```java
package effectivejava.chapter9.item65;

import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
import java.util.Arrays;
import java.util.Set;

// Reflective instantiaion demo 
public class ReflectiveInstantiation {
    // Reflective instantiation with interface access
    public static void main(String[] args) {
        // Translate the class name into a Class object
        Class<? extends Set<String>> cl = null;
        try {
            cl = (Class<? extends Set<String>>)  // Unchecked cast!
                    Class.forName(args[0]);
        } catch (ClassNotFoundException e) {
            fatalError("Class not found.");
        }

        // Get the constructor
        Constructor<? extends Set<String>> cons = null;
        try {
            cons = cl.getDeclaredConstructor();
        } catch (NoSuchMethodException e) {
            fatalError("No parameterless constructor");
        }

        // Instantiate the set
        Set<String> s = null;
        try {
            s = cons.newInstance();
        } catch (IllegalAccessException e) {
            fatalError("Constructor not accessible");
        } catch (InstantiationException e) {
            fatalError("Class not instantiable.");
        } catch (InvocationTargetException e) {
            fatalError("Constructor threw " + e.getCause());
        } catch (ClassCastException e) {
            fatalError("Class doesn't implement Set");
        }

        // Exercise the set
        s.addAll(Arrays.asList(args).subList(1, args.length));
        System.out.println(s);
    }

    private static void fatalError(String msg) {
        System.err.println(msg);
        System.exit(1);
    }
}
```



尽管这个程序就像一个“玩具”，但是它所演示的这种方法是非常强大的。这个玩具程序可以很容易地变成一个「**通用的集合测试器**」，通过侵入式地操作一个或者多个集合实例，并**检查是否遵守 `Set` 接口的约定，以此来验证指定的 `Set` 实现**。同样地，它也可以变成一个通用的集合性能分析工具。实际上，它所演示的这种方法足以实现一个成熟的服务提供者框架（ serviceprovider framework）。绝大多数情况下，使用反射机制时需要的也正是这种方法。

这个例子显示了反射的两个缺点。第一，这个例子在运行时生成了 6 种不同的异常。如果没用反射实例化，所有这些都会是编译时错误。第二，代码冗长，通过类名生成了一个实例，但其实一个构造器调用就能完成。然而，这些缺点仅仅局限于实例化对象的那部分代码。一旦对象被实例化，它与其他的 Set 实例就难以区分。在实际的程序中，通过这种限定使用反射的方法，大部分代码可以不受影响。



如果编译这个程序，会得到一个未受检 unchecked 转换警告。这个警告是合法的，因为即使不是一个 `Set` 实现， 转换为 `Class<? extends Set<String>>` 也会成功，这种情况下会抛出 `ClassCastException`。可以通过 Item 24 的方式抑制警告。



另一个值得注意的附带问题是，程序使用了 `System.exit`，很少会有需要调用这个方法，它会终止整个虚拟机，但是，它对于「命令行有效性的非法终止」是非常合适的。



**类对于在运行时可能不存在的其他类、方法或者域的依赖性，用反射法进行管理，这种用法是合理的，但是很少使用。**如果*要编写一个包，并且它运行的时候必须依赖其他某个包的多个版本*，这种做法可能就非常有用。这种做法就是，在支持包所需要的最小环境下对它进行编译，通常是最老的版本，然后以反射方式访问任何更加新的类或者方法。如果企图访问的新类或者新方法在运行时不存在，为了使这种方法有效你还必须采取适当的动作。所谓适当的动作，可能包括使用某种其他可替换的办法来达到同样的目的，或者使用简化的功能进行处理。



简而言之，反射机制是一种功能强大的机制，对于特定的复杂系统编程任务，它是非常必要的，但它也有一些缺点。**如果你编写的程序必须要与编译时未知的类一起工作，如有可能，就应该仅仅使用反射机制来实例化对象，而访问对象时则使用编译时已知的某个接口或者超类**。

