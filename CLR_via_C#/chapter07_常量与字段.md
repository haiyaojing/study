#### 1.常量值从不变化
编译器会在定义常量的程序集的元数据中查找该符号，提取常量的值并将值潜入IL代码中,所以不需要在运行时为常量分配内存

#### 2.readonly
只能在初始值和构造器里进行修改，解决了常量存在的版本问题