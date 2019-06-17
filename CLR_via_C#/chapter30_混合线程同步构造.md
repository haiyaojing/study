#### 1.一个简单的混合锁
```
internal sealed class SimpleHybridLock : IDisposable {
    // Int32由基元用户模式构造Interlocked的方法使用
    private Int32 m_waiters = 0;
    // AutoResetEvent是基元内核模式构造
    private AutoResetEvent m_waiterLock = new AutoResetEvent(false);

    public void Enter() {
        // 指出这个线程想要获得锁
        if (Interlocked.Increment(ref m_waiters) == 1)
            return; // 锁可自由使用，无竞争，直接返回
        // 另一个线程拥有锁(发生竞争)，使这个线程等待
        m_waiterLock.WaitOne(); // 这里产生较大的性能影响
        // 拿到锁
    }

    public void Leave() {
        // 这个线程准备释放锁
        if (Interlocked.Decrement(ref m_waiters) == 0)
            return; // 没有其他线程在等待，直接返回
        
        // 有其他线程正在阻塞，唤醒其中一个
        m_waiterLock.Set(); //这里产生较大的细嫩影响
    }

    public Dispose() { m_waiterLock.Dispose(); }
}
```

#### 2.自旋、线程所有权和递归
转换为内核模式会造成巨大的性能损失，而且线程占有锁的时间通常都很短，为了提高应用程序的总体性能，可以让一个线程在用户模式中自旋一小段时间，再转换成内核模式。  

```
internal sealed class AnotherHybridLock : IDisposable {
    private Int32 m_waiters = 0;

    private AutoResetEvent m_waiterLock = new AutoResetEvent(true);

    private Int32 m_spincount = 4000;

    public void Enter() {
        Int32 threadId = Thread.CurrentThread.ManagedThreadId;
        if (threadId == m_owningThreadId) {
            m_recursion++;
            return;
        }

        SpinWait spinWait = new SpinWait();
        for(Int32 spinCount = 0; spinCount < m_spincount; spinCount++) {
            // 如果锁可以使用时，这个线程就获取它
            if (Interlocked.CompareExchange(ref m_waiters, 1, 0) == 0) goto GotLock;

            // 黑科技:给其他线程运行的机会，希望锁被释放
            spinWait.SpinOnce();
        }

        // 自旋结束
        if (Interlocked.Increment(ref m_waiters) > 1) {
            // 仍然是竞态，这个线程必须阻塞
            m_waiterLock.WaitOne(); // 等待锁
        }

    GotLock:
        // 一个线程获得锁时，记录ID，并指出线程拥有锁一次
        m_owningThreadId = threadId;
        m_recursion = 1;
    }

    public void Leave() {
        Int32 threadId = Thread.CurrentThread.ManagedThreadId;
        if (threadId != m_owningThreadId)
            throw new SynchronizationLockException("Lock not owned by calling thread");
        
        // 递减递归计数。
        if (--m_recursion > 0) return;

        m_owningThreadId = 0;

        m_waiterLock.Set();
    }

    public void Dispose() { m_waiterLock.Dispose(); }
}
```
在锁中添加的每个行为都回影响它的性能  

#### 3.FCL中的混合构造

