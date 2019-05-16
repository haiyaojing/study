#### 1.实例构造器
类的修饰符为abstract，编译器生成的默认构造器的可访问性是protected，否则就是public
如果基类没提供无参构造器，派生类必须显示调用一个基类构造器，否则会报错
如果类的修饰符为static(sealed和abstract，静态类在元数据中是抽象密封类)，编译器不会生成	
默认构造器

C#编译器生成IL代码时，允许以内联(嵌入)的方式初始化实例字段，但在幕后，它会将这种语法
转换成构造器方法中的代码来执行初始化，这同时提醒我们注意代码的膨胀效应。
建议做法：

CLR允许为值类型定义构造器，但必须显示调用才会执行。C#不允许值类型定义无参构造器(默认有这个构造器)，但CLR允许。

构造器里必须将所有字段初始化，结构体中不能实例化属性或初始化字段。
下面是对值类型的全部字段进行赋值的一个替代方案
```
public SomeValType(int x) {
    this = new SomeValType(); // 会将所有值初始化为0/null，this被认为是只读的
}
```

#### 2.类型构造器
类默认没有定义类型构造器，如果定义也只能定义一个，并且类型构造器永远没有参数
```
internal sealed class SomeRefType {
    static SomeRefType() {
        // do something    被首次访问时执行这里的代码
    }
}
```
类型构造器总是私有的，因此不允许出现修饰符
虽然能在值类型中定义类型构造器，但永远不要这么做，有时CLR并不会给与执行
类型构造器只能访问类型的静态字段，并且它的常规用途就是初始化这些字段。

#### 3.操作符重载
CLR规范要求操作符重载方法必须是public和static
C#要求操作符重载方法中至少一个参数类型与当前定义这个方法的类型相同
```
public class Complex {
    public static Complex operator+(Complex c1, Complex c2){...}
}
```
#### 4.转换操作符重载
```
public sealed class Rational {
    public Rational(int num) { ... } // 由一个int32构造一个Rational
    // 将一个int32隐式构造并返回一个Rational对象
    public static implicit operator Rational(int num) {
        return new Rational(num);
    }
    // 由一个Rational类型显式返回一个int32
    public static explicit operator Int32(Rational r) {
        return r.num;
    }
}
```
implicit关键字告诉编译器为了生成代码来调用方法，不需要在源代码中间进行显式转型
explicit关键字告诉编译器只有在发现了显式转型时，才调用方法

#### 5.扩展方法
1. C#只支持扩展方法，不支持扩展属性、扩展事件、扩展操作符等
2. 扩展方法必须在非泛型的静态类中声明
3. C#编译器在静态类中查找扩展方法时，要求静态类本身必须具有文件作用域，否则可能会提
示以下消息，error CS1109:扩展方法必须在顶级静态类中定义；StringBuilderExtensions是嵌
套类
4. 由于静态类可以取任何名字，所以C#需要花一定时间来寻找扩展方法，它必须检查文件作用	
域中的所有静态类，为防止找到非想要的扩展方法，可以使用using指明指定
5. 多个静态类可以定义先沟通的扩展方法，但同时就要求使用静态方法的语法
6. 使用需谨慎，用一个扩展方法扩展一个类型时，同时也扩展了其派生类型
7. 扩展方法可能存在版本控制问题，原生添加了该方法的情况

还可以为接口定义扩展方法
```
public static void ShowItems<T>(this IEnumerable<T> collection) {
    foreach(var item in collection) {
        Console.WriteLine(item;
    }
}
```
任何表达式只要它的最终类型实现了IEnumerable<T>接口就可以调用上述扩展接口

还可以为委托类型定义扩展方法
```
public static void InvokeAndCatch<TException>(this Action<Object> d, object o) {
    where TException :Exception {
        try { d(o);}
        catch(TException) {}
}
Action<Object> action = o => Console.WriteLine(o.GetType());
action.InvokeAndCatch<NullReferenceException>(null);
```

还可以为枚举类型添加扩展方法

C#编译器允许创建委托来引用一个对象上的扩展方法

#### 6.分部方法
1. 他们只能在分布类或结构中声明
2. 分部方法的返回类型始终是void，任何参数都不能用out修饰符来标记(运行期间可能不存在)
3. 分部方法的声明和实现必须具有完全一致的签名
4. 如果没有对应的实现部分，则不能在代码中创建一个委托来引用这个分部方法
5. 分部方法总是被视为private的