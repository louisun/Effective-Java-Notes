# Item20. 类层次优于标签类



有时你可能会碰到一个类，它的实例有两个或更多的风格，并且包含一个标签属性（tag field），表示实例的风格。 例如，考虑这个类，它可以表示一个圆形或矩形：



```java
// Tagged class - vastly inferior to a class hierarchy!
class Figure {
    enum Shape { RECTANGLE, CIRCLE };

    // Tag field - the shape of this figure
    final Shape shape;

    // These fields are used only if shape is RECTANGLE
    double length;
    double width;

    // This field is used only if shape is CIRCLE
    double radius;

    // Constructor for circle
    Figure(double radius) {
        shape = Shape.CIRCLE;
        this.radius = radius;
    }

    // Constructor for rectangle
    Figure(double length, double width) {
        shape = Shape.RECTANGLE;
        this.length = length;
        this.width = width;
    }

    double area() {
        switch(shape) {
          case RECTANGLE:
            return length * width;
          case CIRCLE:
            return Math.PI * (radius * radius);
          default:
            throw new AssertionError(shape);
        }
    }
}
```



**这样的标签类具有许多缺点。 他们杂乱无章的样板代码，包括枚举声明，标签属性和switch语句。** 可读性更差，因为多个实现在一个类中混杂在一起。 内存使用增加，因为实例负担属于其他风格不相关的领域。 属性不能成为final，除非构造方法初始化不相关的属性，导致更多的样板代码。 构造方法在编译器的帮助下，必须设置标签属性并初始化正确的数据属性：如果初始化错误的属性，程序将在运行时失败。 除非可以修改其源文件，否则不能将其添加到标记的类中。 如果你添加一个风格，你必须记得给每个switch语句添加一个case，否则这个类将在运行时失败。 最后，一个实例的数据类型没有提供任何关于风格的线索。 总之，**标签类是冗长的，容易出错的，而且效率低下**。

幸运的是，像Java这样的面向对象的语言为定义一个能够表示多种风格对象的单一数据类型提供了更好的选择：**子类型化（subtyping）。标签类仅仅是一个类层次的简单的模仿。**

**要将标签类转换为类层次，首先定义一个包含抽象方法的抽象类，该标签类的行为取决于标签值。** 在`Figure`类中，只有一个这样的方法，就是`area`方法。 这个抽象类是类层次的根。 如果有任何方法的行为不依赖于标签的值，把它们放在这个类中。 同样，如果有所有的方法使用的数据属性，把它们放在这个类。`Figure`类中不存在这种与类型无关的方法或属性。

接下来，为原始标签类的每种类型定义一个根类的具体子类。 在我们的例子中，有两个类型：圆形和矩形。 在每个子类中包含特定于改类型的数据字段。 在我们的例子中，半径属性是属于圆的，长度和宽度属性都是矩形的。 还要在每个子类中包含根类中每个抽象方法的适当实现。 这里是对应于 `Figure` 类的类层次：

```java
// Class hierarchy replacement for a tagged class
abstract class Figure {
    abstract double area();
}

class Circle extends Figure {
    final double radius;

    Circle(double radius) { this.radius = radius; }

    @Override double area() { return Math.PI * (radius * radius); }
}
class Rectangle extends Figure {
    final double length;
    final double width;

    Rectangle(double length, double width) {
        this.length = length;
        this.width  = width;
    }
    @Override double area() { return length * width; }
}
```

这个类层次纠正了之前提到的标签类的每个缺点。 代码简单明了，不包含原文中的样板文件。 每种类型的实现都是由自己的类来分配的，而这些类都没有被无关的数据属性所占用。 所有的属性是 final 的。 编译器确保每个类的构造方法初始化其数据属性，并且每个类都有一个针对在根类中声明的每个抽象方法的实现。 这消除了由于缺少`switch-case`语句而导致的运行时失败的可能性。 多个程序员可以独立地继承类层次，并且可以相互操作，而无需访问根类的源代码。 每种类型都有一个独立的数据类型与之相关联，允许程序员指出变量的类型，并将变量和输入参数限制为特定的类型。

**类层次的另一个优点是可以使它们反映类型之间的自然层次关系**，从而提高了灵活性，并提高了编译时类型检查的效率。 假设原始示例中的标签类也允许使用正方形。 类层次可以用来反映一个正方形是一种特殊的矩形（假设它们是不可变的）：

```java
lass Square extends Rectangle {
    Square(double side) {
        super(side, side);
    }
}
```

**请注意，上述层中的属性是直接访问的，而不是访问方法。 这里是为了简洁起见，如果类层次是公开的，这将是一个糟糕的设计。**

总之，标签类很少有适用的情况。 如果你想写一个带有明显标签属性的类，请考虑标签属性是否可以被删除，而类是否被类层次替换。 当遇到一个带有标签属性的现有类时，可以考虑将其重构为一个类层次中。

