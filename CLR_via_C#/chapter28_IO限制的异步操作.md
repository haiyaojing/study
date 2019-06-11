#### 1.CLR线程池基础  
CLR包含了代码来管理它的**线程池**(thread pool)  
每CLR一个线程池:这个线程池由CLR控制的所有AppDomain共享。如果一个进程中加载了多个CLR，那么每个CLR都有它的线程池。  

应用程序向线程池发出需求多请求，线程池会尝试只用这一个线程来服务所有请求，直到请求速度超过了线程池线程处理它们的速度，就会额外创建线程。  

ThreadPool类  
```
static Boolean QueueUserWorkItem(WaitCallback callback);
static Boolean QueueUserWorkItem(WwaitCallback callback, Object state);
```
这些方法向线程池的队列添加一个工作项以及可选的状态数据，方法会立即返回。  
```
delegate void WaitCallback(Object state);
```

#### 2.执行上下文  
每个线程都关联了一个执行上下文数据结构  
1. 安全设置(压缩栈、Thread的Principal属性和Windows身份)
2. 宿主设置(System.Threading.HostExecutionContextManager)
3. 逻辑调用上下文数据(System.Runtime.Remoting.Messaging.CallContext的LogicalSetData和LogicalGetData方法)

每当一个线程(初始线程)使用另一个线程(辅助线程)时，前者的执行上下文应该流向(复制到)辅助线程。这样确保了辅助线程执行的任何操作使用的都是相同的安全设置和宿主设置。还确保了在初始线程的逻辑调用上下文中存储的任何数据都适用于辅助线程。  

默认情况下,CLR自动执行前者，将造成上下文信息(大量信息)传给辅助线程，对性能造成影响。  

System.Threading.ExecutionContext控制线程的执行上下文如何流向  
```
public sealed class ExecutionContext : IDisposable, ISerializable {
    [SecurityCritical] 
    public static AsyncFlowControl SuppressFlow();
    public static void RestoreFlow();
    public static Boolean IsFlowSuppressed();
}
```
例子  
```
public static void Main() {
    // 将一些数据放到Main线程的逻辑调用上下文中
    CallContext.LogicalSetData("Name", "haiyaojing");
    // 初始化要由一个线程池线程中的一些工作
    // 线程池线程能访问逻辑调用上下文数据
    ThreadPool.QueueUserWorkItem(state => Console.WriteLine($"Name={CallContext.LogicalGetData("name")}"));
    // 阻止Main线程的执行上下文流动
    ExecutationContext.SuppressFlow();
    // 初始化要由线程池线程做的工作
    // 线程池线程不能访问逻辑调用上下文数据
    ThreadPool.QueueUserWorkItem(state=>Console.WriteLine($"Name={CallContext.LogicalGetData("name")}"));
    // 恢复
    ExecutationContext.RestoreFlow();
}
```
输出:  
Name=haiyaojing  
Name=  

#### 3.协作式取消和超时  
System.Threading.CancellationTokenSource  
```
public sealed class CancellationTokenSource : IDisposable {
    public CancellationTokenSource();
    public void Dispose(); // 释放资源(比如WaitHandle)

    public Boolean IsCancellationRequested { get;}
    public CancellationToken token { get; }
    
    public void Cancel(); // 内部调用Cancel并传递false
    public void Cancel(Boolean throwOnFirstException);
}

public struct CancellationToken {
    public static CancellationToken None { get; }

    public Boolean IsCancellationRequested { get; } // 由非通过Task调用的操作调用
    public void ThrowIfCancellationRequested(); // 由通过Task调用的操作调用

    // CancellationTokenSource取消时，WaitHandle会受到信号
    public WaitHandle WaitHandle { get; }

    public Boolean CanBeCanceled { get; }

    // 在取消一个CancellationTokenSource时调用的方法
    public CancellationTokenRegisteration Register(Action<Object> callback, Object state, Boolean useSynchronizationContext);
}

```

```
var cts = new CancellationTokenSource();
cts.Token.Register(() => Console.WriteLine("Cancel1"));
cts.Token.Register(() => Console.WriteLine("Cancel2"));

cts.Cancel();
```
以上代码输出:  
Cancel1  
Cancel2  

