1.C#对可空值类型的支持
在C#中，Int32?等价于Nullable<Int32>，在此基础上，还允许开发人员在可空实例上执行转换和
转型，C#还允许向可空类型应用操作符

1、一元操作符(+、++、-、==、！、~)
操作数是null，结果就是null
2、二元操作符（+、-、*、/、%、&、|、^、<<、>>)
两个操作是任何一个是null，结果就是null，对于Boolean？操作数时，是个例外
两个操作数都不是null，和平常一致
两个都是null，结果就是null 
一个为null, null & true    null  		null | true  true
    null & false   false 	  null | false null
3、相等性操作符 (==、!=)
两个都是null，相等
一个是null，不相等
都是不是null，用值判断
4、关系操作符(<、>、<=、>=)
任何一个是null就是false 
两个都不是null就比较值

操作可空类型会生成相当多的IL代码，而且速度慢于非可空类型

2.C#的空结合操作符
??操作符

3.CLR对可空值类型的特殊支持
针对装箱、拆箱、调用GetType和调用接口方法提供的，使可空类型能无缝的集成到CLR中

1、装箱
对Nullable<T>进行装箱，要么返回null，要么返回已装箱的T
2、拆箱

3、通过可空值类型调用GetType
CLR会返回类型T而不是Nullable<T>
4、通过可空值类型调用接口方法
int? n = 5;
int result = ((IComparable) n).CompareTo(5); // result为0
