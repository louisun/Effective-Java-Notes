# Item30. 用enum代替int常量

Java 1.5 增加了两个新的引用类型家族：「枚举类型」和「注解类型」。



## int 常量不好



枚举类型（ `enum type`）是指由一组固定的常量组成合法值的类型，例如一年中的季节太阳系中的行星或者一副牌中的花色。



`int`枚举模式：在编程语言中还没有引入枚举类型之前，表示枚举类型的常用模式是声明一组具名的`int`常量，每个类型成员一个常量：



![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2018-12-07-23-10-25_r74.png)



## 枚举常量

用枚举类型可以这样：

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2018-12-07-23-13-11_r18.png)



Java 的枚举类型功能十分强大，Java 的枚举本质是 `int` 值。



枚举类型是通过`public static final` 域为每个「**枚举常量**」导出实例的类。没有可以访问的构造器，所以枚举类型是真正的 final，客户端不能创建枚举类型，也不能对它进行扩展，**而只有声明过的枚举类型**。



**也就是说，枚举类型是实例受控的。它们是单例（Singleton）的泛型化，本质上是单元素的枚举。**



枚举类型提供了「编译安全」，如果声明一个参数的类型为 Apple，就能保证被传到该参数上的非「`null`」引用对象一定属于三个有效的 Apple 值之一，试图传递类型错误的值，或者用`==`比较不同枚举类型的值，会导致编译时错误。



> 枚举类型允许添加任意的方法和域，并实现任意接口（它是一个较为完整类），提供了所有 `Object` 方法的 **高级实现**，实现了 `Comparable` 和 `Serializable`接口，并针对枚举类型的可任意改变性设计了序列化方式。



## 为什么要对枚举类添加方法和域？

首先，可能是想**将数据与它的常量关联**，以增强枚举类型：





枚举类型是写在最上面的，比如 `Planet` 中的 `MERCURY(3.302e+23, 2.439e6)`，可以当做是`Planet MERCURY = new Planet(3.302e+23, 2.439e6){//这里还能覆盖方法}`这样的写法



```java
public enum Planet {
    // 逗号一起实例化，看了上面的解释就明白了
	MERCURY(3.302e+23, 2.439e6), 
    VENUS(4.869e+24, 6.052e6), 
    EARTH(5.975e+24,6.378e6),
    MARS(6.419e+23, 3.393e6), 
    JUPITER(1.899e+27, 7.149e7),
    SATURN(5.685e+26, 6.027e7), 
    URANUS(8.683e+25, 2.556e7),
    NEPTUNE(1.024e+26, 2.477e7);
    
	private final double mass; // 质量：千克
	private final double radius; // 半径
	private final double surfaceGravity; // 重力加速度 a

	// Universal gravitational constant in m^3 / kg s^2
	private static final double G = 6.67300E-11;

	// 构造函数
	Planet(double mass, double radius) {
		this.mass = mass;
		this.radius = radius;
		surfaceGravity = G * mass / (radius * radius);
	}

	public double mass() {
		return mass;
	}

	public double radius() {
		return radius;
	}

	public double surfaceGravity() {
		return surfaceGravity;
	}

	public double surfaceWeight(double mass) {
		return mass * surfaceGravity; // F = ma
	}
}
```





**为了将数据与枚举常量关联，要声明实例域， 并编写一个带有数据并保存在域的构造器。**枚举类型 **天生不可变**，所有域都该是 `final` 的，可以是公有的，但最好还是私有并提供公有方法。 



```java
public class WeightTable {
	public static void main(String[] args) {
		double earthWeight = Double.parseDouble(args[0]);	// 地球上的重量(牛)
		double mass = earthWeight / Planet.EARTH.surfaceGravity();	// 物体质量
		for (Planet p : Planet.values()) // 在各个星球上的重量
            // 注意 %n 在 Java 中 printf 的用法
			System.out.printf("Weight on %s is %f%n", p, p.surfaceWeight(mass));
	}
}
```



