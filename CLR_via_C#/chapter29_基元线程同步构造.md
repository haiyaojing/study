#### 1. 类库和线程安全
尽量避免进行线程同步。具体就是避免使用像静态字段这样的共享数据  
可尝试使用值类型  
多个线程对共享数据进行读取是没任何问题的，比如String(不可变)

线程安全的方法意味着两个线程试图同时访问数据时，数据不会被破坏  
```
public static Int32 Max(Int32 val1, Int32 val2) {
    return (val1 < val2) ? val2 : val1;
}
```
上面方法是线程安全的，即使没有任何锁  

FCL遵循:使自己的所有静态方法都线程安全，使所有实例方法都非线程安全(如果实例方法的目的是协调线程，则实例方法应该是线程安全的)

#### 2.基元用户模式和内核模式构造
有两种基元构造:用户模式(user-mode)和内核模式(kernel-mode)  

只有Windows操作系统内核才能停止一个线程的运行，在用户模式中运行的线程可能被系统抢占，但线程会以最快的速度再次调度。(自旋) (活锁 livelock)  
而内核模式要求应用程序的线程中调用由操作系统内核实现的函数。Windows会注射线程以免它浪费CPU时间(死锁deadlock)  
活锁浪费CPU时间，内存(线程栈等)  
死锁浪费内存  

#### 3.用户模式构造

CLR保证以下数据类型的变量的读写是原子性的:Boolean、Char、(S)Byte、(U)Int16、(U)Int32、(T)IntPtr、Single以及引用类型。意味着变量中的所有直接都一次性读取或写入。  
```
internal static class SomeType {
    public static Int32 x = 0;
}

// 一个线程执行
SomeType.x = 0x01234567;
```
x变量会一次性(原子性)地从0x00000000变成0x01234567 
如果是Int64时
```
SomeType.x = ox0123456789abcdef;
```
另一个线程查询就可能得到值0x0123456700000000或0x0000000089abcdef值  

本节讨论的基元用户模式构造用于规划这些原子性读写/写入操作的时间。还可强制对(U)Int64和Double类型的变量进行原子性的、规划好了时间的访问  

有两种基元用户模式线程同步构造:  
1. **易变构造(volatile construct)**  
    在特定的时间，它在包含一个简单数据类型的变量上执行原子性的读或写操作  

2. **互锁构造(interlocked construct)**  
    在特定的时间，它在包含一个简单数据类型的变量上执行原子性的读或写操作  

所有易变和互锁构造都要求传递对包含简单数据类型的一个变量的引用(内存地址)  

**易变构造**

```
internal static class StrangeBehavior {
    private static Boolean s_stopWorker = false;

    public static void Main() {
        Thread t = new Thread(Worker);
        t.Start();
        Thread.Sleep(5000);
        s_stopWorker = true;
        Console.WriteLine("Main: waiting for worker to stop");
        t.Join();
    }

    private static void Worker(Object o) {
        Int32 x = 0;
        while(!s_stopWorker) x++;
        Console.WriteLine($"Worker: stopped when x = {x}");
    }
}
```
实际上，编译器会对程序执行各种优化。当Worker方法编译时，编译器发现s_stopWorker在该方法中永远都不变化。因此如果s_stopWorker为true就显示"Worker...."。如果为false就进入一个无限循环。优化导致循环很快完成，该值只在循环前检查一次，不会在每次迭代都检查  

```
internal sealed class ThreadsSharingData {
    private int m_flag = 0;
    private int m_value = 0;

    public void Thread1() {
        m_value = 5;
        m_flag = 1;
    }

    public void Thread2() {
        if (m_flag == 1) {
            Console.WriteLine(m_value);
        }
    }
}
```
编译器和CPU在解释代码时，可能会翻转Thread1方法中的两行代码。执行顺序也不一定是期望的那样

静态System.Threading.Volatile类提供了2个静态方法  
```
public static class Volatile {
    public static void Write(ref Int32 location, Int32 value);
    public static Int32 Read(ref Int32 localtion);
}
```
这些方法比较特殊，它们实际上会禁止C#编译器、JIT编译器和CPU平常执行的一些优化  
1. Volatile.Write方法强迫location中的值在调用时写入。此外，按照编码顺序，之前的加载和存储操作必须在调用Volatile.Write之前发生  
2. Volatile.Read方法强迫location中的值在调用时读取。...  

