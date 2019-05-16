#### 1.各种数值与Char实例的相互转换
```
char c = 'A';
//转型（强制类型转换）
int k = (int)c;// k== 65
//使用Convert类型
//(转换都以checked方式执行，数据丢失则抛异常)
char c = Convert.ToChar(10000);// 抛出异常
//使用IConvertible接口
c = （(IConvertible）65).ToChar(null);
```
#### 2.C#不允许使用new操作符从字面字符串构造String对象
```
String s = new String("hello:");//错误
```

#### 3.字符串比较
忽略语言文化是字符串比较最快的方式

#### 4.字符串留用
```
public static String Intern(String str);
```
获取一个String，获得它的哈希码，并在内部哈希表中查询是否有相匹配的。如果存在完全
相同的字符串，就返回对现有String对象的引用，否则就创建字符串副本。
```
public static String IsInterned(String str);
```
有就返回，无则返回null

虽然能提升性能和内存利用率，但是也可能影响性能，使用需谨慎，默认C#编译器关闭~

#### 5.StringBuilder
任何时候只要StringBuilder的AppendFormat方法需要获取对象的字符串表示，就会调用这个接
口的Format方法，可在内部通过一些巧妙地操作对字符串格式化进行全面控制。
```
sb.AppendFormat(provider, format, arg);
```
每次格式化时都会调用provider的Format方法