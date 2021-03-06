#### 1.托管堆基础
1. 调用IL指令newobj，为代表资源的类型分配内存（一般使用C#new操作符来完成）
2. 初始化内存，设置资源的初始状态并使资源可用。类型的实力构造器负责设置初始状态。
3. 访问类型的成员来使用资源（有必要可以重复）
4. 摧毁资源的状态以进行清理
5. 释放内存。垃圾回收期独自负责这一步

一般来说，大多数类型都不需要步骤4。使用需特殊清理的类型时，只是有时需要尽快处理，而
不是等着GC介入，可在这些类中调用一个额外的方法（Dispose），按自己的节奏清理资源

#### 2.从托管堆分配资源
CLR要求所有对象都从托管堆分配。进程初始化时，CLR划出一个地址空间区域作为托管堆。CLR还要维护一个指针NextObjPtr，指向下一个对象在堆中的分配位置。刚开始的时候，NextObjPtr设置为地址空间区域的基地址。一个区域被非垃圾对象填满后，CLR会分配更多的区域。

C#的new操作符导致CLR执行以下步骤
1. 计算类型的字段所需要的字节数
2. 加上对象的开销所需要的字节数。每个对象都需要两个开销字段：类型对象指针和同步块索引(32位系统各自需要32位<8字节>)
3. CLR检查区域中是否有分配对象所需要的字节数。如果有则NextObjPtr指针指向的地址放入对象，为对象分配的字节会被清零。接着调用类型的构造器，new操作符返回对象引用。在返回这个引用之前，NextObjPtr指向下一个对象放入托管堆时的地址（加上此对象占用字节数即可）

对于托管堆，分配对象只需在指针上加一个值（指向下一个可用地址），速度很快，许多程序中在内存中连续分配这些对象，所以会因为引用的“局部化”而获得性能上的提升。意味着进程的工作集会非常小，只需要使用很少的内存，从而提高了速度。还意味着代码使用的对象可以全部驻留在CPU的缓存中。

#### 3.垃圾回收算法
至于对象生存期的管理，有的系统采用的是某种引用计数算法（循环引用可能导致问题，阻止两个对象的计数器达到0）
CLR改为使用一种引用跟踪算法。引用跟踪算法只关心引用类型的变量，因为只有这种变量才能引用堆上的对象；值类型直接包含值类型实例。
引用类型变量可在许多场合使用，包括类的静态和实例字段，或者方法的参数、局部变量。我们将所有引用类型的变量称为根。
CLR开始GC时
1. 首先暂停进程中的所有线程
2. CLR进入GC的标记阶段
   
    1. CLR遍历堆所有对象将同步索引快中的一位设置为0，表明对象都需要删除
    2. CLR检查所有的活动根，查看它们引用了那些对象
    3. 任何根引用了堆上的对象，CLR则会标记那个对象，也就是将同步索引块中的位设为1（已被标记的不再检查，防止循环引用）（已标记的对象认为是可达的）
3. GC的压缩阶段
CLR对堆中已标记的对象进行移动，使他们占用连续的内存空间(恢复了引用了局部化，减少应用程序的工作集，提升了将来访问这些对象的性能，可用空间也是连续的)压缩意味着托管堆解决了本机（原生）堆的空间碎片化问题
CLR还要从每个根减去所引用的对象在内存中偏移的字节数，保证每个根还是引用之前一样的对象

如果GC后回收不了内存并且进程中没有空间来分配新的GC区域，说明进程的内存已耗尽，试图分配更多内存则会抛出OutOfMemoryException

保证了内存不泄露以及不可能访问被释放的内存造成的内存损坏

静态字段引用的对象一直存在，直到加载类型的AppDomain卸载为止，因此尽量避免使用静态字段

#### 4.垃圾回收和调试
![21-01](../Pictures/CLR_via_C_Sharp/21_01.png)

TimerCallback只执行一次，GC时发现Main方法再也没用过变量t
Microsoft提供了一个解决方案
使用C#编译器的/debug开关编译程序集时，编译器会应用System.Diagnostics.DebuggableAttribute，并为结果程序集设置DebuggingModes的DisableOptimizations标志。运行时编译方法时，JIT会将所有根的生存期延长至方法结束
在Console.ReadLine();后添加 t = null;并不可行，会被编译器优化掉；正确做法是比如t.Dispose()等保证存活的代码