```java
Weight on MERCURY is 69.912739
Weight on VENUS is 167.434436
Weight on EARTH is 185.000000
Weight on MARS is 70.226739
Weight on JUPITER is 467.990696
Weight on SATURN is 197.120111
Weight on URANUS is 167.398264
Weight on NEPTUNE is 210.208751
```







一些与枚举常量相关的行为只需要在定义枚举的类或包中使用。 这些行为最好以私有或包级私有方式实现。 然后每个常量携带一个隐藏的行为集合，允许包含枚举的类或包在与常量一起呈现时作出适当的反应。 与其他类一样，除非你有一个令人信服的理由将枚举方法暴露给它的客户端，否则将其声明为私有的，如果需要的话将其声明为包级私有（第13条）



如果一个枚举是广泛使用的，它应该是一个顶级类; 如果它的使用与特定的顶级类绑定，它应该是该顶级类的成员类（第22条）。 例如，`java.math.RoundingMode`枚举表示小数部分的舍入模式。 `BigDecimal`类使用了这些舍入模式，但它们提供了一种有用的抽象，它并不与`BigDecimal`有根本的联系。 **通过将`RoundingMode`设置为顶层枚举，类库设计人员鼓励任何需要舍入模式的程序员重用此枚举，从而提高跨API的一致性**。



## 行为与常量关联



## switch

```java
// Enum type that switches on its own value - questionable
public enum Operation {
    PLUS, MINUS, TIMES, DIVIDE;

    // Do the arithmetic operation represented by this constant
    public double apply(double x, double y) {
        switch(this) {
            case PLUS:   return x + y;
            case MINUS:  return x - y;
            case TIMES:  return x * y;
            case DIVIDE: return x / y;
        }
        throw new AssertionError("Unknown op: " + this);
    }
}
```



此代码可行但不好看，若没有 `throw` 语句，它将不能编译，虽然技术角度看代码的结束部分是可以执行到的，但实际上是不可能执行到这行代码的。更糟糕的是这段代码很脆弱，如果添加了新的枚举常量，却忘记给 `switch` 添加相应条件，枚举任可编译，但运算可能失败。





### 特定于常量的方法实现

还好有一种更好的方法将不同行为与每个枚举常量关联：在枚举类型中声明一个抽象的`apply`方法，并用**常量特定的类主体中**的每个常量的具体方法重写它。 这种方法被称为**特定于常量（constant-specific）的方法实现**：



```java
// Enum type with constant-specific method implementations
public enum Operation {
  // 相当于每个枚举类型都实现了 apply 抽象方法
  PLUS  {public double apply(double x, double y){return x + y;}},
  MINUS {public double apply(double x, double y){return x - y;}},
  TIMES {public double apply(double x, double y){return x * y;}},
  DIVIDE{public double apply(double x, double y){return x / y;}};
	
  // 抽象方法
  public abstract double apply(double x, double y);
}
```

如果向第二个版本的操作添加新的常量，则不太可能会忘记提供`apply`方法，因为该方法紧跟在每个常量声明之后。 万一忘记了，编译器会提醒你，因为枚举类型中的抽象方法必须被所有常量中的具体方法重写。



## 特定于常量的「方法」结合特定于常量的「数据」



```java
// Enum type with constant-specific class bodies and data
public enum Operation {
    PLUS("+") {
        public double apply(double x, double y) { return x + y; }
    },
    MINUS("-") {
        public double apply(double x, double y) { return x - y; }
    },
    TIMES("*") {
        public double apply(double x, double y) { return x * y; }
    },
    DIVIDE("/") {
        public double apply(double x, double y) { return x / y; }
    };
    
    private final String symbol;	// 特定于常量的数据(符号)
    Operation(String symbol) { this.symbol = symbol; }	// 构造函数
    @Override public String toString() { return symbol; }	// toString 
    public abstract double apply(double x, double y);	// 特定于常量的方法
}
```



```java
public static void main(String[] args) {
    double x = Double.parseDouble("2.0");
    double y = Double.parseDouble("4.0");
    for (Operation op : Operation.values())
        System.out.printf("%f %s %f = %f%n",
                          x, op, y, op.apply(x, y));
}
```



