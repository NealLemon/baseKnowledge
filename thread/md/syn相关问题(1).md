# 多线程相关问题（二）

由于近期在做知识储备，在做很多的复习，把之前看过的内容重温一遍真的像重新看一遍一样，真的是一入JAVA深似海。

## 锁相关问题

### 1.synchronized底层原理？

#### Monitor

在讲解之前我们需要了解一下Monitor，Monitor:每个java对象天生自带了一把看不见的锁。我们可以把他看作是一种同步工具或者同步机制。

#### Java对象头

主要结构是由Mark Word 和 Class Metadata Address 组成

p1.png

p2.png

轻量级锁和偏向锁是Java 6 对 synchronized 锁进行优化后新增加的,这里我们主要分析一下重量级锁也就是通常说synchronized的对象锁，锁标识位为10，其中指针指向的是monitor对象。

在Java虚拟机(HotSpot)中，monitor是由ObjectMonitor实现的,接下来我们看一下虚拟机实现的源码，整体源码在  [objectMonitor.hpp](http://hg.openjdk.java.net/jdk8u/jdk8u60/hotspot/file/37240c1019fd/src/share/vm/runtime/objectMonitor.hpp)  由于是C++编写的，我们这里只做逻辑解释就好了。我们来看一下关键部分的代码

```c++
  // initialize the monitor, exception the semaphore, all other fields
  // are simple integers or pointers
  ObjectMonitor() {
    _header       = NULL;
    _count        = 0;  //计数器
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;  //指向持有ObjectMonitor 锁的线程
    _WaitSet      = NULL;   //等待池
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ;  //锁池
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
    _previous_owner_tid = 0;
  }
```



我们可以看到，这段代码中 有等待池(`_WaitSet`)和锁池(`_EntryList`)，用来保存objectwaiter的列表，每个对象锁的线程都封装在objectwaiter中。`_owner`指向持有ObjectMonitor 锁的线程，当多个线程同时访问同一段同步代码的时候呢，首先会进入到 锁池中，当某一个线程获取到 `monitor`时，就进入到 `ObjectMonitor`区域并将其中的`_owner`设置为当前线程，同时计数器+1 ，若当前线程调用wait()方法，将释放`monitor`,`_owner`就恢复为null,计数器-1，同时该线程的objectwaiter就进入到等待池中。同理如果当前线程执行完毕，也是相同操作。

由于`monitor`存在于每个java对象的对象头中，synchronized就是通过这种方式去获取锁的。

具体流程图如下

p3.png

#### 根据demo理解synchronized具体语义实现。

我们先来新建一个demo类。

```java
/**
 * synchronized底层实现demo类
 */
public class SyncBlockAndMethod {
    //同步代码快
    public void syncBlockTask() {
        synchronized (this) {
            System.out.println("Hello World by syncBlockTask");
        }
    }
    //同步方法
    public synchronized void syncMethodTask() {
        System.out.println("Hello World by syncMethodTask");
    }
}
```

然后使用javac命令将其编译为SyncBlockAndMethod.class文件。

接着我们使用javap命令 来看一下该文件的字节码。具体命令如下

`javap -verbose SyncBlockAndMethod.class`

我们截取了这两个方法的字节码代码来分析一下。

###### syncBlockTask方法

```c++
 public void syncBlockTask();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter       //试图获取对象锁 
         4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         7: ldc           #3                  // String Hello World by syncBlockTask
         9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        12: aload_1
        13: monitorexit   //释放对象锁
        14: goto          22
        17: astore_2
        18: aload_1
        19: monitorexit  //释放对象锁
        20: aload_2
        21: athrow
        22: return
      Exception table:
         from    to  target type
             4    14    17   any
            17    20    17   any
      LineNumberTable:
        line 5: 0
        line 6: 4
        line 7: 12
        line 8: 22
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 17
          locals = [ class other/SyncBlockAndMethod, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4
```

从字节码中可知同步语句块的实现使用的是monitorenter 和 monitorexit 指令，其中monitorenter指令指向同步代码块的开始位置，monitorexit指令则指明同步代码块的结束位置，当执行monitorenter指令时，当前线程将试图获取 objectref(即对象锁) 所对应的 monitor 的持有权，当 objectref 的 monitor 的进入计数器为 0，那线程可以成功取得 monitor，并将计数器值设置为 1，取锁成功。倘若其他线程已经拥有 objectref 的 monitor 的所有权，那当前线程将被阻塞，直到正在执行线程执行完毕，即monitorexit指令被执行，执行线程将释放 monitor(锁)并设置计数器值为0 ，其他线程将有机会持有 monitor 。编译器会自动产生一个异常处理器，这个异常处理器声明可处理所有的异常，它的目的就是用来执行 monitorexit 指令。从字节码中也可以看出多了一个monitorexit指令，它就是异常结束时被执行的释放monitor 的指令。

###### syncMethodTask方法

```c++
 public synchronized void syncMethodTask();
    descriptor: ()V
     // ACC_SYNCHRONIZED指明该方法为同步方法
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #5                  // String Hello World by syncMethodTask
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 11: 0
        line 12: 8
}
```

方法级的同步是隐式，即无需通过字节码指令来控制的,当方法调用时，调用指令将会 检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先持有monitor。然后再执行方法，最后再方法完成(无论是正常完成还是非正常完成)时释放monitor。如果执行线程持有了monitor，其他任何线程都无法再获得同一个monitor。如果一个同步方法执行期间抛 出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的monitor将在异常抛到同步方法之外时自动释放。



### 2.synchronized的锁优化

- 锁消除
- 锁粗化
- 偏向锁
- 轻量级锁
- 重量级锁
- 自适应自旋梭



### 3.synchronized和ReentrantLock的区别?

- 实现
  - synchronized依赖虚拟机实现的，操作java对象头中的Mark Word
  - ReentrantLock位于java.util.concurrent.locks包中基于AQS实现,调用Unsafe类的park()方法
- 可重入性
  - 两者都是可重入的
- 公平锁/非公平锁
  - synchronized只能是非公平锁
  - ReentrantLock既可以是公平锁又可以使非公平锁
- 功能性
  - Synchronized的使用比较方便简洁，并且由编译器去保证锁的加锁和释放。
  - ReentrantLock可以更细粒度的控制调用lock()之后必须调用unlock()
    - 判断是否有线程在排队等待获取锁
    - 带超时的锁获取尝试
    - 感知有没有成功获取锁
    - 提供了一个Condition（条件）类，用来实现分组唤醒需要唤醒的线程们。
- 性能
  - 由于JDK6后，synchronized进行了很多锁优化，因此差距不大。