还可以通过连接另一组CancellationTokenSource来新建一个CancellationTokenSource对象，任何一个链接的CancellationTokenSource被取消，这个新的CancellationTokenSource对象就会被取消  
```
var cts1 = new CancellationTokenSource();
cts1.Token.Register(() => Console.WriteLine("Cancel1"));

var cts2 = new CancellationTokenSource();
cts2.Token.Register(() => Console.WriteLine("Cancel2"));

var linedCts = CancellationTokenSource.CreateLinkedTokenSource(cts1.Token, cts2.Token);
linkedCts.Token.Register(() => Console.WriteLine("LinkedCts Canceled"));

cts2.Cancel();

// 显示哪个被取消了
Console.WriteLine($"{cts1.IsCancellationRequested} {cts2.IsCancellationRequested} {linedCts.IsCancellationRequested}");

```
输出  
LinkedCts Canceled  
canceled  
false true true  

过一段时间后取消操作  
```
public sealed class CancellationTokenSource : IDisposable {
    public CancellationTokenSource(Int32 millisecondsDelay);
    public CancellationTokenSource(TimeSpan delay);
    public void CancelAfter(Int32 millisecondsDelay);
    public void CancelAfter(TimeSpan delay);
}
```

#### 4.任务  
以下写法做相同的事情  
```
ThreadPool.QueueUserWorkItem(ComputeBoundOp, 5);
new Task(ComputeBoundOp, 5).Start();
Task.Run(() => ComputeBoundOp(5););
```
向构造器传递TaskCreationOptions标志来控制Task的执行方式  
```
[Flags, Serializable]
public enum TaskCreationOptions {
    None                = 0x00, //默认
    // 提议TaskScheduler你希望该任务尽快执行
    PreferFairness      = 0x0001,
    // ...应尽可能地创建线程池线程
    LongRunning         = 0x0002,
    // 该提议总是被采纳:将一个Task和它的父Task关联
    AttachedToParent    = 0x0004,
    // 该提议总是被采纳:如果一个任务试图和这个父任务连接，它就是一个普通任务，而不是子任务
    DenyChildAttach     = 0x0008,
    // 该提议总是被采纳:强迫子任务使用默认调度器而不是父任务的调度器
    HideScheduler       = 0x0010
}
```
有的标志指示"提议"  
后几个总是得以采纳是因为它们和TaskScheduler本身无关  

例子  
```
private static Int32 Sum(Int32 n) {
    Int32 sum = 0;
    for (; n > 0; sum += n, n--);
    return sum;
}

var t = new Task<Int32>(n => Sum(123), 2);
t.Start();
t.Wait(); // 显式等待任务完成
int k = t.Result; // 内部调用Wait()
```
如果计算限制的任务抛出未处理的异常，异常会被"吞噬"并存储到一个结合中，而线程池线程可以返回到线程池中。  
AggregateException  

Task的静态WaitAny方法会注射调用线程，知道数组中的任何Task对象完成。方法返回Int32数组索引值，指明完成的是哪个Task对象。方法返回后，线程被唤醒并继续运行。  
如果发生超时，方法将返回-1    
如果WaitAny通过一个CancellationToken取消，会抛出一个OperationCanceledException  

Task的静态WaitAll方法。如果所有Task对象完成返回true。发生超时返回false。...  
***
**取消任务**  
```
private static Int32 Sum(CancellationToken ct, Int32 n) {
    Int32 sum = 0;
    for(; n > 0; n--) {
        ct.ThrowIfCancellationRequested();
        checked { sum += n; }
    }
    return sum;
}
```
取消后调用ThrowIfCancellationRequested方法会抛出OperationCanceledException异常  


```
CancellationTokenSource cts = new CancellationTokenSource();
Task<Int32> t = Task.Run(() => Sum(cts.Token, 10000), cts.Token);
cts.Cancel();
try {
    Console.WriteLine($"the sum is: {t.Result}");
} catch (AggregateException x) {
    x.Handle(e => e is OperationCanceledException);
    Console.WriteLine("Sum was canceled");
}
```
在Task调度前执行Cancel，任务永远都不会执行  
***
**任务完成时自动启动新任务**  
通过Wait方法可以等待任务完成，但极有可能造成线程池创建新线程，这增大了资源的消耗也不利于性能和伸缩性  
ContinueWith  
```
Task<Int32> t = Task.Run(() => Sum(CancellationToken.None, 10000));
Task cwt = t.ContinueWith(task => Console.WriteLine($"the sum is : {task.Result}"));
```
ContinueWith返回对新Task对象的引用。一般都忽略这个Task对象  