```
2.000000 + 4.000000 = 6.000000
2.000000 - 4.000000 = -2.000000
2.000000 * 4.000000 = 8.000000
2.000000 / 4.000000 = 0.500000
```





## 枚举类型自带的 valueOf 方法



枚举类型具有自动生成的`valueOf(String)`方法，**该方法将常量名称转换为常量本身**。



 **如果在枚举类型中重写`toString`方法，请考虑编写`fromString`方法将自定义字符串表示法转换回相应的枚举类型。** 下面的代码（类型名称被适当地改变）将对任何枚举都有效，只要每个常量具有唯一的字符串表示形式：

```java
private static final Map<String, Operation> stringToEnum = new HashMap<>();
static {
    for(Operation op:values()){
        stringToEnum.put(op.toString(), op);	// 字符串与常量的映射
    }
}

public static Operation fromString(String symbol){
    return stringToEnum.get(symbol);
}
```



Java 8 以后用函数流编程可以这么写：

```java
// Implementing a fromString method on an enum type
private static final Map<String, Operation> stringToEnum =
        Stream.of(values()).collect(toMap(Object::toString, e -> e));

// Returns Operation for string, if any
public static Optional<Operation> fromString(String symbol) {
    return Optional.ofNullable(stringToEnum.get(symbol));
}
```





请注意，`Operation`枚举常量被放在`stringToEnum`的 map 中，它来自于创建枚举常量后运行的静态属性初始化。前面的代码在`values()`方法返回的数组上使用流；在Java 8之前，我们创建一个空的`hashMap`并遍历值数组，将字符串到枚举映射插入到map中，如果愿意，仍然可以这样做。但请注意，尝试让每个常量都将自己放入来自其构造方法的 map 中不起作用。这会导致编译错误，这是好事，因为如果它是合法的，它会在运行时导致`NullPointerException`。除了编译时常量属性之外，枚举构造方法不允许访问枚举的静态属性。此限制是必需的，因为静态属性在枚举构造方法运行时尚未初始化。这种限制的一个特例是枚举常量不能从构造方法中相互访问。

另请注意，fromString方法返回一个`Optional<String>`。 这允许该方法指示传入的字符串不代表有效的操作，并且强制客户端面对这种可能性。





## 如何让「特定于常量的方法」共享代码

特定于常量的方法实现的一个缺点，是它们使得难以在枚举常量之间共享代码。 例如，考虑一个代表工资包中的工作天数的枚举。 该枚举有一个方法，根据工人的基本工资（每小时）和当天工作的分钟数计算当天工人的工资。 在五个工作日内，任何超过正常工作时间的工作都会产生加班费; 在两个周末的日子里，所有工作都会产生加班费。 使用switch语句，通过将多个`case`标签应用于两个代码片段中的每一个，可以轻松完成此计算：





```java
// Enum that switches on its value to share code - questionable
enum PayrollDay {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY,
    SATURDAY, SUNDAY;

    private static final int MINS_PER_SHIFT = 8 * 60;

    int pay(int minutesWorked, int payRate) {
        int basePay = minutesWorked * payRate;

        int overtimePay;
        switch(this) {
          case SATURDAY: case SUNDAY: // Weekend
            overtimePay = basePay / 2;
            break;
          default: // Weekday
            overtimePay = minutesWorked <= MINS_PER_SHIFT ? 0 :
              	(minutesWorked - MINS_PER_SHIFT) * payRate / 2;
        }
        return basePay + overtimePay;
    }
}
```



这段代码无可否认是简洁的，但从维护的角度来看是危险的。 假设你给枚举添加了一个元素，可能是一个特殊的值来表示一个假期，但忘记在 switch 语句中添加一个相应的 case 条件。 该程序仍然会编译，但付费方法会默默地为工作日支付相同数量的休假日，与普通工作日相同。



要使用特定于常量的方法实现安全地执行工资计算，必须为每个常量重复加班工资计算，或将计算移至两个辅助方法，一个用于工作日，另一个用于周末，并调用适当的辅助方法来自每个常量。 这两种方法都会产生相当数量的样板代码，大大降低了可读性并增加了出错机会。



