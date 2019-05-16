###1.异常
```
catch(Exception e) {
    throw e; //CLR认为这个是异常的起点，FxCop报错
}
catch(Exception e) {
    throw; //不影响CLR对异常起点的认知，FxCop不在报错
}
```
###2.finally	
无论线程抛出什么类型的异常，finally块中的代码都会执行。应该先用finally块清理那些已经成功
的启动的操作，再返回至调用者或者执行finally块之后的代码，另外还通常利用finally块显式释放对象以免资源泄露

许多编程语言都提供了一些构造来简化清理代码的编写，例如lock、using和foreach语句，C#编
译器会为它们自动生成try/finally块。另外重写的析构器（Finalize方法），C#也会自动生成块
1. 使用lock语句时，锁在finally块中释放
2. 使用using语句时，在finally块中调用对象的dispose方法
3. 使用foreach语句时，在finally块中调用IEnumerator对象的Dispose方法
4. 定义析构器时，在finally块中调用基类的Finalize方法