修正上述程序
```
internal sealed class ThreadsSharingData {
    private int m_flag = 0;
    private int m_value = 0;

    public void Thread1() {
        // 注意：在1写入m_flag之前，必须先将5写入m_value
        m_value = 5;
        Volatile.Write(ref m_flag, 1);
    }

    public void Thread2() {
        // 注意：m_value必然在读取了m_flag之后读取
        if (Volatile.Read(ref m_flag) == 1) {
            Console.WriteLine(m_value);
        }
    }
}
```

**C#对易变字段的支持**
volatile关键字，可应用于一下任何类型的静态或实例字段:Boolean、(S)Byte、(U)Int32、(U)IntPtr、Single和Char。还可以将volatile关键字应用于引用类型的字段，以及基础类型为(S)Byte、(U)Int16和(U)Int32的任何枚举字段。  
JIT编译器保证对一边字段的所有访问都是以易变读取或写入的方式执行，不必显示调用Volatile的静态方法。另外volatile关键字告诉C#和JIT编译器不将字段缓存到CPUT的寄存器中，确保制度按的所有读写操作都在RAM进行  
```
internal sealed class ThreadsSharingData {
    private volatile Int32 m_flag = 0;
    private          Int32 m_value = 0;

    public void Thread1() {
        // 注意：将1写入m_flag之前，必须先将5写入m_value
        m_value = 5;
        m_flag = 1;
    }

    public void Thread2() {
        // 注意：m_value必须在读取m_flag之后读取
        if (m_flag == 1) {
            COnsole.WriteLine(m_value);
        }
    }
}
```
C#不支持以传引用的方式将volatile字段传给方法。  
```
Boolean success = Int32.TryParse("123", out m_amount);
// 上一行代码导致C#编译器生成一下警告信息:
// CS0420:对volatile字段的引用不被视为volatile
```

**互锁构造**
静态类System.Threading.Interlocked中的每个方法都执行一次原子读取以及写入操作。  
此外Interlocked的所有方法都建立了完整的内存栅栏(memory fence)。调用某个Interlocked方法之前的任何变量写入都在这个Interlocked方法调用之前执行；调用之后的任何变量读取都在这个调用之后读取。  
对Int32变量进行操作的静态方法时目前最常用的方法  
```
public static class Interlocked {
    // return (++location)
    public static Int32 Increment(ref Int32 location);

    // return (--location)
    public static Int32 Decrement(ref Int32 location);

    // return (location == value)
    // 注意：value可能是一个辅助，从而实现减法运算
    public static Int32 Add(ref Int32 location1, Int32 value);

    // Int32 old = location1; location1 = value; return old;
    public static Int32 Exchange(ref Int32 location1, Int32 value);

    // Int32 old = location1;
    // if (location1 == comparand) location1 = value;
    // return old;
    public static Int32 CompareExchange(ref Int32 location1, Int32 value, Int32 comparand);
    ...
}
```
上述方法还有一些重载版本能对Int64值进行处理。还提供了Exchaange和CompareExchange方法，接受Object等类型的参数以及这两个方法还有其泛型版本，约束为class   

