#####1.合理定义属性
属性不能作为out或ref参数传给方法，而字段可以
```
private String Name {
get { return null}
set{}
}
MethodWithOutParam(out Name); // 编译器报错
```
#####2.对象和集合初始化器
```
Employee e = new Employee(){ Name = "Jeff", Age = 45 };
```
#####3.匿名类型
```
var o1 = new {Name = "Jeff", Year = 1993};

String Name = "Test";
DateTime dt = DateTime.Now;
var o2 = new {Name, dt.Year};
```
匿名类型会重写Equals和GetHashCode方法

属性只读
