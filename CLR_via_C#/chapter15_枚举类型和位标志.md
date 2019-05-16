###1.枚举类型
枚举类型是强类型的
枚举类型是值类型（不能定义任何方法、属性和事件的特殊值类型），可通过扩展方法模拟添加
方法

可通过调用System.Enum的静态方法GetValues 或者 System.Type的实例方法GetEnumValues返回一个数组，数组中的每个元素都对应枚举类型中的一个符号名称，每个元素都包含符号名称的数值
通常使用Enum.IsDefined方法查看是否有定义改枚举
此方法比较慢，因为在内部使用了反射
Enum类定义了一个HasFlag方法
尽量避免使用HasFlag方法，由于它获取Enum类型的参数，所以传给它的值都必须装箱
###2.位标志

###3.向枚举类型添加方法
```
static class FileAttributesExtensionMethods {
   public static Boolean IsSet(this FileAttributes flags, FileAttributes flagToTest) {}
}
```