参考模型:   
```
internal sealed class MultiWebRequest {
    // 这个辅助类用于协调所有异步操作
    private AsyncCoordinator m_ac = new AsyncCoordinator();

    // 这是想要查询的web服务器机器响应(异常或Int32)的集合
    // 注意：多个线程访问该字段不需要以同步方式进行，因为构造后键就是只读的
    private Dictionary<String, Object> m_servers = new Dictionary<String, Object>(
        {"http://Wintellect.com/", null},
        {"http://1.1.1.1/", null},
    );

    public MultiWebRequest(Int32 timeout = Timeout.Infinite) {
        // 异步方式发起所有请求
        var httpClient = new HttpClient();
        foreach(var server in m_servers.Keys) {
            m_ac.AboutToBegin(1);
            httpClient.GetByteArrayAsync(server).ContinueWith(task => ComputeResult(server, task));
        }
        // 告诉AsyncCoordinator所有操作已发起，并在所有操作完成、调用Cancel或者发生超时的情况下调用AllDoen
        m_ac.AllBegin(AllDone, timeout);
    }

    private void ComputeResult(String server, Task<Byte[]> task) {
        Object result;
        if (task.Exception != null) {
            result = task.Exception.InnerException;
        } else {
            result = task.Result.Length;
        }
        // 保存结果(exception/sum)，指出一个操作完成
        m_servers[server] = result;
        m_ac.JustEnded();
    }

    public void Cancel() { m_ac.Cancel(); }

    private void AllDone(CoordinationStatus status) {
        switch(status) {
            case CoordinationStatus.Cancel:
                break;
            case CoordinationStatus.Timeout:
                break;
            case CoordinationStatus.AllDone:
                    foreach(var server in m_servers.keys) {
                        Object result = server.Value;
                        if (result is Exception) {
                            ...
                        } else {
                            Console.WriteLine($"returned {result} bytes.");
                        }
                    }
                break;
        }
    }
}
```
可能存在情况：所有wen服务器请求完成、调用AllBegun、发生超时以及调用Cancel。这是AsyncCoordinator会选择1个赢家和三个输家，确保AllDone方法不被多次调用。  
```
internal sealed class AsyncCoordinator {
    private Int32 m_opCount = 1;        // AllBegun内部调用JustEnded来递减它
    private Int32 m_statusReported = 0;     // 0=false, 1=true
    private Action<CoordinationStatus> m_callback;
    private Timer m_timer;

    // 该方法必须在发起一个操作前调用
    public void AboutToBegin(Int32 opsToAdd = 1) {
        Interlocked.Add(ref m_opCount, opsToAdd);
    }

    // 该方法必须在处理好一个操作的结果之后调用
    public void JustEnded() {
        if (Interlocked.Decrement(ref m_opCount) == 0)
            ReportStatus(CoordinationStatus.AllDone);
    }

    // 该方法必须在发起所有操作之后调用
    public void AllBegun(Action<CoordinationStatus> callback, Int timeout = Timeout.Infinite) {
        m_callback = callback;
        if (timeout != Timeout.Infinite) {
            m_timer = new Timer(TimeExpired, null, timeout, Timeout.Infinite);
        }
        JustEnded();
    }

    private void TimeExpired(Object o) { ReportStatus(Coordination); }
    public void Cancel() { ReportStatus(CoordinationStatus.Cancel); }

    private void ReportStatus(CoordinationStatus status) {
        // 㘝状态从未报告过，就报告它；否则忽略它
        if (Interlocked.Exchange(ref m_statusReported, 1) == 0)
            m_callback(status);
    }
}
```

**实现简单的自旋锁**
Interlocked主要用于操作Int32值。如果需要原子性地操作类对象中的一组字段。需要采取一个办法阻止所有线程，只允许其中一个进入对字段进行操作的代码区域。  
```
internal struct SimpleSpinLock {
    private Int32 m_ReourceInUse; // 0=false(default), 1=true

    public void Enter() {
        while(true) {
            // 总是将资源设为"正在使用"(1)
            // 只有从"未使用"变成"正在使用"才会返回
            if (Interlocked.Exchange(ref m_ResourceInUse, 1) == 0) return;
            // 在这里添加"黑科技"
        }
    }

    public void Leave() {
        // 将资源标记为"未使用"
        Volatile.Write(ref m_ResourceInUse, 0);
    }
}
```
下面这个类展示了如何使用SimplePinLock  
```
public sealed class SomeResource {
    private SimpleSpinLock m_s1 = new SimplePinLock();

    public void AccessResource() {
        m_s1.Enter();
        // 一次只有一个线程才能进入这里访问资源
        m_s1.Leave();
    }
}
```
如果两个线程同时调用Enter，那么Interlocked.Exchange会确保一个线程将m_resourceInUse从0变到1，并发现m_resourceInUse为0，然后这个线程从Enter返回，使它能继续执行AccessResource方法中的代码。另一个线程会将m_resourceInUse从1变到1，所以会不停的调用Exchange进行"自旋"，知道第一个线程调用Leave。  

第一个线程Leave后会将m_resourceInUse更改回0。这造成正在"自旋"的线程能够将m_resourceInUse从0变到1，能从Enter返回。开始访问SomeResource对象的字段。  

