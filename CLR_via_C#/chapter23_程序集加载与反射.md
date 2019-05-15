1.程序集加载
JIT编译器将方法IL代码编译成本机代码时，会查看IL代码中引用了那些类型。
在运行时，JIT编译器利用程序集的TypeRef和AssemblyRef元数据表来确定哪个程序集定义了所引用的类型
在AssemblyRef元数据表的记录项中，包含了构成程序集强名称的各个部分
JIT获取这些部分——包括名称。版本、语言文化和公钥标记——并把它们连接成一个字符串。然后JIT尝试将于该标识匹配的程序集加载到AppDomain中。如果加载的是弱命名的程序集，标识只包含程序集的名称（无版本、语言文化和公钥）

Assembly.Load导致CLR向程序集应用一个版本绑定重定向策略，并在GAC中查找程序集。没找到就接着去应用程序基目录、私有路径子目录和codebase位置查找。如果是弱命名程序集，不会应用重定向策略也不会去GAC查找
大多数动态可扩展应用程序中，Assembly的Load方法是将程序集加载到AppDomain的首选方式

Assembly.LoadFrom方法加载指定了路径名的程序集
LoadFrom流程
1、调用System.Reflection.AssemblyName类的静态GetAssemblyName方法。该该方法打开指定的文件，找到AssemblyRef元数据表的记录项，提取程序集标识信息，然后一个AssemblyName对象的形式返回这些信息（文件同时会关闭）
2、调用Assembly的Load方法，将AssemblyName对象传给它
3、CLR引用版本绑定重定向策略，并在各个位置查找匹配的程序集。找到后加载并返回

LoadFrom方法允许传递一个URL作为实参（传递一个url，并且要求联网状态，CLR会下载安装到用户的缓存目录）
一个机器可能存在多个具有相同标识的程序集，LoadFrom会在内部调用Load方法，所有CLR可能不会加载你指定的文件

Assembly.LoadFile 这个方法可从任意路径加载程序集，并且可以将相同表示的程序集多次加载到一个AppDomain中
通过LoadFile加载程序集时，CLR不会自动解析任何依赖性问题，如果更新了程序集，代码必须向AppDomain的AssemblyResolve事件等级，并让事件回调方法显示地加载任何依赖性的程序集

Assembly.ReflectionOnlyLoadFrom和Assembly.ReflectionOnlyLoad（较为少见） 用于工具只想通过反射分析程序集的元数据，而其中代码都不会执行（试图执行代码会抛出异常）

ReflectionOnlyLoadFrom方法加载由路径指定的文件：文件的强名称标识不会获取，也不会在GAC和其他位置搜索文件
ReflectionOnlyLoad会在GAC、应用程序基目录、私有路径和codebase位置查找搜索（不会应用版本控制策略，指定什么版本就是什么版本。可通过AppDomain的APplyPolicy方法控制）
利用反射来分析程序集时，代码经常需要向AppDomain的ReflectionOnlyAssemblyResolve事件注册一个回调方法，以便手动加载任何引用的程序集。回调被调用时，必须调用Assembly的ReflectionOnlyLoadFrom或ReflectionOnlyLoad方法来显式加载引用的程序集并返回对该程序集的引用
CLR不提供单独卸载程序集的能力，卸载程序集必须卸载包含它的整个AppDomain

VS中可对添加的每个DLL，将它的“生成操作”更改为“嵌入的资源”，C#编译器会把DLL文件嵌入EXE文件中，以后只需要部署这个EXE文件
在运行时，CLR会找不到依赖的DLL程序集。可以在程序初始化时，向AppDomain的ResolveAssembly事件登记一个回调方法

2.利用反射构建动态可扩展应用程序
元数据是用一系列表来存储的。生成程序集或模块时，编译器会创建一个类型定义表、一个字段定义表、一个方法定义表以及其他表。
利用System.Reflection命名空间中包含的类型，可以代码反射这些元数据表。实际上这个命名空间中的类型为程序集或模块中包含的元数据提供了一个对象模型。
利用对象模型中的类型，可以枚举类型定义元数据表中的所有类型，而针对每个类型都可获取它的基类型、它实现的接口以及类型关联的标志。利用该命名空间中的其他类型，还可解析对应元数据表来查询类型的字段、方法、属性和时间。还可发现特性。甚至允许判断引用的程序集，返回一个方法IL字节流。

3.反射的性能
缺点：
1、无法保证类型安全性，严重依赖字符串
2、反射速度慢。反射时，类型及成员的名称在编译时位置；扫描程序集的元数据时，反射机制就会不停的执行字符串搜索，而且字符串搜索不区分大小写
应该避免利用反射来访问字段或调用方法/属性。应该用以下方式来动态发现和构造类型实例
1、让类型从编译时已知的基类型派生。在运行时构造派生类的实例，将它的引用放到基类型的变量中，在调用基类型的虚方法
2、让类型 实现编译时已知的接口。...

4.反射
Assembly的ExportedTypes属性：显示其中定义的所有公开导出的类型
调用System.Reflection.IntrospectionExtensions的GetTypeInfo扩展方法将Type对象转换成TypeInfo对象

获取TypeInfo对象会强迫CLR确保已加载类型的定义程序集，从而对类型进行全面解析。

5.构造类型的实例
获得对Type派生对象的引用之后，就可以构造实例了。FCL提供了以下几个机制
1、System.Activator的CreateInstance方法
2、System.Activator的CreateInstanceFrom方法
3、System.AppDomain的方法
4、System.Reflection.ConstructorInfo的Invoke实例方法
利用前面的机制可以为除数组和委托之外的所有类型创建对象
创建数组：Array的静态CreateInstance方法
创建委托：调用MethodInfo的静态CreateDelegate方法
构造泛型类型的实例需要先获取对开放类型的引用，然后调用Type的MakeGenericType方法并向其传递一个数组（其中包含作为类型实参使用的类型）。然后，获取返回的Type对象并把它传给上面列出的某个方法
Type openType = typeof(Dictionary<,>);
Type closedType = openType.MakeGenericType(typeof(string), typeof(Int32));
Object o = Activator.CreateInstance(closedType);

5.发现类型的成员


利用反射来访问成员，三种方式
1、直接获取字段等
FieldInfo fi = obj.GetType().GetTypeInfo().GetDeclaredField("m_someField");
fi.SetValue(obj, 33);

2、创建对应方法、属性的委托（不能创建对字段的委托）
MethodInfo mi = obj.GetType().GetTypeInfo().GetDeclaredMethod("someProp");
var setSomeProp = pi.SetMethod.CreateDelegate<Action<Int32>>(obj);
setSomeProp(233);

3、dynamic
dynamic obj = Activator.CreateInstance(t, args);
obj.m_someField = 5;

6.使用绑定句柄减少进程的内存消耗
缓存类型或类型成员，但其派生对象需要大量内存，可能会有负面效果
可以使用运行时句柄代替对象以减少工作集（占用的内存）
FCL定义了三个运行时句柄RuntimeTypeHandle,RuntimeMethodHandle,RuntimeFieldHandle，三个都是值类型，都只包含一个IntPtr字段，IntPtr是一个句柄，引用了AppDomain的Loader堆中的一个类型、字段或方法

可很简单的将句柄与重量级的Type或MemberInfo对象转换
Type->RuntimeTypeHandle，调用Type的静态GetTypeHandle方法并传递Type对象引用
RuntimeTypeHandle->Type，Type的静态方法GetTypeFromHandle，传递那个Runtime...
...

