文章链接:https://neuecc.medium.com/unitask-v2-zero-allocation-async-await-for-unity-with-asynchronous-linq-1aa9c96aa7dd
github:https://github.com/Cysharp/UniTask

以下是对其中的一些点的一些摘要

以往async/await模型存在以下问题:
1. Callback process still remains
2. Can’t use try-catch-finally because of the yield syntax
3. Allocation of the lambda and the coroutine itself
4. Difficult to do cancellation process(coroutine only stop, not call dispose)
5. Impossible to control multiple coroutines (serial/parallel processing)


相对于传统的async/await，重写了async自动展开的状态机，参考.net 5的ValueTask实现了基于struct的UniTask，提供了针对unity使用的各种异步接口，linq的接口
主要优化:
1. Zero-allocation of the entire async method to further improve performance
2. Asynchronous LINQ(UniTaskAsyncEnumerable, Channel, AsyncReactiveProperty)
3. Increased PlayerLoop timing (new LastPostLateUpdate will have the same effect as WaitForEndOfFrame)
4. Support for external assets such as Addressables and DOTween

通常堆内存分配发生时机:
1. Task Allocation
2. Boxing the AsyncStateMachine
3. Allocation of the Runner encapsulating the AsyncStateMachine
4. Allocation of delegate for MoveNext

使用时需要注意的点:
1. Awaiting the instance multiple times.
2. Calling AsTask multiple times.
3. Using .Result or .GetAwaiter().GetResult() when the operation hasn’t yet completed, or using them multiple times.
4. Using more than one of these techniques to consume the instance.

其余的参考文章即可。