"自旋"会浪费CPU时间，阻止CPU做其他工作。因此自旋锁只应该用于保护那些执行得非常快的代码区域。  

"自旋锁"一般不要在单CPU机器上使用。因为在这种机器上，一方面是希望获得锁的线程自旋，一方面是占有锁的线程不能快速释放锁。如果占有锁的线程优先级低于想要获取锁的线程(自旋线程)，可能占有锁的线程没机会运行。造成"活锁"情形。  

Windows有时会短暂的动态提升一个线程的优先级，因此使用自旋锁的线程，应该禁止像这样的优先级提升:参考System.Diagnostics.Process和System.Diagnostics.ProcessThread的PriorityBoostEnabled属性。  

FCL提供了System.Threading.SpinLock结构，使用SpinWait结构来增强性能。还提供了超时支持。(值类型，轻量级内存友好的对象。一定不要传递SpinLock实例，值类型会进行复制，会失去同步)  

线程可告诉系统在指定时间内不想被调度
```
// Thread
public static void Sleep(Int32 millisecondsTimeout);
```
参数为0时代表放弃本次调度，为-1代表无穷  

要求Windows在当前CPU上调度另一个线程  
```
public static Boolean Yield();
```
调用Yield的效果介于调用Thread.Sleep(0)和Thread.Sleep(1)之间。Thread.Sleep(0)不逊于较低优先级的线程运行，而Thread.Sleep(1)总是强迫进行上下文切换

**Interlocked Anything模式**

```
public static Int32 Maximum(ref Int32 target, Int32 value) {
    Int32 currentVal = target, startVal, desiredVal;

    // 不要再循环中访问目标(target)，除非是想要改变它时另一个线程也在动它
    do {
        // 记录这次循环迭代的起始值
        startVal = currentVal;
        // 基于startValu和value计算desiredVal
        desireVal = Math.Max(startVal, value);

        // 注意：线程在这里可能被抢占，以下代码不是原子性的
        // if (target == startVal) target = desiredVal;

        // 而应该使用原子性的CompareExchange方法，它返回在target在(可能)被方法修改之前的值
        currentVal = Interlocked.CompareExchange(ref target, desiredVal, startVal);

        // 如果target的值在这一次循环迭代中被其他线程改变，就重复
    } while(startVal != currentVal);

    return desiredVal;
}
```

#### 4.内核模式构造 
内核模式构造 比用户模式的构造慢得多，一个原因是它们要求Windows操作系统自身的配合，另一个原因是在类和对象上调用的每个方法都造成线程从托管代码转换为本级用户模式代码，在转换成本机内核模式代码。还要按相反方向返回。  

优点:  
1. 检测到一个资源上的竞争时，Windows会阻塞输掉的线程  
2. 内核模式的构造可实现本机和托管线程之间的同步  
3. 可同步在一台机器的不同进程中运行的线程  
4. 可应用安全性设置，防止未经授权的账户访问他们  
5. 线程可一直阻塞，知道集合中的所有内核模式构造可用，或知道集合中的任何内核模式构造可用
6. 在内核模式的构造上阻塞的线程可指定超时值；指定时间内访问不到希望的资源，线程就可以解除阻塞并执行其他任务  

事件和信号量是两种基元内核模式线程同步构造。其他内核模式构造，如互斥体则是在这两种基元模式构造上构建的。  

System.Threading.WaitHandle抽象基类。包装一个Windows内核对象句柄  
WaitHandle
    EventWaitHandle
        AutoResetEvent
        MunualResetEvent
    Semaphore
    Mutex

WaitHandle基类内部有一个SafeWaitHandle字段，它容纳了一个Win32内核对象句柄。在构造气体的WaitHandle派生类时初始化。  
在一个内核模式的构造上调用的每个方法都代表一个完整的内存栅栏(表明调用这个方法之前的任何变量写入都必须在这个方法调用之前发生；而这个调用之后的任何变量读取都必须在这个调用之后发生)  

