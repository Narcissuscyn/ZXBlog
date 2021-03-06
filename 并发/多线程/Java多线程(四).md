# Java多线程(四)

## 一、并发编程挑战

### 1、上下文切换

#### 1)、时间片

即使是单核处理器也支持多线程执行代码，CPU通过给每个线程分配**CPU时间片**来实现这个机制。

时间片是CPU分配给各个线程的时间，因为时间片非常短，所以CPU通过不停地切换线程执行，让我们感觉多个线程是同时执行的，时间片一般是几十毫秒（ms）。

#### 2)、并行一定更快吗？

相同的程序，并行版本不一定比串行快。

因为CPU在由一个线程切换到另一个线程时，需要保留该线程当下的执行状态（执行到了哪一行，有哪些变量和变量值），在下次继续执行该线程时恢复到原来的状态，这个保存-恢复会消耗一定得到时间。

#### 3)、如何避免频繁的上下文切换?

* 1)、降低锁粒度: 如将数据的ID按照Hash算法取模分段，不同的线程处理不同段的数据，这在`ConcurrentHashMap`中有所体现。
* 2)、CAS算法(CAS无所操作)。**Java的Atomic包使用CAS算法来更新数据，而不需要加锁**。不同于`synchronized`在获得释放锁的过程中会引起线程的切换。
* 3)、使用最少线程。避免创建不需要的线程，比如任务很少，但是创建了很多线程来处理，这
  样会造成大量线程都处于等待状态，尽量使用线程池技术。

协程：在单线程里实现多任务的调度，并在单线程里维持多个任务间的切换

### 2、死锁

如何避免死锁?

1. **避免一个线程同时获取多个锁**；
2. 避免一个线程同时占用多个资源，尽量保证每个锁只占用一个资源；
3. 尝试使用**定时锁**，使用`lock.tryLock(timeout)`来替代使用内部锁机制，一定时间获取不到就返回；
4. 对于数据库锁，**加锁和解锁必须在一个数据库连接里**，否则会出现解锁失败的情况；

## 二、并发机制底层实现原理

### 1、本地内存和线程安全问题

相关:

* 1)、缓存行: CPU不会直接和内存交互，而是通过总线将数据读到自己的缓存行中。
* 2)、写缓冲区: CPU不会直接和内存交互，会将要读取的数据先写入到自己的写缓冲区，随后才会刷新到内存。
* 3)、本地内存: 这是虚拟出来的概念，实际并不存在，包括了**缓冲行，写缓冲区**等概念。
  * 实际中都是由很多CPU来执行并发程序，不同处理器同时执行不同的线程（每个线程都有一个仅对执行自己的处理器可见的本地内存)。
  * 所以就会出现主内存中`i = 1`，线程A读取到自己的本地内存i++，于此同时线程B也读取到主内存`i= 1`到自己的本地内存执行i++，待两个线程的本地内存刷新到主内存时`i = 2`。于是引发了线程安全问题。

线程安全问题总结:

* 线程都是在自己的本地内存中操作共享变量的，仅对执行自己的处理器可见而对其他处理器不可见。
* 而它们在自己的本地内存对共享变量的更新何时会刷新到主内存、会按照什么顺序刷新到主内存是不可预见的。

使用volatile关键字可以保证共享变量之间的可见性。

volatile做的两件事:

* 1)、锁定缓存行；
* 2)、刷新内存，保证数据一致性；

![1555499561112](assets/1555499561112.png)

一旦一个共享变量（类的成员变量、类的静态成员变量）被volatile修饰之后，那么就具备了两层语义：

1. 保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。
2. **禁止进行指令重排序**。

### 2、volatile实现原理

加入volatile关键字和没有加入volatile关键字时所生成的汇编代码发现，**加入volatile关键字时，会多出一个lock前缀指令**

lock前缀指令实际上相当于一个**内存屏障**（也成内存栅栏），内存屏障会提供3个功能：

1. 它确保指令重排序时**不会把其后面的指令排到内存屏障之前的位置**，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；
2. **它会强制将对缓存的修改操作立即写入主存**；
3. 如果是写操作，它会导致其他CPU中对应的缓存行无效。

在JVM底层volatile是采用“内存屏障”来实现的。

有两层语义

1. 保证可见性、不保证原子性
2. 禁止指令重排序

