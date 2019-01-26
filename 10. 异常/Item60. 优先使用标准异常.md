# Item60. 优先使用标准异常



专家级程序员追求并且通常也能够实现「**高度的代码重用**」，缺乏经验的程序员则未必。Java 类库提供了一组「**未受检异常**」，**它们满足了绝大多数 API 的异常抛出需要**。



重用现有异常的好处：使 API 更易于学习和使用、可读性更好，并且异常类越少，意味着内存印迹 footprint 越小，装载这些类的世界开销也越少。



## 非法参数、非法状态异常（未受检）

**最常被重用的未受检异常是** `IllegalArgumentException`，即非法参数异常，当调用者「**传递的参数不合适**」的时候，往往就会抛出这个异常。假如一个参数代表了「某个动作的重复次数」，如果程序员就企图使这个对象传递一个「负数」，就会抛这个异常。



另一个经常被重用的未受检异常是 `IllegalStateException`，即非法状态异常，如果因为接收对象的状态而使调用非法，通常就会抛出一个异常。例如，如果在某个对象被正确初始化之前，调用者企图使用这个对象，就会抛出这个异常。





可以这么说，所有**错误的方法调用**都可以被归结为**非法参数**或者**非法状态**，但是，其他还有一些标准异常也被用于某些**特定情况下的非法参数和非法状态**。如果调用者在某个不允许 `null` 值的参数中传递了 `null`，习惯的做法就是抛出 `NullPointerException`，而不是 `IllegalArgumentException`。同样地，如果调用者在表示序列下标的参数中传递了越界的值，应该抛出的就是`IndexOutOfBoundsException`，而不是 `IllegalArgumentException`。



## UnsupportedOperationException

另一个值得注意的通用异常是 `UnsupportedOperationException`。如果**对象不支持所请求的操作**，就会抛出这个异常。和本条目其他异常相比，它很少用到，因为绝大多数对象都会支持它们实现的所有方法。如果「接口的具体实现没有实现该接口定义的一个或多个可选操作」，它就可以使用这个异常。例如对于只支持追加操作的 `List` 实现，如果有人试图从列表中删除元素，它就会抛出这个异常。



## 常用的异常

下面是几种常用的异常

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-26-21-47-41_r53.png)



- `IllegalArgumentException`
- `IllegalStateException`
- `NullPointerException`
- `IndexOutOfBoundException`
- `ConcurrentModificationException`
- `UnsupportedOperationException`





虽然它们是 Java 平台类库中**迄今为止最常被重用的异常**，但是，在条件许可的情况下，**其他的异常也可以被重用**。例如，如果要实现诸如复数或者有理数之类的算术对象，也可以重用 `ArithmeticException` 和 `NumberFormatException`。如果某个异常能够满足你的需要，就不要犹豫，使用就是，不过，一定要确保抛出异常的条件与该异常的文档中描述的条件一致。这种重用必须建立在语义的基础上，而不是建立在名称的基础之上。而且，如果希望稍微增加更多的失败—捕获（ failure- capture）信息，可以放心地把现有的异常进行子类化。



最后，一定要清楚，选择重用哪个异常并不总是那么精确，因为上表中的“使用场合”并不是相互排斥的。例如，考虑表示一副纸牌的对象。假设有个处理发牌操作的方法，它的参数是发一手牌的纸牌张数。假设调用者在这个参数中传递的值大于整副纸牌的剩余张数。这种情形既可以被解释为 `IllegalArgumentException`（ `handSize`参数的值太大），也可以被解释为 `IllegalStateException`（相对于客户的请求而言，纸牌对象包含的纸牌太少）。在这个例子中感觉 `IllegalArgumentException`要好一些，不过，这里并没有严格的规则。