```
public abstract class WaitHandle : MarshalByRefObject, IDisposable {
    // WaitOne内部调用Win32 WaitForSingleObjectEx函数
    public virtual Boolean WaitOne();
    public virtual Boolean WaitOne(Int32 millisecondsTimeout);
    public virtual Boolean WaitOne(TimeSpan timeout);

    // WaitAll内部调用Win32 WaitForMultipleObjectsEx函数  bWaitAll传递TRUE
    public static Boolean WaitAll(WaitHandle[] waitHandles);
    public static Boolean WaitAll(WaitHandle[] waitHandles, Int32 millisecondsTimeout);

    // WaitAny内部调用Win32 WaitForMultipleObjectsEx函数
    public static Int32 WaitAny(WaitHandle[] waitHandles);
    public static Int32 WaitAny(WaitHandle[] waitHandles, Int32 millisecondsTimeout);

    public const Int32 WaitTimeout = 258; //超时就从WaitAny返回

    // Dispose内部调用Win32 CloseHandle函数  自己不要调用
    public void Dispose();
}
```
注意:  
1. 调用WaitHandle的WaitOne方法让调用线程等待底层内核对象收到信号。如果对象收到信号返回true,超时返回false  
2. 调用WaitHandle的静态WaitAll方法，让调用线程等待所有指定的所有内核对象都收到信号。如果所有对象都收到信号就返回true,超时返回false  
3. WaitAny~,Int32是收到信号的内核对象对应的数组元素索引  
4. 传递给WaitAll和WaitAny的数组元素数不能超过64个  
5. Dispose留给GC去做~。GC知道什么时候没有线程使用对象，自动进行垃圾处理  

内核模式构造的一个常见用途是创建在任何时刻只允许它的一个实例运行的应用程序  
```
public static class Program {
    public static void Main() {
        using (new Semaphore(0, 1, "SomeUniqueStringIdentifyingMyApp", out createdNew)) {
            if (createNew) {
                // 这个线程创建了内核对象，所以肯定没有这个应用程序的其他实例正在运行
            } else {

            }
        }
    }
}
```

**Event构造**  
事件是由内核维护的Bookean变量。事件为false，在事件上等待的线程就阻塞；事件为true，就解除阻塞。  

当一个自动重置事件为true时，它只唤醒一个阻塞的线程，因为在解除第一个线程的阻塞后，内核将事件自动重置为false，造成其他线程继续阻塞。  
当一个手动重置事件为true时，它将唤醒等待它的所有线程的阻塞，因为必须要代码手动将事件重置为false  

```
public class EventWaitHandle : WaitHandle {
    public Boolean Set();           // 将Boolean设为true;总是返回true
    public Boolean Reset();         // 将Boolean设置false;总是返回true
}

public sealed class AutoResetEvent() : EventWaitHandle {
    public AutoResetEvent(Boolean initialState);
}

public sealed class ManualResetEvent() : EventWaitHandle {
    public ManualResetEvent(Boolean initialState);
}
```
通过自动重置事件创建线程同步锁(和前面的SimpleSpinLock一样的效果)  
```
internal sealed class SimpleWaitLock : IDisposable {
    private readonly AutoResetEvent m_available;
    public SimpleWaitLock() {
        m_available = new AutoResetEvent(true); // 最开始可自由使用
    }

    public void Enter() {
        // 在内核中阻塞，直到资源可用
        m_available.WaitOne();
    }
    
    public void Leave() {
        m_avaliable.Set();
    }
}
```
SimpleSpinLock 和 SimpleWaitLock的性能截然不同  
锁上没竞争时，SimpleWaitLock比SimpleSpinLock慢得多，因为SimpleWaitLock的Enter和Leave方法的每一个调用都强迫调用线程从托管代码转换成内核代码，在转换回来。  
在存在竞争的情况下，输掉的线程会被内核阻塞，不会自旋，不浪费CPU时间。  

**Semaphore构造**  
信号量是内核维护的Int32变量。信号量为0时，在信号量上等待的线程会阻塞；大于0时解除阻塞。在信号量上等待的线程解除阻塞时，内核自动从信号量的计数中减1。  

```
public sealed class Semaphore : WaitHandle {
    public Semaphore(Int32 initialCount, Int32 maximumCount);
    public Int32 Release(); //调用Release(1); 返回上一个计数
    public Int32 Release(Int32 releaseCount);
}
```

总结三种内核模式基元的行为:  

1. 多个线程在一个自动重置事件上等待时，设置事件只导致一个线程被解除阻塞  
2. 多个线程在一个手动重置事件上等待时，设置事件导致所有线程被解除阻塞  
3. 多个线程在一个信号量上等待时，释放信号量导致releaseCount个线程被解除阻塞  

