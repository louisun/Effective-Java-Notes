# Item40. 谨慎设计方法签名



1. **谨慎选择方法名称**：始终遵循标准命名习惯（见第56条）。

2. **不要过于追求提供便利的方法**：方法要尽其所能，不能太多，否则会变得复杂。只有当一组操作经常备用到，才考虑提供方法。
3. 避免过长的参数列表：小于等于 4 个。可以拆分多个方法，或者创建辅助类，或者用 Builder 模式。





对于方法的参数类型，要优先使用「**接口**」而不是「**类**」（接口的实现类），否则会限制只能传入特定的实现。



对于 `boolean` 参数，要优先使用「**两个元素的枚举类型**」：

```java
public enum TemperatureScale { FAHRENHEIT, CELSIUS };
```



比如 `Theremometer.newInstance(TemperatureScale.CELSIUS)` 比 `Theremometer.newInstance(true)` 更有用











。