可在调用ContinueWith时传递一组TaskContinuationOptions枚举值(前六个和前面TaskCreationOptions一样)  
```
[Flags, Serializable]
public enum TaskContinuationOptions {
    None                = 0x00, //默认
    // 提议TaskScheduler你希望该任务尽快执行
    PreferFairness      = 0x0001,
    // ...应尽可能地创建线程池线程
    LongRunning         = 0x0002,
    // 该提议总是被采纳:将一个Task和它的父Task关联
    AttachedToParent    = 0x0004,
    // 该提议总是被采纳:如果一个任务试图和这个父任务连接，它就是一个普通任务，而不是子任务
    DenyChildAttach     = 0x0008,
    // 该提议总是被采纳:强迫子任务使用默认调度器而不是父任务的调度器
    HideScheduler       = 0x0010,
    // 除非前置任务完成，否则禁止延续任务完成(取消)
    LazyCancellation    = 0x0020,
    // 这个标志指出你希望由第一个任务的线程执行
    // ContinueWith任务，第一个任务完成后，调用
    // ContinueWith的线程接着执行ContinueWith任务
    ExecuteSynchronously = 0x8000,

    // 这些标志指出在什么情况下运行ContinueWith任务
    NotOnRanToCompletion = 0x10000,
    NotOnFaulted         = 0x20000,
    NotOnCanceled        = 0x40000,
    
    // 便利组合
    OnlyOnCanceld        = NotOnRanToCompletion | NotOnFaulted, // 新任务只有在第一个任务被取消时执行
    OnlyOnFaulted        = NotOnRanToCompletion | NotOnCanceled, // 第一个任务抛出未处理的异常时执行
    OnlyOnRanToCompletion = NotOnFaulted | NotOnCanceled, // 只有第一个任务顺利完成(中间没有取消，没有未处理的异常时执行)
}
```
一个Task完成时，它的所有未运行的延续任务都会被自动取消  
***
**启动子任务**  
```
Task<Int32[]> parent = new Task<Int32[]>(() => {
    var results = new Int32[3]; // 创建一个数组来存储结果
    // 这个任务创建并启动3个子任务
    new Task(() => result[0] = Sum(10000), TaskCreationOptions.AttachedToParent).Start();
    new Task(() => result[1] = Sum(10000), TaskCreationOptions.AttachedToParent).Start();
    new Task(() => result[2] = Sum(10000), TaskCreationOptions.AttachedToParent).Start();
    return results;
})

// 父任务及其子任务运行完成后，用一个延续任务显示结果
var cwt = parent.ContinueWith(parentTask => Array.ForEach(parentTask.Result, Console.WriteLine));
// 启动父任务便于启动它的子任务
parent.Start();
```
***
**任务内部揭秘**  
每个Task对象都包括一组字段  
1. Int32 ID
2. 代表Task执行状态的一个Int32
3. 对父任务的引用
4. 对Task创建时指定的TaskScheduler的引用
5. 对回调方法的引用
6. 对要传给回调方法的对象的引用(可通过Task的只读属性AsyncState查询)
7. 对ExecutationContext的引用
8. 对manualResetEventSlim对象的引用

另外每个Task对象都有对根据需要创建的补充状态的引用，包括  
1. CancellationToken
2. ContinueWith对象集合
3. 为抛出未处理异常的子任务而准备的一个Task对象集合

如果不需要任务的附加功能，建议使用ThreadPool.QueueUserWorkItem能获得更好的资源利用率  

Task的只读Status属性可了解它在其生命期的什么位置  
```
public enum TaskStatus {
    Created,                // 任务已显示创建，可以手动Start
    WaitingForActivation,   // 任务已隐式创建，会自动开始
    WaitingToRun,           // 任务已调度，但尚未运行
    Running,
    WaitingForChildrenToComplete,

    //以下是最终状态
    RanToCompletion,
    Canceled,
    Faulted,
}

```
***
**任务工厂**  
System.Threading.Tasks命名空间定义了TaskFactory和一个TaskFactory<TResult>类型
要创建一组返回void的任务，就构造一个TaskFactory；要创建一组具有返回特定返回类型的任务，就构造一个TaskFactory<TResult>，并通过泛型TResult实参传递任务的返回类型  

