#### 1.编译器直接支持的数据类型成为基元类型
C#的int直接映射到System.Int32类型，以下四行代码都能正确编译并生成相同IL代码
![5-01](../Pictures/CLR_via_C_Sharp/05_01.png)
#### 2.C#的string关键字直接映射到System.String，所以两者没有区别

#### 3.c#的int始终映射到System.Int32，无论在32位还是64位操作系统上运行都是32位整数

#### 4.转型
C#总是对结果进行截断，而不进行向上取整
除了转型，基本类型还能携程字面值(literal)。可被看成类型本身的实例，比如 123.ToString()

#### 5.checked和unchecked
让编译器控制溢出的一个办法是使用/checked+编译器开关，溢出CLR会抛OverflowException
还可以在代码的特定区域控制溢出检查
```
UInt32 invalid = unchecked((Uint32)(-1)); //OK
Byte b = 100;
b = checked((Byte)(b + 200));// 抛出异常
```
还支持checked和uncheck语句，在块中的所有表达式都进行或不进行溢出检查
checked操作符或语句唯一的作用就是决定生成哪个版本的加、减、乘和数据转换IL指令

#### 6.引用类型和值类型
使用引用类型需要注意
1. 内存必须从托管堆分配
2. 堆上分配的每个对象都有一些额外成员，且必须初始化
3. 对象中的其他字节（为字段而设）总是设为0
4. 从托管堆分配对象时可能强制执行一次GC

#### 7.所有值类型都称为结构或枚举

#### 8.如果使用new操作符，C#会认为实例已经初始化
```
SomeVal v1; // struct
Int32 a = v1.x; // 不能通过编译，认为未初始化
```

#### 9.值类型有时候能提供更好的性能，除非满足以下条件，否则不应该将类型声明为值类型
1. 类型具有基元类型的行为。不可变类型，通常用readonly修饰。
2. 类型不需要从其他任何类型继承。
3. 内心也不派生出其他任何类型

类型实例的大小，实参通过传值方式传递，值类型可能对性能造成损害。所以一般还需要满足以下任意条件
1. 类型实例较小(16字节或者更小)
2. 类型的实例较大，但不作为方法实参传递也不从方法返回

值类型的优势是不作为对象在托管堆上分配

#### 10.值类型与引用类型的区别
1. 值类型对象有两种表现形式：未装箱与已装箱，相反引用类型处于已装箱形式
2. 值类型从System.ValueType派生。重写了Equals方法，能在两个对象字段值完全匹配的前提下返回true,还重写了GetHashCode，生成哈希码时会将对象的实例字段中的值考虑在内。
3. 由于不能讲值类型作为基类，所以不应在值类型中引入新的虚方法。所有方法都不能是抽象的，都是隐式密封的。
4. 引用类型的变量包括堆中对象的地址。而值对象的变量总是包含其基础类型的一个值，切初始化为0。
5. 值类型的复制会逐字段的复制。而引用类型只会复制内存地址。
6. 由于未装箱的值类型不在堆上分配，如果不再活动，为他们分配的存储就会自动释放，而不是等待垃圾回收。

#### 11.值类型的装箱与拆箱
将值类型转换成引用类型要使用装箱机制。
1. 在托管堆上分配内存。分配的内存包括值类型各字段所需要的内存量，还要加上托管堆,所有对象都有的两个额外字段所需要的内存量（类型对象指针、同步块索引）
2. 值类型的的字段复制到新分配的堆内存
3. 返回对象地址。现在改地址是对象引用；值类型变成了引用类型
拆箱不是直接将装箱过程倒过来，拆箱的代价比装箱小得多。拆箱其实就是获取指针的过程，拆箱不要求在内存中复制任何字节，往往紧接着拆箱会发生一次字段复制。

已装箱类型实例在拆箱时，内部发生下面这些事情。
1. 如果包含“对已装箱值类型实例的引用”变量为null，抛空异常
2. 如果引用的对象不是所需值类型的已装箱实例，抛InvalidCastException

**<font color = "#ff0000"> 因此拆箱只能拆成未装箱的值类型</font>**
```
Int32 x = 5;
Object o = x;// 装箱
Int16 y = (Int16) o; // 抛InvalidCastException
Int32 z = (Int32) 0;// 拆箱并将字段从已拆箱实例复制到栈变量中
```
有的语言(比如c++）允许在不复制字段的前提下对已装箱的值进行拆箱，拆箱返回已装箱对象中未装箱部分的地址，可直接用该指针操纵未装箱实例的字段，避免在堆上分配新对象和复制所有字段2次

#### 12.涉及装箱拆箱的几种情形
1. 调用ToString
若显示实现了ToString方法，则不必装箱，若调用的基类方法则会被装箱
2. 调用GetType
调用非虚方法GetType必须装箱，因为是从System.Object继承的
3. 接口类型的变量还是指向的已装箱变量

#### 13.对象相等性和同一性
定义自己的类型时，重写的Equals必须符合相等性的4个特征
1. Equals必须自反：x.Equals(x) -> true
2. Equals必须对称：x.Equals(y)和y.Equals(x)返回相同的值
3. Equals必须可传递：x.Equals(y) -> true; y.Equals(z) -> true 必然 z.Equals(x) ->true
4. Equals必须一致：比较的两个值不变，其返回值也不能变

通常还需要做下面几件事情
1. 让类型实现System.IEquatable<T>接口的Equals方法
2. 重载==和!=操作符方法

#### 14.对象哈希码
1. 算法要提供良好的随机分布，使哈希表获得最佳性能
2. 克制算法中调用基类的GetHashCode方法，并包含它的返回值。一般不要调用Object或ValueType的GetHashCode方法
3. 算法至少使用一个实例字段
4. 理想情况下，算法使用的字段应该不可变
5. 算法执行速度尽量快
6. 包含相同值的不同对象返回相同的哈希码

#### 15.dynamic
编译器允许使用隐式转型语法将表达式从dynamic转型为其他类型
C#编译器会生成payload代码，在运行时根据对象实际类型判断要执行什么操作
dynamic的一个限制是只能访问对象的实例成员，因为dynamic变量必须引用对象