#####1.数组
除了数组元素，数组对象占据的内存块还包括一个类型对象指针，一个同步索引块和一些额外成
员(overhead也叫做开销字段)

**二维数组**
```
Double[,] doubles = new Double[3, 5];
```
CLR还支持**交错数组**，即数组构成的数组
```
Point[][] myPolygons = new Point[3][];
myPolygons[1] = new Point[2];
myPolygons[2] = new Point[55];
```

#####2.数组转型
对于元素为引用类型的数组，CLR允许转型。CLR不允许值类型元素的数组转型
（用Array.Copy，Copy方法能够在复制元素时进行必要的类型转换）
转型条件：数组维数相同并且必须存在从元素源类型到目标类型的隐式或显式转换

#####3.所有数组都隐式派生自System.Array
System.Array类型公开了很多用于数组处理的心态方法，这些方法均获取一个数组引用作为参数
比如：BinarySearch、Clear、ConvertAll、Copy

#####4.所有数组都隐式实现IEnumerable、ICollection和IList
创建一维0基数组类型时，CLR自动使数组类型实现IEnumerable<T>，ICollection<T>和IList<T>，同时还为数组类型的所有基类实现这三个接口，只要他们是引用类型，下面的层次结构对此做出了澄清

这些都由CLR自动实现

如果数组包含值类型的元素，数组类型不回位元素的基类型实现接口
DateTime[] dtArray;
只会实现IEnumerable<DateTime>,ICollection<DateTime>和IList<DateTime>接口，不会为DateTime的基类型实现泛型接口(原因是值类型的数组在内存中的布局与引用类型不一样)

#####5.数组的传递和返回
Array.Copy执行的是**浅拷贝**
如果定义返回数组引用的方法，而且数组中不包含元素，那么方法既可以返回null也可以返回对包含0个元素的一个数组的引用（建议用后者）
```
Appointment[] apppoints = GetAppointmentForToday();
for (int i = 0; i < apppoints.Length; i++){}
```
返回null则需要先判断是否为null

#####6.创建下限非0的数组
调用数组的静态CreateInstance方法来动态创建数组。
CreateInstance为数组分配内存，将参数信息保存到数组的overhead部分，然后返回数组引用
1. 如果是2维及以上，可以吧CreateInstance返回的引用转型为一个ElementType[]变量，以简化对数组中元素的访问
2. 如果是一维，C#要求必须使用该Array的GetValue和SetValue方法访问数组元素
```
var q = (int[,]) Array.CreateInstance(typeof(int), new int[] { 5, 4 }, new int[] { 5, 4});
for(var i = q.GetLowerBound(0); i <= q.GetUpperBound(0); i++)
{
    for(var j = q.GetLowerBound(1); j <= q.GetUpperBound(1); j++)
    {
        Console.WriteLine($"i={i} j={j} value={q[i,j]}");
    }
}
```
#####7.数组
```
// 声明二维数组
int[,] a2dim =  new int[100, 50];
访问方式  a2dim[i, j]

//将二维数组声明交错数组（向量构成的向量）
int[][] ajagged = new int[100][];
for(int i = 0; i < 100; i++)
    ajagged[i] = new int[50];

// 不安全数据访问技术
private static unsafe int32 UnsafeDimArrayAcess(int[,] a) {
    int num = 0;
    fixed(int* pi = a) {
        for(int i = 0; i < 100; i++) {
            int baseOfDim = i * 100;
            for (int j = 0; j < 50; j++) {
                sum += pi[baseOfDim + j];
            }
        }
    }
}
```