#### 5.代
CLR的GC是基于代的垃圾回收期，对代码做以下假设
对象越新，生存期越短
对象越老，生存期越长
回收堆的一部分，速度快于回收整个堆
![21-02](../Pictures/CLR_via_C_Sharp/21_02.png)
![21-03](../Pictures/CLR_via_C_Sharp/21_03.png)
![21-04](../Pictures/CLR_via_C_Sharp/21_04.png)
![21-05](../Pictures/CLR_via_C_Sharp/21_05.png)
![21-06](../Pictures/CLR_via_C_Sharp/21_06.png)
![21-07](../Pictures/CLR_via_C_Sharp/21_07.png)
![21-08](../Pictures/CLR_via_C_Sharp/21_16.png)
![21-09](../Pictures/CLR_via_C_Sharp/21_17.png)

6.垃圾回收触发条件
CLR在检测第0代超过预算时触发一次GC，这是GC最常见的触发条件，下面列出其他条件
1、代码显式调用System.GC的静态Collect方法
2、Windows报告低内存情况
3、CLR正在卸载AppDomain
4、CLR正在关闭

7.大对象
目前仍未85000字节或更大的对象是大对象,CLR以不同方式对待大小对象
1、大对象不是在小对象的地址空间分配，而是在京城地址空间的其他地方分配
2、目前版本的GC不压缩大对象，因为在内存中移动它们代价过高（可能造成地址空间碎片化）
3、大对象总是说第二代

8.垃圾回收模式
CLR启动时会选择一个GC模式，进程终止前该模式不会改变
1、工作站(默认)
该模式针对客户端应用程序优化GC。GC造成的延时很低。在该模式中，GC假定机器上运行的其他应用程序都不会消耗太多的CPU资源
2、该模式针对服务器端应用程序优化GC。被优化的主要是吞吐量和资源利用。GC嘉定机器上没有运行其他应用程序，并假定机器的所有CPU都可辅助完成GC

配置文件要为应用程序添加一个gcServer元素，下面是实例配置文件
<configuration>
<runtime>
<gcServer enabled="true"/>
</runtime>
</configuration>
运行时可查询 GCSetting的IsServerGC来查询
还支持两种子模式
并发（默认）或非并发
并发：垃圾回收期有一个额外的后台线程，在运行时并发标记对象。垃圾回收期运行一个普通优先级的后台线程来查找不可达对象。通常 消耗内存大于非并发垃圾回收期
<gcConcurrent enabled="false"/>关闭并发回收器
![21-07](../Pictures/CLR_via_C_Sharp/21_08.png)

9.强制垃圾回收
可读取GC.MaxGeneration属性用来查询托管堆支持的最大代数；该属性总是返回2
还可调用GC类的Collect方法强制垃圾回收
![21-07](../Pictures/CLR_via_C_Sharp/21_09.png)
10.监视应用程序的内存使用
GC类提供了一下静态方法，可查看某一代发生了多少次垃圾回收，或者托管堆中的对象当前使用了多少内存
Int32 CollectionCount(Int32 generation);
Int64 GetTotalMemory(Boolean forceFullCollecion);

11.使用需要特殊清理的类型
包含本机资源的类型被GC时，GC会回收对象在托管堆中使用的内存，但会造成本机资源的泄露，所以CLR提供了称为终结的机制，允许对象在被判定为垃圾之后，但在对象内存被回收之前自信一些代码。任何包装了本机资源（文件、网络连接、套接字、互斥体）的类型都支持终结。
System.Object定义了受保护的虚方法Finalize。垃圾回收期判定对象是垃圾后，会调用对象的Finalize方法（如果重写）
```
class SomeType {
    //这是一个Finalize方法
    ~SomeType() {
    // 这里的代码会进入Finalize方法
    }
}
```
被视为垃圾的对象在垃圾回收完毕后才调用Finalize方法，所以 这些对象的内存不是立马回收的，因为Finalize方法可能要执行访问字段的代码，可终结对象在回收时必须存活，造成它被提升到另一代，使对象获得比正常时间长，增大了内存好用，所以应该避免终结，而且其引用的所有对象也会提升，所以要避免为引用类型的字段定义可终结对象。
Finalize方法的执行时间是控制不了的，也不保证多个Finalize方法调用顺序。
Finalize方法中不要访问定义了Finalize方法的其他类型的对象：那些对象可能已经终结了。但可以安全访问值类型的实例或者没有定义Finalize方法的引用类型的对象。调用静态方法也要当心，这些方法可能在内部访问已终结的对象，导致行为不可预测
CLR用一个特殊的、高优先级的专用线程调用Finalize方法来避免死锁 。
使用需谨慎，是为释放本机资源而设计的，强烈建议不要重写Object的Finalize方法，可以使用Microsoft在FCL中提供的辅助类，从System.Runtime.InteropServices.SafeHandle这个特殊基类派生
(从此类派生比较困难，可使用Microsoft.Win32.SafeHandles里面提供的安全句柄的派生类)

