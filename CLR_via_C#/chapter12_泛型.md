1.泛型的优势
源代码保护：使用人员不需要访问算法的源代码
类型安全
更清晰的代码
更佳的性能：引用类型差异没那么明显，时间和垃圾时间基本相同

2.泛型的基础结构
为了使泛型工作，Microsoft必须完成以下工作
1、创建新的IL指令，使之能识别类型实参
2、修改现有元数据表的格式，以便表示具有泛型参数的类型名称和方法
3、修改各种编程语言来支持信誉发，允许开发者定义和引用泛型类型和方法
4、修改编译器使之能生成新的IL指令和修改的元数据格式
5、修改JIT编译器使之能支持类型实参的IL指令来生成正确的本机代码
6、创建新的反射成员，使开发者能查询类型和成员，以判断有没有泛型参数以及运行时创
建泛型类型和方法定义
7、修改调试器以显示和操作泛型类型、成员、字段以及局部变量
8、代码提示
泛型类型同一性
List<DateTime> dt1 = new List<DateTime>();
可简化为
internal sealed class DateTimeList : List<DateTime>{
// 无任何实现
}
表面上是方便了，但是会丧失同一性和相等性
typeof(List<DateTime>) 不等于 typeof(DateTimeList)
意味着方法原型接受DateTimeList参数，而不可将一个List<DateTime>传入；反之则可以 

C#允许使用简化的语法来引用泛型封闭类型，同时不会影响相等性
using DateTimeList = System.Collections.Generic.List<System.DateTime>;
代码爆炸
（为不同的方法，类型生成不同的方法/类型组合生成本机代码-工作集变大-损害性能）
CLR的优化措施
1、加入为特定类型实参调用了一个方法，以后再用相同类型实参调用，CLR只会为这个方	
法/类型组合编译一次代码
2、若引用类型实参都完全相同，代码能够贡献，不同程序集都会使用相同的代码（若是值
类型，CLR必须专门为那个值类型生成本机代码，因为值类型的大小不定，即使Int32和
UInt32也无法共享代码，因为可能要使用不同的本机CPU指令）
3.泛型接口

4.泛型委托
UInt32也无法共享代码，因为可能要使用不同的本机CPU指令）
public delegate TReturn CallMe<TReturn, Tkey, TValue>(TKey key, TValue value)
建议尽量使用FCL预定义的泛型Action和Func委托
5.委托和接口的逆变和协变泛型类型实参
委托的每个泛型类型参数都可标记为协变量或逆变量。利用这个功能，可将泛型委托类型的变量
转换为相同的委托类型（但泛型参数类型不同）。泛型类型参数可以是以下任何一种形式。
1、不变量  泛型类型参数不能修改
2、逆变量  意味着泛型类型参数可以从一个类更改为它的某个派生类。在C#是用in关键字标记逆
变量形式的泛型类型参数。逆变量泛型类型参数只出现在输入位置，比如作方法参数。
3、协变量  意味着泛型类型参数可以从一个类更改为它的某个基类。C#是用out关键字标记协变
量形式的泛型类型参数。协变量泛型类型参数只能出现在输出位置，比如方法返回类型。

public delegate TResult Func<in T, out TResult>(T arg);
其中T用in关键字标记，成为逆变量；TResult用out关键字标记，成为协变量
声明以下变量
Func<Object, ArgumentException> fn1 = null;
则可转型为另一个泛型类型参数不同的Func类型
Func<String, Exception> fn2 = fn1; // 不需要显式转型
示例
public interface IEnumerator<in T> : IEnumerator {}
由于T是逆变量，以下代码可顺利编译和运行
int32 Count(IEnumerator<Object> collection){}
int32 c = Count(new[] {"Test"});
6.作为out/ref实参传递的变量必须具有与方法参数相同的类型，以防止损害类型安全性

7.泛型方法和类型推断
void Swap<T>(ref T t1, ref T t2);
int n1 = 2; int n2 = 3;
Swap(ref n1, ref n2);// OK
Object o1 = "hello";
String o2 = "world";
Swap(ref o1, ref o2);// error


可定义多个方法，其中一个接受具体数据类型，另一个接受泛型类型参数
void Display(int i);
void Display<T>(T t);
8.泛型和其他成员
属性、索引器、事件、操作符方法、构造器和终结器本身不能有类型参数。但它们能在泛型类型
中定义，而且这些成员代码中可以使用类型的类型参数
9.可验证性和约束
CLR不逊于基于类型参数名称或约束来进行重载；只能基于元数（类型参数数量）对类型或方法
进行重载
10.主要约束
1、class PrimaryConstraintOfStream<T> where T : Stream {}
必定是与约束类型相同的类型或者其派生类型
2、class
约定为引用类型，任何类类型、接口类型、委托类型和数组类型都满足这个约束
3、struct
约定是值类型
11.次要约束
类型参数可以指定0-多个吃药约束，次要约束代表接口类型。这种约束代表向编译器承诺类型实
参实现了接口。
还有一种吃药约束称为类型参数约束，也称作裸类型约束。允许一个泛型类型或方法规定：指定
的类型实参要么是约束的类型要么是约束类型的派生类。一个类型参数可以指定0-多个类型参数约束
List<TBase> ConvertList<T, TBase>(IList<T> list) 
where T : TBase {
List<TBase> baseList = new List<Tbase>(list.Count);
for(int i = 0; i < list.Count; i ++) {
baseList.Add(list[i]);
}
return baseList;
}

12.构造器约束
new()
注意构造器约束和值约束不可同时使用，编译器认为是错误
13.其他可验证性问题
1、泛型类型变量的转型
将泛型类型转型为其他类型是非法的，除非是与约束兼容的类型
void CastingAGernericTypeVal<T>(T obj) {
int x = (int32) obj;// 错误
string s = (string) obj;//错误
}
转型为引用类型时还可以是用as操作符
2、将泛型类型变量设为默认值
将泛型类型变量设为null是非法的，除非将泛型类型约束成引用类型，可考虑default(T)
3、将泛型类型与null值比较
T被约束为struct，C#编译器会报错
4、两个泛型类型变量相互比较
如果泛型类型参数不能肯定是引用类型，对一个泛型类型的两个变量进行比较是非法的
5、泛型类型变量作为操作数使用
将操作符应用于泛型类型额操作数会出现大量问题