创建以上任何工厂类时，要向构造器传递工厂创建的所有任务都具有的默认值。具体的说，要向任务工厂传递希望任务具有的CancellationToken、TaskScheduler、TaskCreationOptions和TaskContinuationOptions设置  

```
Task parent = new Task(() => {
    var cts = new CancellationTokenSource();
    var tf = new TaskFactory<Int32>(
        cts.Token,
        TaskCreationOptions.AttachedToParent,
        TaskContinuationOptions,ExecuteSynchronously,
        TaskScheduler.Default);
    // 这个任务创建并启动三个子任务
    var childTasks = new[] {
        tf.StartNew(() => Sum(cts.Token, 10000)),
        tf.StartNew(() => Sum(cts.Token, 20000)),
        tf.StartNew(() => Sum(cts.Token, Int32.MaxValue)),
    };

    // 任何子任务抛出异常就取消其余子任务
    for(Int32 task = 0; task < childTask.Length; task++) {
        childTasksptask.ContinueWith(
            t => cts.Cancel(), TaskContinuationOptions.OnlyOnFaulted);
    }
    tf.ContinueWhenAll(
        childTask,
        completedTasks => completedTasks.Where(
            t => !t.IsFaulted && !t.IsCanceled).Max(t => t.Result),
        CancellationToken.None)
            .ContinueWith(t => Console.WriteLine($"the maxinum is {t.Result})),
                TaskContinuationOptions.ExecuteSynchronously);
    )
});

// 子任务完成后，也显示任何未处理的异常
parent.ContinueWith(p => {
    ...
}, TaskContinuationOptions.OnlyOnFaulted);
parent.Start();
```
***
**任务调度器**  
TaskScheduler对象负责执行被调度的任务，同时向VS调试器公开任务信息。FCL提供了两个派生自TaskScheduler的类型:线程池任务调度器(默认)和同步上下文任务调度器。  

通过TaskScheduler的静态Default属性获取对默认任务调度器的引用  

通过TaskScheduler的静态FromCurrentSynchronizationContext方法来获得对同步上下文任务调度器的引用  

```
class MyForm : Form {
    private TaskScheduler m_syncContextTaskScheduler;
    public MyForm() {
        m_syncContextTaskScheduler = TaskScheduler.FromCurrentSynchronizationContext();

        Text = "Start";
        Visible = true;
        Width = 800;
        heiht = 100;
    }

    private CancellationTokenSource m_cts;
    protected override void OnMouseClick(MouseEventArgs e) {
        if (m_cts != null) {
            m_cts.Cancel();
            m_cts = null;
        } else {
            Text = "Running";
            m_cts = new CancellationTokenSource();
            // 这个任务使用默认任务调度器,在一个线程池中执行
            Task<Int32> t = task.Run(() => Sum(m_cts.Token, 10000), m_cts.Token);

            // 这些任务使用同步上下文任务调度器，在GUI线程上执行
            t.ContinueWith(task => Text = "Result="  + task.Result, CancellationToken.None, TaskContinuationOptions.OnlyOnRanToCompletation, m_syncContextTaskScheduler);

            t.ContinueWith(task => Text = "Operation Canceled", CancellationToken.None, TaskContinuationOptions, OnlyOnCanceled,m_syncContextTaskScheduler);
        }
        base.OnMouseClick(e);
    }
}
```
线程池线程不能更新UI组件，否则会抛出InvalidOperationException  
GUI线程则可以  

定义自己的TaskScheduler派生类，以下是Microsoft提供的  
1. IOTaskScheduler
    这个任务调度器将任务排队给线程池的IO线程而不是工作者线程  
2. LimitedConcurrencyLevelTaskScheduler
    这个任务调度器不允许超过n(一个构造器参数)个任务同时执行  
3. OrderedTaskScheduler
    这个任务调度器一次只允许一个任务执行  
4. PrioritizingTaskScheduler
    这个任务调度器将任务送入CLR线程池队列。之后可调用Prioritize指出一个Task应该在所有的任务之前处理。可以调用Deprioritize使一个Task在所有普通任务之后处理  