CLR赋予这个类三个很酷的功能
1. 首次构造CriticalFinalizerObject派生类型的对象时，CLR立即对继承层次结构中的所有Finalize方法进行JIT编译。构造对象时就编译这些方法，可确保对象确定为垃圾对象后本机资源肯定会得以释放。内存紧张时，CLR可能找不到足够的内存来编译Finalize方法，这会阻止Finalize方法执行造成本机资源泄露。另外Finalize方法中的代码可能引用了另一个程序集中的类型，但CLR定位该程序集失败，那么资源得不到释放
2. CLR是调用了非CriticalFinalizerObject派生类型的Finalize方法后才调用其派生类型的Finalize方法。这样托管资源类就可以在它们Finalize方法中成功访问CriticalFinalizerObject派生类型的对象。例如FileStream类的Finalize方法可以放心的将数据从内存写入磁盘，因为它知道此时磁盘文件还没关闭
3. 如果AppDomain被一个宿主应用程序强行中断，CLR将调用CriticalFinalizerObject派生类型的Finalize方法，宿主应用程序不再信任它内部运行的托管代码时，也可以利用这个功能将资源释放

SafeHandle是抽象类。
大部分本机资源都用句柄(32位系统是32位值)进行操作。
SafeHandle派生类非常有用，保证了本机资源在垃圾回收时得以释放

1. 
```
// 不健壮，直接与本机代码交互，可能在句柄赋值给handle变量后，线程中断导致泄露
static extern IntPtr CreateEventBad...
```
2. 
```
// 健壮
static extern SafeHandle CreateEventGood...
```

