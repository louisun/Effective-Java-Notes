# Item46. for-each优先于for



这是一个传统的 for 循环来遍历集合：

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-24-11-03-23_r58.png)

这是一个传统的 for 循环来遍历数组：



![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-24-11-05-30_r54.png)



这些做法都比用 `while` 好，但也不是完美的。迭代器和索引遍历都会造成一些混乱，而且可能出错，也无法保证编译器能发现错误。



`for-each` 循环完全隐藏了迭代器或索引遍历，避免了混乱和出错的可能性。

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-24-11-06-53_r34.png)



对多个集合进行嵌套迭代，`for-each` 的优势更为明显：

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-24-11-08-17_r75.png)





![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-24-11-18-34_r27.png)



Bug 是对外部集合调用了太多次 next 方法。



![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-24-11-19-00_r2.png)



用 `for-each` 自动解决这个问题：



![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-24-11-19-15_r85.png)



`for-each`不近可以遍历集合和数组，还能遍历任何实现 `Iterable` 接口的对象。

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-24-11-23-44_r59.png)





总之，`for-each` 循环在简洁性和预防 Bug 方面有传统 `for` 循环无法比拟的优势，且没有性能损失，应该尽可能用 `for-each`，但是有 3 种情况不能用 `for-each` 循环：



1. **过滤**：需要遍历集合，并删除选定的元素，就要用显示的迭代器来调用其 `remove` 方法。
2. **转换**：遍历列表或数组，并取代它部分或全部的元素值，就需要列表迭代器或数组索引，以便设定元素的值。
3. **平行迭代**：如果需要并行地遍历多个集合，需要显示地控制迭代器或索引遍历，以便所有迭代器或索引遍历可以得到同步前移。

依然任何一种情况，就要用普通的 `for` 循环。