通过使用执行加班计算的具体方法替换`PayrollDay`上的抽象`overtimePay`方法，可以减少样板。 那么只有周末的日子必须重写该方法。 但是，这与switch语句具有相同的缺点：如果在不重写`overtimePay`方法的情况下添加另一天，则会默默继承周日计算方式。



你真正想要的是每次添加枚举常量时被迫选择加班费策略。 幸运的是，有一个很好的方法来实现这一点。 这个想法是将加班费计算移入私有嵌套枚举中，并将此策略枚举的实例传递给`PayrollDay`枚举的构造方法。 然后，`PayrollDay`枚举将加班工资计算委托给策略枚举，从而无需在`PayrollDay`中实现switch语句或特定于常量的方法实现。 虽然这种模式不如switch语句简洁，但它更安全，更灵活



```java
// The strategy enum pattern
enum PayrollDay {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY,
    SATURDAY(PayType.WEEKEND), SUNDAY(PayType.WEEKEND);

    private final PayType payType;

    PayrollDay(PayType payType) { this.payType = payType; }
    PayrollDay() { this(PayType.WEEKDAY); }  // Default

    int pay(int minutesWorked, int payRate) {
        return payType.pay(minutesWorked, payRate);
    }
    
    // The strategy enum type
    private enum PayType {
        WEEKDAY {
            int overtimePay(int minsWorked, int payRate) {
                return minsWorked <= MINS_PER_SHIFT ? 0 :
                  (minsWorked - MINS_PER_SHIFT) * payRate / 2;
            }
        },
        WEEKEND {
            int overtimePay(int minsWorked, int payRate) {
                return minsWorked * payRate / 2;
            }
        };
        
        abstract int overtimePay(int mins, int payRate);
        private static final int MINS_PER_SHIFT = 8 * 60;

        int pay(int minsWorked, int payRate) {
            int basePay = minsWorked * payRate;
            return basePay + overtimePay(minsWorked, payRate);
        }
    }
}
```





如果对枚举的 switch 语句不是实现常量特定行为的好选择，那么它们有什么好处呢?**枚举类型的 switch 有利于给外部的枚举类型增加特定于常量的行为。**例如，假设`Operation`枚举不在你的控制，你希望它有一个实例方法来返回每个**相反的操作**。你可以用以下静态方法模拟效果:



```java
// Switch on an enum to simulate a missing method
public static Operation inverse(Operation op) {
    switch(op) {
        case PLUS:   return Operation.MINUS;
        case MINUS:  return Operation.PLUS;
        case TIMES:  return Operation.DIVIDE;
        case DIVIDE: return Operation.TIMES;

        default:  throw new AssertionError("Unknown op: " + op);
    }
}
```





## 什么时候用枚举



一般来说，枚举会优先使用 `comparable` 而非 `int` 常量，与 `int` 常量相比，枚举有个小小的缺点，**即装载和初始化枚举时会有空间和时间的成本**。除了受资源约束的设备，在实践中不必在意这个问题。



>  那么什么时候应该使用枚举呢？**每当需要一组固定常量的时候**。当然，这包括“**天然的枚举类型**”，例如行星、一周的天数以及棋子的数目等等。但它也包括你在编译时就知道其所有可能值的其他集合，例如菜单的选项、操作代码以及命令行标记等。枚举类型中的常量集并不一定要始终保持不变。专门设计枚举特性是考虑到枚举类型的二进制兼容演变。





总而言之，与`int`常量相比，枚举类型的优势是不言而喻的。枚举要易读得多，也更加安全，功能更加强大。许多枚举都不需要显式的构造器或者成员，但许多其他枚举则受益于“每个常量与属性的关联”以及“提供行为受这个属性影响的方法”。只有极少数的枚举受益于将多种行为与单个方法关联。在这种相对少见的情况下，特定于常量的方法要优先于启用自有值的枚举。如果多个枚举常量同时共享相同的行为，则考虑策略枚举。

