# Item48. 要精确，避免用float和double



`float` 和 `double` 是为科学计算和工程计算设计的，执行「二进制浮点运算」，是为了在广泛的数值范围提供「较为精确」的快速近似计算设计的，非常不适合用于「货币运算」，不应该被用于「**需要精确结果**」的场合。



```java
System.out.println(1.03 - .42)
```



输出的结果是 `0.6100000000000001`

下面这个例子，只能循环 3 次，结果是 `$0.3999999999999999`，当然是不正确的，应该是 `0.4`才对。

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-24-13-55-08_r93.png)

正确的方法是使用 `BigDecimal`、`int`、`long`进行货币计算。



![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-24-13-58-55_r45.png)



改为 `BigDecimal` 后，可以发现能循环 4 次，结果是 `$0.00`，这才是正确答案。



`BigDecimal` 有 2 个缺点：

1. 和基本类型相比，不是很方便
2. 运算很慢



另外精确计算就是用没有小数点的 `int` 或 `long`，取决于数值的大小，同时要自己处理十进制小数点，上面那个例子中最明显的做法是以「分」为单位进行运算（就变成整数的运算了）：



![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-24-14-02-30_r81.png)

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-24-14-02-50_r6.png)



