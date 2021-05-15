# 并发笔记

## 内存可见性

Synchronize：Java 内置锁可以实现内存可见性的保障，对于一个变量在多线程间进行共享修改时。通过内置锁进行同步修改（同一个锁）可以保障 前一个线程对该变量对修改总是能被后一个线程可见。

Volatile：Volatile 关键字修饰的字段修改总是能实时的被其它线程所看见，因为被该关键字修饰的字段不允许存在对处理器不可见的地方。而且该关键字还有个特性，该关键字修饰的字段不能参与内存的从排序操作。针对该特性就能保障了对该字作进行写入时，该字段之前的所有内存变量也会被写入到主内存中从而被其它线程所识别。

#### 线程安全

Java 线程安全主要是通过锁来实现，但是在此之外还可以通过线程封闭来实现一种线程安全操作（该实现主要是一种程序的设计方式，而非 Java 语言级）