ManualResetEventSlim类和SemaphoreSlim类
这两个构造的工作法师与对应的类和模式构造完全一致，只是他们都在用户模式中"自旋"，而且都推迟到发生第一次竞争时，才创建类和模式的构造。  
他们的Wait方法允许传递一个超时值和一个CancellationToken  
```
//提供的简化版本的 <see cref="T:System.Threading.ManualResetEvent" />。
public class ManualResetEventSlim : IDisposable
{
    public WaitHandle WaitHandle { get; }
    /// <summary>获取是否已设置事件。</summary>

    public bool IsSet { get; private set; }
    
    public int SpinCount { get; private set; }
    
    public void Reset();

    public void Wait();

    public void Wait(CancellationToken cancellationToken);

    public bool Wait(int millisecondsTimeout);
  
    public bool Wait(int millisecondsTimeout, CancellationToken cancellationToken);
    /// <summary>
    ///   释放 <see cref="T:System.Threading.ManualResetEventSlim" /> 类的当前实例所使用的所有资源。
    /// </summary>
    public void Dispose();
    /// <summary>
    ///   释放由 <see cref="T:System.Threading.ManualResetEventSlim" /> 占用的非托管资源，还可以另外再释放托管资源。
    /// </summary>
    /// <param name="disposing">
    ///   为 true 则释放托管资源和非托管资源；为 false 则仅释放非托管资源。
    /// </param>
    protected virtual void Dispose(bool disposing);
}

//对可同时访问资源或资源池的线程数加以限制的 <see cref="T:System.Threading.Semaphore" /> 的轻量替代。
public class SemaphoreSlim : IDisposable
{
    //可以输入信号量的剩余线程数。</returns>
    public int CurrentCount { get; }

    public WaitHandle AvailableWaitHandle { get; }
   
    public SemaphoreSlim(int initialCount);
    
    public SemaphoreSlim(int initialCount, int maxCount);
    
    public void Wait();
    
    public void Wait(CancellationToken cancellationToken);
    
    public bool Wait(int millisecondsTimeout);
    
    public bool Wait(int millisecondsTimeout, CancellationToken cancellationToken);
    /// <returns>输入信号量时完成任务。</returns>
    public Task WaitAsync();
    
    public Task WaitAsync(CancellationToken cancellationToken);
    
    protected virtual void Dispose(bool disposing);
}
```
虽然没有一个AutoResetEventSlim类，但可以构造一个SemaphoreSlim对象，并将maxCount设为1。  

**Monitor类和同步块**  
最常用的混合型同步构造就是Monitor类，它提供了支持自旋、县城所有权和递归的互斥锁。  
堆中的每个对象都可关联一个名为**同步块**的数据结构。同步块包含字段，这些字段和前面的AnotherHybridLock类的字段相似。  
为内核对象、拥有线程ID、递归计数以及等待线程技术提供了相应的字段。  

Monitor是静态类，它的方法接受对任何堆对象的引用。这些方法对指定对象的同步块中的字段进行操作。  
```
public static class Monitor {
    public static void Enter(Object obj);
    public static void Exit(Object obj);

    // 还可指定尝试进入锁时的超时值
    public static Boolean TryEnter(Object obj, Int32 millisecondsTimeout);

    public static void Enter(Object obj, ref Boolean lockTaken);
    public static void TryEnter(Object obj, Int32 millisecondsTimeout, ref Boolean lockTaken);
}
```
为堆中每个对象都关联一个同步块数据结构显得很浪费，尤其是大多数对象的同步块都不使用。  
CLR的处理办法:CLR初始化时在堆中分配一个同步块数组(每当一个对象在堆中创建的时候，都有两个额外的开销字段与它关联。第一个是"类型对象指针",第二个是"同步块索引")  

