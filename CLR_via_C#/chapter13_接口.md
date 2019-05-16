###1.C#编译器要求将实现接口的方法标记为public。CLR要求将接口方法标记为virtual
小例子
```
Base b = new Base();
b.Dispose(); // Base's Dispose
((IDisposable) b.Dispose() // Base'sDispose

Derived d = new Derived();
d.Dispose();// Derived's Dispose
((IDisposable) d).Dispose(); // Derived's Dispose

b = new Derived();
b.Dispose(); // base's Dispose
((IDisposable)b).Dispose(); // Derived's Dispose
```

###2.实现多个具有相同方法名和签名的接口
必须使用“显示接口方法实现”来实现这个类型的成员
```
public interface IWindow {
    Object GetMenu();
}
public interface IREstaurant{
    Object GetMenu();
}

class Test : IWindow, IRrestaurant {
    Object IWindow.GetMenu(){}
    Object IRestaurant.GetMenu(){}
    Object GetMenu(){} // 这个方法是可选的，与接口无关
}
```
代码在使用时必须将其转换成具体的接口才能调用所需的方法，例如
```
((IWindow)mp).GetMenu()
```
###3.设计：基类还是接口
IS-A对比CAN-DO关系
类型只能继承一个实现
1. 易用性 从基类派生的新类型通常比实现接口的所有方法要容易得多
2. 一致性实现 无论接口协议订立的多好，都无法保证所有人百分百正确实现它
3. 版本控制 向基类添加一个方法，所有派生类都继承该方法，而接口强迫所有继承者修改代码