因此自动重置事件在行为上和最大计数为1的信号量非常相似。区别在于，可以在一个自动重置事件上连续多次调用Set，同时仍然只有一个线程解除阻塞。在一个信号量上多次调用Release，会使它的内部计数一直递增，造成解除大量的线程的阻塞。  
```
//允许多个线程并发访问一个资源(如果所有线程以只读方式访问资源，就是安全的)
public sealed class SimpleWaitLock : IDisposable {
    private Semaphore m_avalilable;

    public SimpleWaitLock(Int32 maxConcurrent) {
        m_avalilable = new Semaphore(maxConcurrent, maxConcurrent);
    }

    public void Enter() {
        // 在内核中阻塞直到资源可用
        m_avalilable.WaitOne();
    }

    public void Leave() {
        // 让其他线程访问资源
        m_avalilable.Release(1);
    }

    public void Dispose() {
        m_avalilable.Close();
    }
}
```

**Mutex构造**

互斥体代表一个互斥的锁。它的工作方式和AutoResetEvent(或计数为1的Semaphore相似)。三者都是一次只释放一个正在等待的线程。  
```
public sealed clss Mutex : WaitHandle {
    public Mutex();
    public void ReleaseMutex();
}
```
Mutex对象会查询调用线程的Int32 ID，记录是那个线程获得了它。一个线程调用ReleaseMutex时，Mutex确保调用线程就是获取Mutex的哪个线程。否则Mutex对象的状态就不会变化，而ReleaseMutex会抛出异常。另外拥有Mutex的线程因为任何原因被终止，在Mutex上等待的某个线程会因为抛出异常而被唤醒。该异常通常会成为未处理的异常，从而终止整个进程。  

Mutex对象内部维护者一个地轨技术，指出拥有该Mutex的线程拥有了它多少次。如果一个线程当前拥有一个Mutex，而后该线程再次在Mutex上等待，技术就会递增，这个线程允许继续运行。线程调用ReleaseMutex将导致技术递减，只有计数编程0，另一个线程才能成为该Mutex的所有者。  

通常当一个方法获取了一个锁，然后调用也需要锁的另一个方法。就需要一个递归锁  
```
internal class SomeClass : IDisposable {
    private readobly Mutex m_lock = new Mutex();

    public void Method1() {
        m_lock.WaitOne();

        Method2(); // 递归的获取到锁

        m_lock.ReleaseMutex();
    }

    public void Method2() {
        m_lock.WaitOne();

        m_lock.ReleaseMutex();
    }

    public void Dispose() { m_lock.Dispose(); }
}
```
如果需要递归锁，也可以使用一个AutoResetEvent来简单的创建  
```
internal sealed class RecursiveAutoResetEvent :IDisposable {
    private AutoResetEvent m_lock = new AutoResetEvent(true);
    private Int32 m_owningThreadId = 0;
    private Int32 m_recursionCount = 0;

    private void Enter() {
        // 获取调用线程的唯一Int32 ID
        Int32 currentThreadId = Thread.CurrentThread.ManagedThreadId;

        //如果调用线程拥有锁，就递增递归计数
        if (m_owningThreadId == currentThreadId) {
            m_recursionCount++;
            return;
        }

        // 调用线程不拥有锁，等待它
        m_lock.WaitOne();

        // 调用线程现在拥有了锁，初始化拥有线程的ID和递归计数
        m_owningThreadId = currentThreadId;
        m_recursionCount = 1;
    }

    public void Leave() {
        // 如果调用线程不拥有锁，就出错了
        if (m_owningThreadId != Thread.CurrentThread.ManagedThreadId)
            throw new InvalidOperationException();
        
        // 从递归计数中减1
        if (--m_recusionCount == 0) {
            // 递归计数为1，表明没有线程拥有锁
            m_owningThreadId = 0;
            m_lock.Set();// 唤醒其他的等待线程(如果有的话)
        }
    }
}

```

虽然RecurisiveAutoResetEvent类的行为和Mutex类完全一样，但在一个线程试图递归的获取锁时，它的性能会好很多，因为跟踪线程所有权和递归的都是托管代码。