5. ThreadPerTaskScheduler
    这个任务调度器为每个任务创建并启动一个单独的线程；不使用线程池  

#### 5.Parallel的静态For、ForEach和Invoke方法  
```
Parallel.For(0, 1000, i => DoWork(i));
Parallel.ForEach(collection, item => DoWork(item));
```
要执行多个方法，可以顺序执行  
```
Method1();
Method2();
Method3();
```
并行执行  
```
Parallel.Invoke(
    () => Method1(),
    () => Method2(),
    () => Method3(),
)
```
**Parallel的所有方法都让调用线程参与处理**  

Parallel的For、ForEach和Invoke方法都提供了接受一个ParallelOptions对象的重载版本  
```
public class ParallelOptions {
    public ParallelOptions();

    // 允许取消操作
    public CancellationToken CancellationToken { get; set; } // 默认为CancellationToken.None

    // 允许执行可以并发操作的最大工作项数目
    public Int32 MaxDegreeOfParallelism { get; set; } // 默认为-1（可用CPU数)

    // 允许指定要使用哪个TaskScheduler
    public TaskScheduler TaskScheduler { get; set; } // 默认为TaskScheduler.Default
}
```
For、ForEach方法有一些重载版本允许场地3个委托  
1. 任务局部初始化委托(localInit),为参与工作的每个任务都调用一次该委托。处理工作项之前
2. 主体委托(body)，为参与工作的各个线程说出你的每一项都调用一次委托
3. 任务局部终结委托(localFinally)，为参与工作的每个任务都调用一次该委托。处理工作项之后，即使主体发生异常也会调用

#### 6.秉性语言继承查询(PLINQ)
使用LINQ to Objects时，只有一个线程顺序处理数据集合中的所有项，称为**顺序查询**
要提高处理性能可以使用并行LINQ(Parallel LINQ),它将顺序查询转换成并行查询，在内部使用任务处理  
```
public static ParallelQuery<TSource>  AsParallel<TSource>(this IEnumerable<TSource> source);
public static ParallelQuery           AsParallel(this IEnumerable source);
```

```
var query = Assembly.GetEntryAssembly().GetExportedTypes().AsParallel();
foreach (var result in query) Console.WriteLine(result);
query.ForAll(t => Console.WriteLine(t)); // 调用的方法如果每次只能一个线程访问，反而会出现并行问题...
```
并行操作切换回顺序操作  
```
public static IEnumerable<TSource> AsSequential<TSource>(this ParallelQuery<TSource> source);
```
PLINQ分析一个查询，然后决定如何最好地处理它。使用Select(Many)或Where的重载版本，并向selector或predicate委托传递一个位置索引时也是如此  

通过调用WithExecutionMode，向它传递某个ParallelExecutionMode标志，从而强迫查询以并行方式处理:  
```
public enum ParallelExecutionMode {
    Default = 0, // 让并行LINQ决定处理查询的最佳方式
    ForceParallelism = 1 // 强迫查询以其并行方式处理
}
```
并行LINQ让多个线程处理数据线，结果必须再合回去。可调用WithMergeOptions，向它传递以下某个ParallelMergeOptions标志，从而控制这些结果的缓冲与合并方式  
```
public enum ParallelMergeOptions {
    Default = 0,            // 目前和AutoBuffered一样
    NotBuffered = 1,        // 结果一旦就绪就开始处理
    AutoBuffered = 2,       // 每个线程在处理前缓冲一些结果
    FullyBuffered = 3       // 每个线程在处理前缓冲所有结果
}
```
这些选项能在某种程度上平衡执行速度和内存消耗。  
NotBuffered最省内存，但处理速度慢一些  
FullyBuffered消耗较多的内存，但运行得最快  
AutoBuffered介于NotBuffered和FullyBuffered之间  

#### 7.执行定时计算限制操作  
System.Threading.Timer让一个线程池线程定时调用一个方法  
```
delegate void TimerCallback(Object state);

public sealed class Timer : MarshalByRefObject, IDisposable {
    public Timer(TimerCallback callback, Object state, Int32 dueTime, Int32 period);
    ...
}
```
构造器的state参数允许在每次调用回调方法时都向它传递状态数据;没有则可以传递null  
dueTime告诉CLR在首次调用回调方法之前要等待多少毫秒(0代表马上调用)  
period制定了以后每次调用回调方法之前要等待多少毫秒(Timeout.Infinite(-1)代表只调用一次)  

