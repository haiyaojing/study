#### 1.Windows如何执行IO操作  
程序构造一个FileStream对象来打开磁盘文件，然后调用Read方法从文件中读取数据。  
调用FileStream的Read方法时，线程从托管代码转变为本机/用户模式代码，Read内部调用Win32 ReadFile函数。  
ReadFile分配一个小的数据结构，称为IO请求包(IO Request Packet, IRP)，一个Byte[]数组的地址(数组用读取的字节来填充)，要传输的字节数以及其他常规性内容。
![28-01](https://note.youdao.com/yws/api/personal/file/0BC188BA85DF4CE190460B1B62C214C9?method=download&shareKey=b23941756763ad62a936f45980fe3301)  
然后，ReadFile将线程从本机/用户模式代码转变为本机/内核模式代码，向内核传递IRP数据结构，从而调用Windows内核。  
根据IRP中的设备句柄，Windows内核知道IO操作要传送给哪个硬件设备。因此Windows将IRP传送给恰当的设备驱动程序的IRP队列。  
每个设备都维护着自己的IRP队列，其中包含了机器上运行的所有进程发出的IO请求。  
IRP数据包到达时，设备驱动程序将IRP信息传给物理硬件设备上安装的电路板。  
硬件设备执行请求的IO操作。  

**在硬件设备执行IO操作期间，发出了IO请求的线程会被Windows设置为睡眠状态**

当硬件完成IO操作后，Windows会唤醒线程，将它调度给一个CPU，使它从内核模式返回用户模式，再返回托管代码。  
FileStream的Read方法返回一个Int32指明从文件中读取的实际字节数，因此可以知道在传给Read的Byte[]中实际能检索到多少个字节   

**Windows异步执行IO操作**
![28-02](https://note.youdao.com/yws/api/personal/file/403AD11CDD3D4661BF8CF9B354A36C8E?method=download&shareKey=3f7da2dc4163efdfcaedce348fb69978)  
调用ReadAsync返回的一个Task<Int32>对象，可在该对象上调用ContinueWith来登记任务完成时执行的回调方法，在回调方法中处理数据。也可以利用异步函数来简化编码  

在内部，CLR的线程池使用名为"IO完成端口"(IO Completion Port)的Windows资源来描述刚才的行为。CLR在初始化时创建一个IO完成端口。当打开硬件设备时，这些设备可以和IO完成端口关联，使设备驱动程序知道将完成的IRP送到哪里。  

#### 2.C#的异步函数  
```
private static async Task<String> IssueClientRequestAsync(String serverName, String message) {
    using(var pipe = new NamedPipeClientStream(serverName, "PipeName", PipeDirection.InOut, PipeOptions.Asynchronous | PipeOptions.WriteThrough)) {
        pipe.Connect();
        pipe.ReadMode = PipeTransmissionMode.Message;

        Byte[] request = Encoding.UTF8.GetBytes(message);
        await pipe.WriteAsync(request, 0, request.Length);
        ...
    }
}
```
一旦将方法标记为async，编译器就会将方法的代码转换成实现了状态机的一个类型。这就允许线程执行状态机中的一些代码并返回而不需要一直执行到结束  
C#的await操作符实际会在Task对象上调用ContinueWith，向它传递用于恢复状态机的方法。  

异步函数的限制  
1. 不能将应用程序的Main方法转变成异步函数。构造器、属性访问器方法和时间访问器方法都不能转变成异步函数。  
2. 异步函数不能使用任何out或ref参数  
3. 不能再catch，finally或unsafe块中使用await操作符  
4. 不能再await操作符之前获得一个支持线程所有权或递归的锁，并在await操作符之后释放他。因为await前后代码可能不是同一个线程执行。  
5. 在查询表达式中，await操作符只能在初始from子句的第一个集合表达式中使用，或者在join子句的集合表达式中使用  

#### 3.编译器将异步函数转换成状态机

首先定义一些简单的类型和方法  
```
internal sealed class Type1 {}
internal sealed class Type2 {}
private static async Task<Type1> Method1Async() {
    /* 以异步方式执行一些操作，最后返回一个Type1对象 */
}
private static async Task<Type2> Method2Async() {
    /* 以异步方式执行一些操作，最后返回一个Type2对象 */
}

private static async Task<String> MyMethodAsync(Int32 argument) {
    Int32 local = argument;
    try {
        Type1 result1 = await Method1Async();
        for(Int32 x = 0; x < 3; x++) {
            Type2 result2 = await Method2Async();
        }
    }
    catch (Exception e) {
        Console.WriteLine("Catch Exception");
    }
    finally {
        Console.WriteLine("Finally");
    }
    return "Done";
}

```
编译器将方法中的代码转换成一个状态机结构。这种结构能挂起和恢复。  

```
// AsyncStateMachine特性支出这是一个异步方法(对使用反射的工具有用)
// 类型指出实现状态机的是哪个结构
[DebuggerStepThrough, AsyncStateMachine(typeof(StateMachine))]
private static Task<String> MyMethodAsync(Int32 argument) {
    // 创建状态机实例并初始化它
    StateMachine stateMachine = new StateMachine() {
        // 创建builder,从这个存根方法返回Task<String>
        // 状态机访问builder来设置Task完成/异常
        m_builder = AsyncTaskMethodBuilder<String>.Create(),

        m_state = -1,      // 初始化状态机位置
        m_argument = argument, // 将实参拷贝到状态机字段
    }

    //开始执行状态机
    stateMachine.m_builder.Start(ref stateMachine);
    return stateMachine.m_builder.Task; // 返回状态机的Task
}

// 这是状态机结构
[CompilerGenerated, StructLayout[LayoutKind.Auto]]
private struct StateMachine : IAsyncStateMachine {
    // 代表状态机builder(Task)及其位置的字段
    public AsyncTaskMethodBuilder<String> m_builder;
    public Int32 m_state;

    // 实参和局部变量现在变成了字段
    public Int32 m_argument, m_localm m_x;
    public Type1 m_resultType1;
    public Type2 m_resultType2;

    // 每个awaiter类型一个字段
    // 任何时候这些字段只有一个是最重要的，那个字段引用最近执行的、以异步方法完成的await

    // 这是状态机方法本身
    void IAsyncStateMachine.MoveNext() {
        String result = null; // Task的结果值
        try {
            Boolean executeFinally = true;  // 先假定逻辑离开try块
            if (m_state == -1) {            // 如果第一次在状态机方法中
                m_local = m_argument;       // 原始方法就从头开始执行
            }

            // 原始代码中的try块
            try {
                TaskAwaiter<Type1> awaiterType1;
                TaskAwaiter<Type2> awaiterType2;

                switch(m_state) {
                    case -1 : // 开始执行try块中的代码
                        // 调用MethodAsync并获得它的awaiter
                        awaiterType1 = Method1Async().GetAwaiter();
                        if (!awaiterType1.IsCompleted) {
                            m_state = 0; // Method1Async要以异步方式完成
                            m_awaiterType1 = awaiterType1; // 保存awaiter以便将来来返回

                            // 告诉awaiter在操作完成时调用MoveNext
                            m_builder.AwaitUnsafeOnCompleted(ref awaiterType1, ref this);
                            // 上述代码调用awaiterType1的OnCompleted，他会在被等待的任务上
                            // 调用ContinueWith(t => MoveNext())...
                            // 任务完成后，continueWith任务调用MoveNext

                            executeFinally = false; // 逻辑不离开try块
                            return; // 线程返回至调用者
                        }
                        // Method1Async 以同步方式完成了
                        break;
                    case 0 : // Method1Async以异步方式完成了
                        awaiterType1 = m_awaiterType1; // 恢复最新的awaiter
                        brea;
                    case 1 : // Method2Async 以异步方式完成了
                        awaiterType2 = m_awaiterType2; // 恢复最新的awaiter
                        goto ForLoopEplilog;
                }
                ForLoopPrologue:
                    m_x = 0; // for循环初始化
                    goto ForLoopBody; // 跳到for循环主体
                ForLoopEpilog:
                    m_resultType2 = awaiterType2.GetResult();
                    m_x++;
                    // ↓↓直通for循环主体
                ForLoopBody:
                    if (m_x < 3>) {
                        ...
                    }
            }
            catch (Exception) {
                Console.WriteLine("catch exception");
            }
            finally {
                if (executeFinally) {
                    Console.WriteLine("Finally");
                }
            }
        }
        catch(Exception e) {
            // 未处理的异常，通过设置异常来完成状态机的Task
            m_builder.SetException(exception);
            return;
        }
        m_builder.SetResult(result);
    }
}
```

#### 4.异步函数扩展性  
在扩展性方面，能用Task对象包装一个将来完成的操作，就可以用await操作符来等待该操作。  
用一个类型(Task)来表示各种异步操作对编码有你，因为可以实现组合操作(比如Task的WhenAll和WhenAny方法)  

除了增强使用Task时的灵活性，异步函数另一个对扩展性有利的地方在于编译器可以再await的任何操作数上调用GetAwaiter。  

#### 5.异步函数和时间处理程序
异步函数的返回类型一般是Task或Task<TResult>，他们代表函数的状态机完成。但异步函数是可以返回void的  
几乎所有事件处理程序都遵循以下方法签名:  
```
void EventHandlerCallback(Object sender, EventArg e);
```

#### 6.FCL的异步函数
1. System.IO.Stream的所有派生类都提供了ReadAsync、WriteAsync、FlushAsync和CopyToAsync方法
2. System.IO.TextReader的所有派生类都提供了ReadAsync、ReadLineAsync...
3. System.Net.Http.HttpClient类提供了GetAsync、GetStreamAsync...
4. Systen.Net.WebRequest的所有派生类(包括FileWebRequest...)都提供了GetRequestStreamAsync和GetResponseAsync方法
5. System.Data.SqlClient.SqlCommand类提供了ExcuteDbDataReaderAsync...
6. 生成Web五福代理类型的工具(比如SvcUtil.exe)夜深沉XxxASync方法  

#### 7。 异步函数和异常处理

Task对象通常抛出一个AggregateException，可查询该异常的InnerExceptions属性来查看  
但将await用于Task时，通常抛出第一个内部异常而不是AggregateException  

如果状态机出现未处理的异常，而返回void的异步函数抛出未处理的异常时，编译器生成的代码将捕捉它，并使用调用者的同步上下文重新抛出它  

#### 8. 应用程序及其线程处理模型  
任何线程可在任何时候做它想做的任何事情，但GUI应用程序引入了一个线程处理模型。在这个模型中，UI元素只能由创建它的线程更新。  