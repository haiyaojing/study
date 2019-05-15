1.委托
1、非托管C/C++中，非成员函数地址 只是一个内存地址。这个地址不包含任何额外信息，比如
函数参数个数、参数类型、函数返回值以及函数的调用协定，因此不是类型安全的(非常轻量级)
2、.NET Framework的回调函数和非托管Windows编程环境的回调函数一样有用，提供了称为委
托的类型安全机制

将方法绑定到委托时，C#和CLR都允许引用类型的协变性和逆变性（值类型和void不支持，存储
结构是变化的，而引用类型始终是一个指针）
协变性：方法能返回从委托的返回类型派生的一个类型
逆变性：方法能接受委托的传入参数的基类

2.委托揭秘
编译器会为委托生成一个完整的类
delegate void Feedback(Int32 value);

所有委托类型都派生自MulticastDelegate
_target（System.Object） 当委托对象包装一个静态方法时，这个字段为null。当委托对象
包装一个实例方法时，这个字段引用的是回调方法要操作的对象
_methodPtr(System.IntPtr) 一个内部的整数值，CLR用它标记要回调的方法
_invocationList (System.Object) 通常为null。构造委托链时引用一个委托数组

每个委托对象实际都是一个包装器，其中包装了一个方法和调用该方法时要操作的对象
Feedback fbStatic = new Feedback(Program.FeedbackToConsole);
Feedback fbInstance = new Feedback(new Program().FeedbackToFile);
fbStatic、fbInstance变量将引用两个独立的、初始化好的Feedback委托对象


Delegate.Combine
Delegate.Remove

3.取得对委托链调用的更多控制
顺序调用委托链中的每一个委托，当一个委托对象出问题时，后续的都会出问题
所以MulticastDelegate提供了一个实例方法GetInvovationList，用于显式调用每一个委托
public abstract class MulticastDelegate : Delegate {
public sealed override Delegate[] GetInvocationList();
}

4.委托定义不要太多（泛型委托）
1、尽量使用.NET Framwwork中定义的委托
2、如果委托要通过C#的params关键字获取数量可变的参数，或者使用ref、out关键字以传引用
的方式传递参数，就可能不得不定义自己的委托

5.简化语法
1、不需要构造委托对象

2、不需要定义回调方法（lambda表达式）


3、局部变量不需要手动包装到类中即可传递给回调方法

推荐：一般行数大于3行的不使用lambda表达式

6.委托和反射
编码时必须知道回调方法需要多少个参数，以及参数的具体类型
System.Delegate.MethodInfo提供了一个CreateDelegate方法，允许在编译时不知道委托的所有
必要信息的前提下创建委托，以下是MethodInfo为该方法定义的重载
public abstract class MethodInfo : MethodBase {
// 构造包装了 一个静态方法的委托
public virtual Delegate CreateDelegate(Type delegateType);
// 构造包装了一个实例方法的委托；target引用this实参
public virtual Delegate CreateDelegate(Type delegateType, Object target);
}

创建好委托后，用Delegate的DynamicInvoke方法调用它
public abstract class Delegate {
public Object DynamicInvoke(params Object[] args);
}

