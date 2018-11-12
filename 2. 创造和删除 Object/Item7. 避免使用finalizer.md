# Item7: 避免使用finalizer

finalizer 通常是不可预测的，也是危险的，一般是不必要的，它会导致不稳定、降低性能以及可移植问题。

其实就是析构方法，Java 实际上并不需要像 C++ 那样手动释放内存，而像回收非内存资源可以用 try-finally 块来完成。

## 不能被及时执行

终结方法不能保证被及时执行，注重时间的方法不应该用 finalizer 来完成，比如关闭打开的文件，这是严重的错误。

由于 JVM 会延迟执行终结方法，导致大量的文件会被打开状态。

## 性能损失

使用终结方法会创建和销毁对象会严重变慢。



## 显示终止法

典型例子是 `InputStream`、`OutputStream`和`java.sql.Connection`上的 `close` 方法，另一个例子是 `java.util.Timer`上的 cancel 方法。

通常与 try-finally 结合起来，以确保及时终止，在 finally 子句调用显示的终止方法，可以保证及时在使用对象的时候有异常抛出，该方法也会执行：

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2018-11-09-22-51-08_r92.png)

## 什么时候用终结方法

作为安全网，或者终止非关键的本地资源。

使用中介方法，就要记住调用 `super.finalize`。

如果用终结方法作为安全网，要记得记录终结方法的非法用法。

如果要把终结方法与公有的非 final 类关联起来，考虑使用中介方法守护者，以确保即使子类的终结方法未能调用 `super.finalize`，该终结方法也会被执行。

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2018-11-09-22-57-23_r32.png)


![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2018-11-09-22-57-01_r42.png)