一个对象在构造时，它的同步块索引初始化为-1，表明不引用任何同步块。  
调用Monitor.Enter时，CLR在数组中找到一个空白同步块，并设置对象的同步块索引，让它引用该同步块。(动态关联)  
调用Exit时，会检查是否有其他任何线程正在等待使用对象的同步块。如果没线程等待，同步块就自由了，Exit将对象的同步块索引设回-1。  
![30-01](https://note.youdao.com/yws/api/personal/file/C425108B632A43D28091425383A3630B?method=download&shareKey=5fe4c8c1bd1218051eebdacd7311f1bb)  

以下代码演示了Monitor类原本的使用方式  
```
internal sealed class Transaction {
    private DateTime m_timeOfLastTrans;

    public void PerformTransaction() {
        Monitor.Enter(this);
        m_timeOfLastTrans = DateTime.Now;
        Monitor.Exit(this);
    }

    public DateTime LastTransaction {
        get {
            Monitor.Enter(this);
            DateTime tmp = m_timeOfLastTrans;
            Monitor.Exit(this);
            return tmp;
        }
    }
}
```
上述代码可能存在问题。每个对象的同步块索引都隐式为公共的。  
```
public static void SomeMethod() {
    var t = new Transaction();
    Monitor.Enter(t); // 这个县城获取对象的公共锁

    // 让一个线程池线程显示LastTransaction时间
    // 线程池会阻塞，知道SomeMethod调用了Monitor.Exit!
    ThreadPool.QueueUserWorkItem(o => Console.WriteLine(t.LastTransaction));

    // 这里执行一些代码
    Monitor.Exit(t);
}
```
建议始终坚持使用私有锁。  
```
private readonly Object m_lock = new Object(); //现在每个Transaction对象都有私有锁

Monitor.Enter(m_lock); 

Monitor.Exit(m_lock);
```

可以发现Monitor根本不应该是实现成静态类，它应该像其他所有同步构造那样实现。  
下面对这些额外的问题进行了总结:  
1. 变量能引用一个代理对象(MarshalByRefObject)。调用Monitor的方法时，传递对代理对象的引用，锁定的是带你对象而不是实际对象。  
2. 如果线程调用Monitor.Enter，传递对类型对象的引用，而且这个类型对象是以"AppDomain中立"的方式加载的，线程就会跨越进程中的所有AppDomain在那个类型上回去锁。破坏了AppDomain本应该提供的隔离能力  
3. 由于字符串可以留用，所以两个完全独立的代码段可能在不知情的情况下获取对内存中一个String对象引用。两个独立的代码段可能在不知情的情况下以同步方式执行  
4. 由于Monitor的方法要获取一个Object，所以值类型会导致值类型装箱，造成线程在已装箱对象上获取锁。每次调用Monitor.Enter都会在一个完全不同的对象上获取锁，造成完全无法实现线程同步。  
5. 向方法应用[MethodImpl(MethodImplOptions.Synchronized)]特性，造成JIT用Monitor.Enter和Monitor.Exit调用包围方法的本机代码。如果方法是实例方法，会将this传递，锁定隐式公共的锁。如果方法是静态的，对类型的类型对象的引用会传递给这些方法，造成锁定"AppDomain中立"的类型。**建议不要使用这个特性**  
6. 调用类型的类型构造器时，CLR要获取类型对象上的一个锁，确保只有一个线程初始化类型对象及其静态字段。同样的，这个类型可能以"AppDomain中立"的方式加载，所以会出问题。**尽量避免使用类型构造器或者保持他们的短小和简单**。  

**lock关键字**
```
lock(this) {

}
```
等价于
```
Boolean lockTaken = false;
try {
    Monitor.Enter(this, ref lockTaken);
    // ...
}
finally {
    if (lockTaken)
        Monitor.Exit(this);
}
```
但在try块，在更改状态时中发生异常，就会损坏状态。锁在finally块中退出时，另一个线程可能开始操作损坏的状态。  
进入try块和离开try块都会影响方法的性能。  
**尽量少使用lock语句**  

**Boolean lockTaken变量**  
加入一个线程进入try块但在Monitor.Enter之前退出，finally得以调用但它的代码不应退出锁。  
Monitor.Enter成功获得锁，Enter方法会将lockTaken设为true  

**ReaderWriterLockSlim类**  
可以构造下面这样的控制线程  
1. 一个线程向数据写入时，请求访问的其他所有线程都被阻塞  
2. 一个线程从数据读取时，请求读取的其他线程允许继续执行，但请求写入的线程被阻塞  
3. 向线程写入的线程结束后，要么解除一个写入线程writer的阻塞，使它能向数据写入，要么解除所有读取线程reader的阻塞，使他们能够并发读取数据。如果没有线程被阻塞，锁就进入可以被自由使用的状态  
4. 从数据读取的所有线程结束后，一个writer线程被解除阻塞，使它能向数据写入。如果没线程被阻塞，锁就进入可以自由使用的状态  

```
public class ReaderWriterLockSlim : IDisposable {
    public ReaderWriterLockSlim(LockRecursionPolicy recursionPolicy);
    public void Dispose();

    public void EnterReadLock();
    public Boolean TryEnterReadLock(Int32 millisecondsTimeout);
    public void ExitReadLock();

    public void EnterWriteLock();
    public Boolean TryEnterWriteLock(Int32 millisecondsTimeout);
    public void ExitWriteLock();
}
```
例子:
```
internal sealed class Transaction : IDisposable {
    private readonly ReaderWriteLockSlim m_lock = new ReaderWriteLockSlim(LockRecursionPolicy.NoRecursion);
    private DateTime m_timeOfLastTrans;

    public void PerformTransaction() {
        m_lock.EnterWriteLock();
        // 以下代码拥有对数据的独占访问权
        m_timeOfLastTrans = DateTime.Now;
        m_lock.ExitWriteLock();
    }

    public DateTime LastTransaction {
        get {
            m_lock.EnterReadLock();
            DateTime temp = m_timeOfLastTrans;
            m_lock.ExitReadLock();
            return temp;
        }
    }
}
```
```
public enum LockRecursionPolicy {
    NoRecursion,
    SupportsRecursion // 支持线程所有权和递归行为(reader-writer锁支持的代价很高昂)
}
```
ReaderWriteLockSlim还提供了允许一个reader线程升级为writer线程。以后线程还可以把自己降级回reader线程。(虽然可以支持先读在写，但没什么太大的作用)  

**OneManyLock类**
```
public sealed class OneManyLock : IDisposable {
    public OneManyLock();
    public void Dispose();

    public void Enter(Boolean exclusive);
    public void Leave();
}
```
内部定义了一个Int64字段来存储锁的状态、一个供reader线程阻塞的Semaphore对象以及一个供writer线程阻塞的AutoResetEvent对象。Int64状态字段分解成一下4个子字段  
1. 4位锁代表锁本身的状态。1=OwnedByWriter，2=OwnedByReaders，3=OwnedByReadersAndWriterPending，4=ReservedForWriter  
2. 20位(0~1048575的一个数)代表锁当前允许进入的、正在读取的reader线程的数量(RR)  
3. 20位代表锁当前正在等待进入锁的reader线程的数量(RW)。这些线程在自动重置事件对象上阻塞  
4. 20为代表正在等待进入锁的writer线程的数量(WW)。这些线程在其他信号量对象上阻塞。  
由于所有与锁相关信息都在单个Int64上，所以可以用Interlocked类的方法来操纵这个字段，使锁的速度非常快，而且线程只有在存在竞争时才阻塞。  

下面说明了线程进入一个锁进行共享访问时发生的事情。  
1. 如果锁的状态是Free:将状态设为OwnedByReaders,RR=1,返回  
2. 如果锁的状态是OwnedByReaders:RR+,返回  
3. 否则:RW++,阻塞reader线程。线程醒来时，循环并重试  

下面说明了共享访问的一个线程离开锁发生的事情。  
1. RR--。
2. 如果RR>0:返回。
3. 如果WW>0:将状态设为ReservedForWriter,WW--,释放一个阻塞的writer线程，返回。
4. 如果RW==0&&WW==0:将状态设为Free，返回。

下面说明了一个线程进入锁进行独占访问时发生的事情。  
1. 如果锁为Free:将状态设为OwnedByWriter，返回。  
2. 如果锁状态为ReservedForWriter:将状态设为OwnedByWriter，返回。  
3. 如果锁为OwnedByWriter：WW++，阻塞writer线程。线程醒来时，循环并重试。  
4. 否则:将状态设为OwnedByReaderAndWriterPending，WW++，阻塞writer线程。线程醒来时，循环并重试。  

下面说明了进行独占访问的一个线程离开锁时发生的事情。  
1. 如果WW==0&&RW==0:将状态设为Free,返回。
2. 如果WW>0:将状态设为ReservedForWriter，WW--，释放一个阻塞的writer线程，返回。
3. 如果WW==0&&RW>0:将状态设为Free,RW=0,唤醒所有阻塞的reader线程，返回。  

注意，所有的位操作都要使用前面的Interlocked Anything模式描述的技术来执行。将操作都转换成线程安全的原子操作。  

**CountdownEvent类**
构造阻塞一个线程，直到它的内部计数器变成0。从某种角度说，和Semaphore的行为相反。  
```
public class CountdownEvent : IDisposable {
    public CountdownEvent(Int32 initialCount);
    public void Dispose();
    public void Reset(Int32 count); // 将CurrentCount设为count
    public void AddCount(Int32 signalCount); // 将CurrentCount递增signalCount
    public Boolean TryAddCount(Int32 singalCount); 
    public Boolean Sinal(Int32 signalCount); // 使CurrentCount递减signalCount
    public Boolean Wait(Int32 millisecondsTimeout, CancellationToken cancellationToken);
    public Int32 CurrentCount { get; }
    public Boolean IsSet { get; } // 如果CurrentCount为0，就返回true
    public WaitHandle WaitHandle { get; }
}
```
一旦CountdownEvent的CurrentCount变成0，它就不能更改了。

**Barrier类**  
控制的一系列线程需要并行工作，从而在一个算法的不同阶段推进。  
例子:当CLR使用它的GC的服务器版本时，GC算法为每个内核都创建了一个线程。这些在不同的应用程序线程的栈向上移动，并发标记堆中的对象。每个线程完成了它自己的那一部分工作之后必须听下来等待其他线程完成。所有线程都标记好后，线程就可以并发的压缩堆的不同部分。每个线程完成它的那部分的堆的压缩之后，线程必须阻塞以等待其他线程。所有线程都完成了对自己那一部分的堆的压缩之后，所有县城都要在应用程序的线程的栈中三星，对根进行修正。只有所有线程都完成这个工作之后，GC的工作才算真正完成。  
```
public class Barrier : IDisposable {
    public Barrier(Int32 paricipantCount, Action<Barrier> postPhaseAction);
    public void Dispose();
    public Int64 AddParticipants(Int32 participantCount); // 添加参与者
    public void RemoveParticipants(Int32 paarticipantCount); // 减去参与者
    public Boolean SignalAndWait(Int32 millisecondsTimeout, CancellationToken cancellationToken);

    public Int64 CurrentPhaseNumber { get; } // 指出进行到哪一个阶段(从0开始)
    public Int32 ParticipantCount { get; } // 指出参与者数量
    public Int32 ParicipantsRemaining { get; } // 需要调用SignalAndWait的线程数
}
```

**线程同步构造小结**  
代码尽量不要阻塞任何线程。  
执行异步计算或IO操作时，将数据交给另一个线程时，应避免多个线程同时访问数据。如果不能完全做到，尽量使用Volatile和Interlocked方法，速度很快而且不阻塞线程。也可以用Interlocked Anything模式执行丰富的操作。  

主要在下面两种情况阻塞线程  
1. 线程模型很简单 
2. 线程有专门用途

要避免阻塞线程，就不要刻意的为线程打上标签。  
如果一定要阻塞线程，为了同步在不同AppDomian或进程中运行的线程，请使用内核模式构造。  
要在一系列操作中原子性地操纵状态，请使用带私有字段的Monitor类。  
避免使用递归锁。  
不要再finally块中释放锁。  
写代码来占有锁，时间不要太长，否则会增大线程阻塞的几率。  
对于计算限制的工作可以使用任务二避免使用大量线程同步构造。  

#### 4.双检锁
开发人员用它将单例对象的构造推迟到程序首次请求该对象时进行。  
以下代码演示了如何用C#实现双检锁技术  
```
public sealed class Singleton {
    private static Object s_lock = new Object();

    private static Singleton s_value = null;

    private Singleton() {}

    public static Singleton GetSingleton() {
        if (s_value != null) return s_value;

        Monitor.Enter(s_lock);
        if (s_value == null) {
            Singleton temp = new Singleton();
            // 保证temp的引用只有在构造器结束执行之后才发布到s_value字段。
            Volatile.Write(ref s_value, temp);
        }
        Monitor.Exit(s_lock);
        return s_value;
    }
}
```
CLR保证对类构造器的调用时线程安全的,所以也可以(不使用双检锁)
```
internal sealed class Singleton {
    private static Singleton s_value = new Singleton();
    ...
}
```
不过缺点是未调用方法，只访问静态字段也会导致单例的创建  

通过定义嵌套类来解决这个问题  
```
internal sealed class Singleton {
    private static Singleton s_value = null;

    private Singleton() {}

    public static Singleton GetSingleton() {
        if (s_value != null) return s_value;

        Singleton temp = new Singleton();
        Interlocked.CompareExchange(ref s_value, temp, null);
        // 如果这个线程竞争失败，新建的第二个实例对象会被垃圾回收
        return s_value;
    }
}
```
以上代码虽然可能创建多个Singleton对象，但它的速度非常快并且不阻塞线程。(创建线程会分配并初始化更多的内尺寸，而且所有dll都会受到一个线程连接通知)  

FCL有两个类型封装了本节描述的模式  
```
public class Lazy<T> {
    public Lazy(Func<T> valueFactory, LazyThreadSafetyMode mode);
    public Boolean IsValueCreated { get; }
    public T value { get; }
}
```

```
public static void Main() {
    Lazy<String> s = new Lazy<String>(() => DateTime.Now.ToLongTimeString(), true);

    Console.WriteLine(s.IsValueCreated); // false
    Console.WriteLine(s.Value); // 调用委托
    Console.WriteLine(s.IsValueCreated); // true
    Thread.Sleep(10000); // 
    Console.WriteLine(s.Value); // 委托没有调用，显示相同结果
}
```

LazyThreadSafetyMode标志  
```
public enum LazyThreadSafetyMode {
    None,                       // 完全没有线程安全支持(适合GUI应用程序)
    ExecutionAndPublication,    // 双检锁技术
    PulicationOnly,             // 使用Interlocked.CompareExchange技术
}
```
内存有限时，不想创建Lazy类的实例。可调用System.Threading.LazyInitializer类的静态方法
```
public static class LazyInitializer {
    // 这两个方法内部调用了Interlocked.CompareExchange
    public static T EnsureInitialized<T>(ref T target) where T : class;
    public static T EnsureInitialized<T>(ref T target, Func<T> valueFactory) where T : class;

    // 这两个方法在内部将同步锁(syncLock)传递给Monitor的Enter和Exit方法
    public static T EnsureInitialized<T>(ref T target, ref Boolean initialized, ref Object syncLock);
    public static T EnsureInitialized<T>(ref T target, ref Boolean initialized, ref Object syncLock, Func<T> valueFactory);
}
```
为EnsureInitialized方法的syncLock参数显式指定同步对象，可以用同一个锁保护多个初始化函数和字段  
```
public static void Main() {
    String name = null;
    LazyInitializer.EnsureInitialized(ref name, () => "hahha");
    Console.WriteLine(name); // hahha

    LazyInitializer.ENsureInitialized(ref name, () => "233");
    Console.WriteLine(name); // hahha
}
```

#### 5.条件变量模式
假定一个线程希望在一个符合条件为true时执行一些代码，可通过条件变量模式实现。
```
public static class Monitor {
    public static Boolean Wait(Object obj);
    public static Boolean Wait(Object obj, Int32 millisecondsTimeout);

    public static void Pulse(Object obj);
    public static void PulseAll(Object obj);
}
```

```
internal sealed class ConditionVariablePattern {
    private readonly Object m_lock = new Object();
    private Boolean m_condition = false;

    public void Thread1() {
        Monitor.Enter(m_lock);
        while(!m_condition) {
            // 条件不满足，等待另一个线程更改条件
            Monitor.Wait(m_lock); // 临时释放锁，使其他线程能获取它
        }

        // 条件满足，处理数据
        Monitor.Exit(m_lock);
    }

    public void Thread2() {
        Monitor.Enter(m_lock);

        m_condition = true;

        //Monitor.Pulse(m_lock); // 锁释放后唤醒一个等待的线程
        Monitor.PulseAll(m_lock);

        Monitor.Exit(m_lock);
    }
}
```
Pulse只解除等待最久的线程的阻塞。  

下面展示一个线程安全的队列，允许多个线程在其中对数据项进行入队和出队操作。注意，除非有一个可供处理的数据线，否则出队一个数据项的线程会移至阻塞。  
```
internal sealed class SynchronizedQueue<T> {
    private readonly Object m_lock = new Object();
    private readonly Queue<T> m_queue = new Queue<T>();

    public void Enqueue(T item) {
        Monitor.Enter(m_lock);
        m_queue.Enqueue(item);
        Monitor.PulseAll(m_lock);

        Monitor.Exit(m_lock);
    }

    public T Dequeue() {
        Monitor.Enter(m_lock);
        while (m_queue.Count == 0) {
            Monitor.Wait(m_lock);
        }
        T item = m_queue.Dequeue();
        Monitor.Exit(m_lock);
        return item;
    }
}
```
#### 6.异步的同步构造
和本章展示的大量构造相比，任务具有下述许多优势  
1. 任务使用的内存比线程少得多，创建和销毁所需要的时间也少得多  
2. 线程池根据可用CPU数量自动伸缩任务规模  
3. 每个任务完成一个阶段后，运行任务的线程回到线程池，在那里接受新任务  
4. 线程池是站在整个进程的高度观察任务。所以它能更好的调度这些任务，减少进程中的线程数，并减少上下文切换  

#### 7.并发集合类
FCL自带4个线程安全的集合类，全部在System.Collections.Concurrent命名空间定义  
1. ConcurrentQueue  
2. ConcurrentStack  
3. ConcurrentDictionary  
4. ConcurrentBag  

所有这些集合类都是"非阻塞"的。如果一个线程试图提取一个不存在的元素，线程会立即返回。  
所有并发集合类都提供了GetEnumerator方法。
ConcurrentStack、ConcurrentQueue和ConcurrentBag类的GetEnumerator获取集合内容的一个"快照"，并从这个快照中返回元素。  
而ConcurrentDictionary不获取快照(枚举字典期间有可能会被变化)  

ConcurrentStack、ConcurrentQueue和ConcurrentBag类实现了IProducerConsumerCollection接口  
```
public interface IProducerConsumerCollection<T> : IEnumerable<T>, ICollection, IEnumerable {
    Boolean TryAdd(T item);
    Boolean TryTake(out T item);
    T[] ToArray();
    void CopyTo(T[] array, int32 index);
}
```
实现了这个接口的任何类都能转变成一个阻塞集合。如果集合已满，负责生产的线程会阻塞。反之集合已空，负责消费的线程会阻塞。  

要将非阻塞的集合转变成阻塞集合，需要构造一个System.Collections.Current.BlockingCollection类，并向构造器传递非阻塞集合的引用  
```
public class BlockingCollection<T> : IEnumerable<T>, ICollection, IEnumerable, IDisposable {
    public BlockingCollection(IProducerConsumerCollection<T> collection, Int32 boundedCapacity);

    public void Add(T item);
    public Boolean TryAdd(T item, int32 msTimeout, CancellationToken cancellationToken);
    public void CompleteAdding();

    public T Take();
    public Boolean TryTake(out T item, Int32 msTimeout, CancellationToken cancellationToken);

    public Int32 BoundedCapacity { get; }
    public Int32 Count { get; }
    public Boolean IsAddingCompleted { get; } // 如果调用了CompleteAdding,则为true

    public IEnumerable<T> GetComsumingEnumerable(CancellationToken cancellationToken);

    public void CopyTo(T[] array, int index);
    public T[] ToArray();
    public void Dispose();
}
```

下例演示如何设置一个生产者/消费者环境，以及如何在完成数据线的添加之后发出通知  
```
public static void Main() {
    var b1 = new BlockingCollection<Int32>(new ConcurrentQueue<Int32>());

    // 由一个线程池线程执行"消费"
    ThreadPool.QueueUserWorkItem(ConsumeItems, bl);
    for (Int32 item = 0; item < 5; item++) {
        Console.WriteLine("Producing:" + item);
        bl.Add(item);
    }

    // 告诉消费者线程不会在集合中添加更多的项了
    bl.CompleteAdding();

    Console.ReadLine();
}

private static void ConsumeItems(Object o) {
    var bl = (BlockingCollection<Int32>)o;
    // 阻塞，知道出现数据线，出现就处理它
    foreach(var item in bl.GetConsumingEnuerable()) {
        Console.WriteLine("Consuming " + item);
    }
    
    Console.WriteLine("All items have been consumed");
}

```
BlockingCollection类还提供了静态AddToAny、TryAddToArray、TakeFromAny和TryTakeFromAny方法。所有这些方法都获取一个BlockingCollection<T>[],以及一个数据项、一个超时值以及一个CancellationToken。