SafeHandle派生类还有个值得注意的功能是防止有人利用潜在的安全漏洞（一个线程试图使用一个本机资源而另一个试图释放该资源。这可能造成句柄循环使用漏洞）SafeHandle内部定义了私有字段来维护一个计数器。一旦某个SafeHandle派生对象被设为有效句柄，计数器就被设为1。每次将SafeHandle派生对象安作为实参传给一个本机方法时，CLR会自动递增计数器，类似的，当本机方法返回到托管代码时，CLR自动递减计数器，Win32 SetEvent的原型
[DllImport("Kernel32", ExactSpelling=true]
private static extern Boolean SetEvent(SafeWaitHandle swh);

如果编写或调用代码将句柄作为一个IntPtr来操作，可以在SafeHandle对象中访问，但就要显式操作应用技术企。通过SafeHandle的DangerousAddRef和DangerousRelease方法来完成。还可以通过DangerousGetHandle方法访问原始句柄
System.Runtime.InteropServices还定义了CriticalHandle类，不提供引用计数器之外和SafeHandle相同，牺牲了安全性换性能

12.使用包装了本机资源的类型
文件不一定要显式关闭(Dispose)后才能删除，有时正好另一线程造成垃圾回收。
并非一定要调用Dispose方法确保本机资源得以清理，本机资源的清理总是会发生的，调用Dispose只是控制时机。另外调用后不会讲托管对象从托管堆上删除。只有在垃圾回收后，托管堆上的内存才得以回收。这意味着即使Dispose了托管对象过去使用的任何本机资源，也能在托管对象上调用方法。
一般不在代码中显示调用Dispose，CLR的垃圾回收期写得非常好，除非是不再使用了。
using语句：语句初始化一个对象，并将它的引用保存到一个变量中，然后再块内访问该变量，编译时自动生成try块和finally块(只能用于实现了IDisposable接口的类型)

13.依赖性问题
FileStream的实现利用了一个内存缓冲区，只有缓冲区写满时，才写入文件。而且只支持字节的写入
StreamWriter支持字符和字符串的写入，接受一个Stream对象引用作为参数，向一个StreamWriter写入时，它会将数据珲春在自己的内存缓冲区中，缓冲区满时，写入Stream中。
通过SW对象写入数据后应调用Dispose，导致SW将数据Flush到Stream对象并关闭该Stream对象（SW会自动再调用FS的Dispose，手动再次调用时FS发现已经Dispose，方法什么都不会做）
如果不调用Dispose，等垃圾回收器检测到垃圾并进行终结，但垃圾回收器不保证对象的终结顺序。Microsoft的解决方案：SW类型不支持终结，所以永远不会将它的缓冲区中的数据flush到FS中。因此忘记在SW上显示调用Dispose，就会丢失数据。

14.GC为本机资源提供的其他功能  （GC类）
public static void AddMemoryPressure(Int64 bytesAllocated);
public static void RemoveMemoryPressure(Int64 bytesAllocated);
提示垃圾回收器实际需要消耗多少内存，垃圾回收器内部会监视内存压力，压力变大时，就强制进行垃圾回收

有的本机资源的数量是固定的（Windows以前限制只能创建5个设备上下文）。应用程序能够打开的文件数量也必须有限制。同样的，一个进程在执行GC前分配数百个对象（每个对象都使用极少内存）。但本机资源数量有限，通常会导致抛出异常。
System.Runtim.InteropServices提供了HandleCollector类
public sealed class HandleCollector {
public HandleCollector(String name, Int32 initialThreshold);
public HandleCollector(String name, Int32 initialThreshold, Int32 maximumThreshold);
public void Add();
public void Remove();
public Int32 Count { get;)
public Int32 InitialThreshold { get; }
public Int32 MaximumThreshold { get; }
public String Name { get; }
}

改类的对象会在内部监视这个计数，计数太大就强制进行垃圾回收。

14.终结的内部工作原理
应用程序创建新对象时，new操作符从堆中分配内存。如果对象的类型定义了Finalize方法，那么在该类型的实力构造器被调用之前，会将指向该对象的指针放到一个终结列表中
![21-10](../Pictures/CLR_via_C_Sharp/21_10.png)

垃圾回收时，垃圾回收期扫描终结列表以查找引用，找到引用后会将其从终结列表移出，并附加到freachable队列。(B、G、H内存已被回收，E、I、J还暂时不能回收，因为Finalize方法还未调用)
![21-11](../Pictures/CLR_via_C_Sharp/21_11.png)

一个特殊的高优先级CLR线程专门调用Finalize方法，freachable为空时，线程睡眠，一旦有记录项时，线程被唤醒(Finalize方法不应该对执行代码的线程做人和假设，比如访问线程的本地存储)
CLR未来可能使用多个终接器线程，所以不应假设Finalize方法会连续调用。在一个终接器线程下，可能有多个CPU分配可终结对象，但只有一个线程，造成线程跟不上分配的速度，从而产生性能和伸缩性方面的问题
freachable列表可以看成是静态字段那种的一个根。因此队列引用是它指向的对象保持可达、不是垃圾。因此对象被复活了（既是垃圾又不是垃圾）
标记freachable对象时，将递归标记对象中的引用类型的字段所引用的对象，所以这些对象也复活了，之后垃圾回收期才结束对垃圾的标识。在这个过程中，一些垃圾对象被复活了。然后垃圾回收期压缩可回收的内存，将复活的对象提升到较老的一代（不理想）。现在，特殊的终结线程清空freachable队列，执行每个对象的Finalize方法。
下一次对老一代进行垃圾回收时，清理内存（可终结对象需要两次垃圾回收才能释放占用的内存，由于被提升至另一代，可能不止两次）
![21-12](../Pictures/CLR_via_C_Sharp/21_12.png)

15.手动监视和控制对象的生存期
CLR为每个AppDomain都提供了一个GC句柄表（GC Handle table)，允许应用程序监视或手动控制对象的生存期
初始空白
表中每个记录项都包含：
对托管堆中一个对象的引用
指出如何监视或控制对象的标记(flag)
使用System.Runtime.InteropServices.GCHandle类型在表中添加或删除记录项
![21-13](../Pictures/CLR_via_C_Sharp/21_13.png)

