# Item2.遇到多个构造器参数要考虑使用 Builder

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2018-11-07-22-01-21_r67.png)

静态工厂和构造器都不能很好地扩展大量可选参数。



```java
// Telescoping constructor pattern - does not scale well! (Pages 10-11)
public class NutritionFacts {
    private final int servingSize;  // (mL)            required
    private final int servings;     // (per container) required
    private final int calories;     // (per serving)   optional
    private final int fat;          // (g/serving)     optional
    private final int sodium;       // (mg/serving)    optional
    private final int carbohydrate; // (g/serving)     optional

    public NutritionFacts(int servingSize, int servings) {
        this(servingSize, servings, 0);
    }

    public NutritionFacts(int servingSize, int servings,
                          int calories) {
        this(servingSize, servings, calories, 0);
    }

    public NutritionFacts(int servingSize, int servings,
                          int calories, int fat) {
        this(servingSize, servings, calories, fat, 0);
    }

    public NutritionFacts(int servingSize, int servings,
                          int calories, int fat, int sodium) {
        this(servingSize, servings, calories, fat, sodium, 0);
    }
    public NutritionFacts(int servingSize, int servings,
                          int calories, int fat, int sodium, int carbohydrate) {
        this.servingSize  = servingSize;
        this.servings     = servings;
        this.calories     = calories;
        this.fat          = fat;
        this.sodium       = sodium;
        this.carbohydrate = carbohydrate;
    }

    public static void main(String[] args) {
        NutritionFacts cocaCola =
                new NutritionFacts(240, 8, 100, 0, 35, 27);
    }
    
}
```

## 1. 使用重叠构造器

可以重叠构造器，比如

```java
NutrittonFacts cacaCola = new NutritionFacts(240,8,100,0,35,27)
```

但是不得不传入很多不想设置的参数，参数如果更多，就会失去控制。


## 2. JavaBean

另一种构造参数的方式是 `JavaBean`，即调用无参构造器来构造对象，然后一个个 `setter`，创建实例很容易，也更可读，但缺点是构造过程被分到几个调用中，构造过程中 JavaBean 可能处于不一致状态，出错调试起来很困难。此外 JavaBean 阻止了把类变成“不可变”，需要保证线程安全。

```java
// JavaBeans Pattern - allows inconsistency, mandates mutability  (pages 11-12)
public class NutritionFacts {
    // Parameters initialized to default values (if any)
    private int servingSize  = -1; // Required; no default value
    private int servings     = -1; // Required; no default value
    private int calories     = 0;
    private int fat          = 0;
    private int sodium       = 0;
    private int carbohydrate = 0;

    public NutritionFacts() { }
    // Setters
    public void setServingSize(int val)  { servingSize = val; }
    public void setServings(int val)     { servings = val; }
    public void setCalories(int val)     { calories = val; }
    public void setFat(int val)          { fat = val; }
    public void setSodium(int val)       { sodium = val; }
    public void setCarbohydrate(int val) { carbohydrate = val; }

    public static void main(String[] args) {
        NutritionFacts cocaCola = new NutritionFacts();
        cocaCola.setServingSize(240);
        cocaCola.setServings(8);
        cocaCola.setCalories(100);
        cocaCola.setSodium(35);
        cocaCola.setCarbohydrate(27);
    }
}
```

## Builder 模式

第三种方法：Builder 模式，不直接生成想要的对象，让客户端利用所有必要的参数调用构造器（或静态工厂），得到 `builder` 对象，然后在 builder 对象的基础上调用类似于 `setter` 的方法，来设置每个相关的可选参数。最后，调用 `build` 方法生成不可变的对象。





```java
// Builder Pattern 
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;

    // 静态 Builder 类
    public static class Builder {
        // 必选参数
        private final int servingSize;
        private final int servings;

        // 可选参数 - 初始化为默认值
        private int calories      = 0;
        private int fat           = 0;
        private int sodium        = 0;
        private int carbohydrate  = 0;

        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings    = servings;
        }

        // 类似 setter 的方法，会返回 Builder 本身
        public Builder calories(int val)
        { calories = val;      return this; }
        public Builder fat(int val)
        { fat = val;           return this; }
        public Builder sodium(int val)
        { sodium = val;        return this; }
        public Builder carbohydrate(int val)
        { carbohydrate = val;  return this; }

        // 最终的 build 方法，返回的是外部的 NutrititionFacts 对象
        public NutritionFacts build() {
            return new NutritionFacts(this);
        }
    }

    // 传入 builder，变量赋值为 Builder 里面的值
    private NutritionFacts(Builder builder) {
        servingSize  = builder.servingSize;
        servings     = builder.servings;
        calories     = builder.calories;
        fat          = builder.fat;
        sodium       = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }

    public static void main(String[] args) {
        NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
                .calories(100).sodium(35).carbohydrate(27).build();
    }
}
```


这样的代码容易编写，易于阅读，更重要的是 builder 模式模拟了具名的可选参数，像 Python 那样：`def func(a,b=1)`


设置了参数的 builder 生成了一个很好的抽象工厂，客户端将这一个builder传给方法，方法能为客户端创建一个或者多个对象。要使用这种用法，需要有个类型来表示 builder：

```java
public interface Builder<T> {
    public T build();
}
```

可以声明 NutritionFacts.Builder 类来实现 `Builder<NutritionFacts>`


带有 Builder 实例的方法通常利用有限制的通配符类型来约束构建器的类型参数:

```java
Tree buildTree(Builder<? extends Node> nodeBuilder){...}
```


## Builder 的不足


为创建对象，必须先创建构造器 builder，可能会影响性能。

Builder 模式比重叠构造器模更冗长，只有在很多参数的情况下才使用，比如 4 个以上。

但是如果一开始参数少，将来也可能添加参数，若开始就用构造器或静态工厂，将来可能无法控制。因此拥有潜在参数的，通常最好一开始就使用构建器。