第一层语义就不做介绍了，下面重点介绍指令重排序。

在执行程序时为了提高性能，编译器和处理器通常会对指令做重排序：

1. 编译器重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序；
2. 处理器重排序。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序；

指令重排序对单线程没有什么影响，他不会影响程序的运行结果，但是会影响多线程的正确性。既然指令重排序会影响到多线程执行的正确性，那么我们就需要禁止重排序。那么JVM是如何禁止重排序的呢？这个问题稍后回答，我们先看另一个原则happens-before，happen-before原则保证了程序的“有序性”，它规定如果两个操作的执行顺序无法从happens-before原则中推到出来，那么他们就不能保证有序性，可以随意进行重排序。其定义如下：

1. 同一个线程中的，前面的操作 happen-before 后续的操作。（即单线程内按代码顺序执行。但是，在不影响在单线程环境执行结果的前提下，编译器和处理器可以进行重排序，这是合法的。换句话说，这一是规则无法保证编译重排和指令重排）。
2. 监视器上的解锁操作 happen-before 其后续的加锁操作。（Synchronized 规则）
3. **对volatile变量的写操作 happen-before 后续的读操作**。（volatile 规则）
4. 线程的start() 方法 happen-before 该线程所有的后续操作。（线程启动规则）
5. 线程所有的操作 happen-before 其他线程在该线程上调用 join 返回成功后的操作。
6. 如果 a happen-before b，b happen-before c，则a happen-before c（传递性）。

我们着重看第三点volatile规则：对volatile变量的写操作 happen-before 后续的读操作。为了实现volatile内存语义，JMM会重排序。

**观察加入volatile关键字和没有加入volatile关键字时所生成的汇编代码发现，加入volatile关键字时，会多出一个lock前缀指令**。**lock前缀指令其实就相当于一个内存屏障**。内存屏障是一组处理指令，用来实现对内存操作的顺序限制。volatile的底层就是通过内存屏障来实现的。

### 3、sychronized

利用synchronized实现同步的基础，Java中的每一个对象都可以作为锁，有以下3种形式

- 对于普通同步方法，锁是当前实例对象
- 对于静态同步方法，锁是当前类的Class对象
- 对于同步方法块，锁住的是synchonized括号里配置的对象

Java 虚拟机中的同步(Synchronization)是基于进入和退出**Monitor对象**实现， **无论是显式同步(有明确的 monitorenter 和 monitorexit 指令**，即同步代码块)还是隐式同步都是如此。

在 Java 语言中，同步用的最多的地方可能是被 synchronized 修饰的同步方法。**同步方法并不是由monitorenter 和 monitorexit 指令来实现同步的，而是由方法调用指令读取运行时常量池中方法的 ACC_SYNCHRONIZED 标志来隐式实现的**。

```java
1    public void add(Object obj){
2        synchronized (obj){
3            //do something
4        }
5    }

反编译后：
 1public class com.zxin.thread.SynchronizedDemo {
 2  public com.wuzy.thread.SynchronizedDemo();
 3    Code:
 4       0: aload_0
 5       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
 6       4: return
 7
 8  public void add(java.lang.Object);
 9    Code:
10       0: aload_1
11       1: dup
12       2: astore_2
13       3: monitorenter //注意此处，进入同步方法
14       4: aload_2
15       5: monitorexit //注意此处，退出同步方法
16       6: goto          14
17       9: astore_3
18      10: aload_2
19      11: monitorexit //注意此处，退出同步方法
20      12: aload_3
21      13: athrow
22      14: return
23    Exception table:
24       from    to  target type
25           4     6     9   any
26           9    12     9   any
27}
```

我们看下第13行~15行代码**，发现同步代码块是使用monitorenter和monitorexit指令来进行代码同步的,注意看第19行代码，为什么会多出一个monitorexit指令，主要是JVM为了防止代码出现异常**，也能正确退出同步方法。

同步方法并不是用monitorenter和monitorexit指令来进行同步的，实际上同步方法会被翻译成普通的方法调用和返回指令如:invokevirtual、areturn指令，在VM字节码层面并没有任何特别的指令来实现被synchronized修饰的方法，而是**在Class文件的方法表中将该方法的access_flags字段中的synchronized标志位置设为1**，表示该方法是同步方法并使用调用该方法的对象或该方法所属的Class在JVM的内部对象表示做为锁对象。