GCHandleType，这是一个标志，指定怎么控制/监视对象，是枚举类型
1. Weak
    该标志允许监视对象的生存期，可检测垃圾回收期在什么时候判定对象在应用程序中不可达
2. WeakTrackResurrection
    与Weak不同的是，此时对象的Finalize方法已经执行，而Weak可能执行
3. Normal
    该标志允许控制对象的生存期。即使没有根引用该变量，该对象也必须留在内存中。该对象内存可以压缩。默认标志
4. Pinned
    该标志允许控制对象的生存期。与Normal不同的是，垃圾回收时，不可压缩。需要将内存地址交给本机代码时，本机代码知道GC不会移动对象，所以可以放心地向托管堆内存写入

调用实例的Free方法使实例无效
下面展示垃圾回收器使用GC句柄表，当垃圾回收发生时
垃圾回收期标记所有可达的对象。然后垃圾回收期扫描GC句柄表，所有Normal，Pinned对象看做是根，同时也进行标记
垃圾回收器扫描GC句柄表，查找Weak记录项。如果一个Weak记录项引用了未标记的对象，该引用标识就是不可达对象，该记录项的引用值更改为null
垃圾回收期扫描终结列表。在列表中，对未标记对象的引用标识是不可达对象，这些引用会移至freachable队列，又变成可达了。
垃圾回收期扫描GC句柄表，查找所有WeakTrackResurrection记录项。如果一个记录项引用了未标记的对象（它现在是由freachable队列中的记录项引用的），该引用标识就是不可达对象，置为null
垃圾回收器对内存进行压缩，内存碎片整理（Pinned对象不会压缩）

需要将托管对象的指针交给本机代码时使用Normal标记因为本机代码将来要回调托管代码并传递指针。但不能就这么将托管对象的指针交给本机代码，因为垃圾回收发生时，对象可能在内存中移动，指针便无效了。
解决方案：
调用GChandle的Alloc方法，传递对象引用和Normal标志
将返回的GCHandle实例转型为IntPtr，将IntPtr传给本机代码
本机代码返回托管代码时，托管代码将传递的IntPtr转型为GCHandle，查询Target属性获得托管对象的引用（当前地址）
本机代码不需要时，调用GCHandle的Free方法释放

本机代码并没有使用托管对象本身，只是通过一种方式引用了对象，有时本机代码需要真正的使用托管对象，这时托管对象必须固定（pinned),从而阻止垃圾回收期压缩对象
常见例子：将托管String对象传给某个Win32函数

使用CLR的P/Invoke机制调用方法时，CLR会自动固定实参，并在本机方法返回时自动解除固定，大多数情况下都不需要使用GCHandle类型来显式固定托管对象，只有在将托管对象的指针传给本机代码，然后本机函数返回，但本机代码将来仍需要使用这些对象时，才显示使用GCHandle类型，最常见的例子就是执行异步IO操作。

C#提供了一个fixed语句，能在一个代码块中固定对象，C#的fixed语句比分配一个固定GC句柄高效。C#编译器会在此局部变量上生成一个特殊的“已固定”标志。找fixed块的尾部还会生成IL指令将局部变量设回null,使变量不再引用任何对象

Weak与WeakTrackResurrection标志（前者更常用）
例子：A定时调用B上面的方法，A持有B的引用，那么就阻止了B的回收。在极少数情况下，可能希望B存活于托管堆中，A就调用B，此时就可以A调用GCHandle的Alloc方法，向方法传递B的引用和Weak标志，A只需要保存返回的GCHandle实例而不是对B的引用
还提供了WeakReference<T>类来提供帮助(只支持弱引用，缺点，它的实例必须在堆上分配)
![21-14](../Pictures/CLR_via_C_Sharp/21_14.png)

将一些数据与另一个实体关联，例如数据可以和一个线程或AppDomain关联
![21-15](../Pictures/CLR_via_C_Sharp/21_15.png)
注意：
1. 多次添加同一个对象的引用，会有异常，必须先删除key再添加
2. 线程安全（性能不出众）
3. 表对象内部存储了对key传递的对象的弱引用（一个WeakReference对象）
4. 保证了只要key标识的对象在内存中，value就一定在内存中，使其超越了普通的WeakReference（普通的即使key保持存活，值也可能被回收）。
