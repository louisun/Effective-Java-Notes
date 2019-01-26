# Item51. 当心字符串连接的性能



字符串连接不适合在大规模场景中，为连接 n 个字符串而重复使用字符串连接操作符，需要 $O（n^2）$ 的时间。这是由于字符串不可变导致的，每次连接内容都要拷贝。



![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-25-06-52-10_r12.png)





用 `StringBuilder` 来代替 `String`：



![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-25-19-14-09_r89.png)



这里`StringBuilder` 预先分配了一定容量的空间，但即使不知道字符串长度，也比第一种方式快 50 倍。