如果回调方法的执行时间很长，计时器可能在三个回调还没完成时再次触发。解决方法:  
构造Timer时，为period指定Timeout.Infinite，在回调方法中，调用Change方法来指定一个新的dueTime,并再次指定period为Timeout.Infinite  
```
public sealed class Timer : MarshalByRefObject, IDisposable {
    public Boolean Change(Int32 dueTime, Int32 period);
    ...
}
```

Timer还提供了一个Dispose方法，允许完全取消计时器，并可在当时处于pending状态的所有回调完成之后，向notifyObject参数标识的内核对象发出信号  
```
public sealed class Timer : MarshalByRefObject, IDisposable {
    public Boolean Dispose();
    public Boolean Dispose(WaitHandle notifyObject);
}
```

如果需要定时执行的操作，可利用Task的静态Delay方法和C#的async和await关键字来编码  
```
internal static class DelayDemo {
    public static void Main() {
        Console.WriteLine("Checking status every 2 seconds);
        Status();
        Console.ReadLine(); // 防止进程终止
    }

    private static async void Status() {
        while(true) {
            Console.WriteLine($"Checking status at {DateTime.Now}");

            // 在循环末尾，在不阻塞线程的前提下延迟2秒
            await Task.Delay(2000); // await允许线程返回
            // 2秒后，某个线程会在await之后介入并继续循环
        }
    }
}
```
计时器  
1. System.Threading.Timer
    要在一个线程池线程上执行定时的后台任务，这是最好的计时器  
2. System.Windows.Forms.Timer
    构造这个类的实例，相当于告诉Windows将一个计时器和调用线程关联(参看Win32 SetTimer函数)。当这个计时器触发时，Windows将一条计时器消息(WN_TIMER)注入线程的消息队列。线程必须执行一个消息泵来提取这些消息。注意这些工作都只由一个县城完成——设置计时器的线程保证就是执行回调方法的线程。而且不会被多个线程并发执行  
3. System.Windows.Threading.DispatcherTimer
    是System.Windows.Forms.Timer类在Silverlight和WPF应用程序中的等价物  
4. Windows.UI.Xaml.DispatcherTimer
    是System.Windows.Forms.Timer类在Windows Store引用中的等价物  
5. System.Timers.Timer
    是System.Threading.Timer类的包装内。计时器到期会导致CLR将事件放到线程池队列中.  

#### 8.线程池管理线程  
**设置线程池限制**  
设置线程池拥有的线程数量  

**管理工作者线程**  
ThreadPool.QueueUserWorkItem方法和Timer类总是将工作项放到全局队列中。  
工作者线程采用FIFO算法将工作项从这个队列取出(所有工作者线程都竞争一个线程同步锁，可能对伸缩性和性能造成某种程度的限制)  

使用默认TaskScheduler来调度Task对象:  
非工作者线程调度一个Task时，该Task被添加到全局队列。但每个工作者线程都有自己的本地队列。工作者线程调度一个Task时，该Task被添加到调用线程的本地队列。  

工作者线程准备好处理工作项时，它总是先检查本地队列来查找一个Task。存在一个Task，工作者线程就从本地队列移出Task并处理工作项(FIFO算法)  

由于工作者线程是唯一允许访问它自己的本地队列头的线程，所以无需同步锁  
![27_01](https://note.youdao.com/yws/api/personal/file/721DA7FA53CE4269B7AD1C37F348DB9F?method=download&shareKey=ae579f1cfdcd25b60d294de1150b1629)  

工作者线程本地队列空了，会尝试从另一个工作线程的本地队列"偷"一个Task。从尾部"偷走"，并要求获取一个线程同步锁。    
如果本地队列都为空，那么工作者线程会使用FIFO算法从全局队列提取一个工作项(取得它的锁)。如果全局队列也为空，工作者线程会进入睡眠状态，等待事情的发生。如果睡眠时间太长，它会自己醒来并销毁自身，允许系统回收线程使用的资源(内核对象、栈、TEB等)  

线程池会快速创建工作者线程，使工作者线程的数量等于传给ThreadPool的SetMinThreads方法的值(从不调用那么就默认为CPU数量)  