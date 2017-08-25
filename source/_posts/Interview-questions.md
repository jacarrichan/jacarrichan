title: 整理面试题目
date: 2016-03-30 19:12:02
tags:  [JVM,设计模式,Spring]
---

常见基础的面试问题

<!-- more -->

### jvm虚拟机 内存
#### 栈

> Java栈是与每一个线程关联的，JVM在创建每一个线程的时候，会分配一定的栈空间给线程。存储局部变量、引用、方法、返回值等。
StackOverflowError：如果在线程执行的过程中，栈空间不够用，那么JVM就会抛出此异常，这种情况一般是死递归造成的。

####  堆

> Java中堆是由所有的线程共享的一块内存区域，堆用来保存各种JAVA对象，比如数组，线程对象等。


#####  堆的分代
JVM堆一般分为三个部分：

Young：年轻代
> Young区被划分为三部分，Eden区和两个大小严格相同的Survivor区，其中Survivor区间中，某一时刻只有其中一个是被使用的，另外一个留做垃圾收集时复制对象用，在Young区间变满的时候，minor GC就会将存活的对象移到空闲的Survivor区间中，根据JVM的策略，在经过几次垃圾收集后，任然存活于Survivor的对象将被移动到Tenured区间。

Tenured：年老代
> Tenured区主要保存生命周期长的对象，一般是一些老的对象，当一些对象在Young复制转移一定的次数以后，对象就会被转移到Tenured区，一般如果系统中用了application级别的缓存，缓存中的对象往往会被转移到这一区间。

Perm：持久代
> Perm代主要保存class,method,filed等对象，这部门的空间一般不会溢出。
java.lang.OutOfMemoryError : PermGen space 的错误，造成这个错误的很大原因就有可能是每次都重新部署，但是重新部署后，类的class没有被卸载掉，这样就造成了大量的class对象保存在了perm中，这种情况下，一般重新启动应用服务器可以解决问题。


### 内存泄露

 内存泄露是怎么发生的
一般来说内存泄漏有两种情况。一种情况如在C/C++ 语言中的，在堆中的分配的内存，在没有将其释放掉的时候，就将所有能访问这块内存的方式都删掉（如指针重新赋值）；另一种情况则是在内存对象明明已经不需要的时候，还仍然保留着这块内存和它的访问方式（引用）。第一种情况，在 Java 中已经由于垃圾回收机制的引入，得到了很好的解决。所以， Java 中的内存泄漏，主要指的是第二种情况。

#### 如何设计一个程序，可以是内存发生泄露
 
```
Vector v = new  Vector( 10 );  
for  ( int  i = 1 ;i < 100 ; i ++ ){  
    Object o = new  Object();  
    v.add(o);  
    o = null ;  
} 
```



### 事务及事务级别(可以联想到数据库事物)

#### 事务特性

事务应该具有4个属性：原子性、一致性、隔离性、持久性。这四个属性通常称为ACID特性。

1. 原子性（atomicity）。一个事务是一个不可分割的工作单位，事务中包括的诸操作要么都做，要么都不做。
1. 一致性（consistency）。事务必须是使数据库从一个一致性状态变到另一个一致性状态。一致性与原子性是密切相关的。
1. 隔离性（isolation）。一个事务的执行不能被其他事务干扰。即一个事务内部的操作及使用的数据对并发的其他事务是隔离的，并发执行的各个事务之间不能互相干扰。
1. 持久性（durability）。持久性也称永久性（permanence），指一个事务一旦提交，它对数据库中数据的改变就应该是永久性的。接下来的其他操作或故障不应该对其有任何影响。

####  事务的传播

1. PROPAGATION_REQUIRED--支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择。 
1. PROPAGATION_SUPPORTS--支持当前事务，如果当前没有事务，就以非事务方式执行。 
1. PROPAGATION_MANDATORY--支持当前事务，如果当前没有事务，就抛出异常。 
1. PROPAGATION_REQUIRES_NEW--新建事务，如果当前存在事务，把当前事务挂起。 
1. PROPAGATION_NOT_SUPPORTED--以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。 
1. PROPAGATION_NEVER--以非事务方式执行，如果当前存在事务，则抛出异常。 

#### 事物的隔离

1. Serializable：最严格的级别，事务串行执行，资源消耗最大；

2. REPEATABLE  READ：保证了一个事务不会修改已经由另一个事务读取但未提交（回滚）的数据。避免了“脏读取”和“不可重复读取”的情况，但是带来了更多的性能损失。

3. READ  COMMITTED:大多数主流数据库的默认事务等级，保证了一个事务不会读到另一个并行事务已修改但未提交的数据，避免了“脏读取”。该级别适用于大多数系统。

4. Read  Uncommitted：保证了读取过程中不会读取到非法数据。隔离级别在于处理多事务的并发问题。

### Tomcat中间件classLoader机制

#### JVM默认的classloader机制

jvm默认定义了三种classloader，分别是bootstrap classloader、extension classloader、system classloader 

1. bootstrap是jvm的一部分，用C写的，每一个java程序都会启动它，去加载%JAVA_HOME%/jre/lib/rt.jar 

1. extension也差不多，它会去加载%JAVA_HOME%/jre/lib/ext/下的类 

1. system则是会去加载系统变量CLASSPATH下的所有类 

Tomcat在这个基础上扩展了Common和Webapp classloader

1. Common — 这个classloader加载的类，对tomcat的类和web app的类都是可见的。通常来说，应用程序的类不应该放在这里。该加载器的加载路径是在$CATALINA_BASE/conf/catalina.properties文件里，通过common.loader属性来定义的，
1. WebappX — 该classloader加载所有WEB-INF/classes里的类，以及WEB-INF/lib里的jar ，该classloader就有意违反了上述的委托模型，它首先看WEB-INF/classes和WEB-INF/lib里是否有请求的类，而不是委托parent classloader去加载。但是，JRE里定义的类不能被覆盖（比如java.lang.String），以及Servlet API会明确地被忽略。 

----------

###  集合

```
Collection
├List
│├LinkedList（非同步、基于链表）
│├ArrayList（非同步、基于动态数组）
│└Vector (同步、线程安全)
│　└Stack
└Set（不包含重复的元素）

Map
├Hashtable（同步的，并且不允许null）
├HashMap（非同步的，并且允许null）
└WeakHashMap
```

TreeMap中所有的元素都保持着某种固定的顺序

哈希冲突
>散列表要解决的一个问题就是散列值的冲突问题，通常是两种方法：链表法和开放地址法。链表法就是将相同hash值的对象组织成一个链表放在hash值对应的槽位；开放地址法是通过一个探测算法，当某个槽位已经被占据的情况下继续查找下一个可以使用的槽位。java.util.HashMap采用的链表法的方式，链表是单向链表。

----------

### 数据库(主要Oracle)

#### 表查询，如何提高效率


#### 如何建索引


---

#### 悲观锁
-  悲观锁是指假设并发更新冲突会发生，所以不管冲突是否真的发生，都会使用锁机制。
-  悲观锁会完成以下功能：锁住读取的记录，防止其它事务读取和更新这些记录。其它事务会一直阻塞，直到这个事务结束。
-  悲观锁是在使用了数据库的事务隔离功能的基础上，独享占用的资源，以此保证读取数据一致性，避免修改丢失。
-  悲观锁可以使用Repeatable Read事务，它完全满足悲观锁的要求。

#### 乐观锁
-  乐观锁不会锁住任何东西，也就是说，它不依赖数据库的事务机制，乐观锁完全是应用系统层面的东西。
-  如果使用乐观锁，那么数据库就必须加版本字段，否则就只能比较所有字段，但因为浮点类型不能比较，所以实际上没有版本字段是不可行的。


### 多线程

1. 进程和线程之间有什么不同？
> 一个进程是一个独立(self contained)的运行环境，它可以被看作一个程序或者一个应用。而线程是在进程中执行的一个任务。Java运行环境是一个包含了不同的类和程序的单一进程。线程可以被称为轻量级进程。线程需要较少的资源来创建和驻留在进程中，并且可以共享进程中的资源。

2. 多线程编程的好处是什么？
> 在多线程程序中，多个线程被并发的执行以提高程序的效率，CPU不会因为某个线程需要等待资源而进入空闲状态。多个线程共享堆内存(heap memory)，因此创建多个线程去执行一些任务会比创建多个进程更好。举个例子，Servlets比CGI更好，是因为Servlets支持多线程而CGI不支持。

3. 用户线程和守护线程有什么区别？
> 当我们在Java程序中创建一个线程，它就被称为用户线程。一个守护线程是在后台执行并且不会阻止JVM终止的线程。当没有用户线程在运行的时候，JVM关闭程序并且退出。一个守护线程创建的子线程依然是守护线程。

4. 我们如何创建一个线程？
> 有三种创建线程的方法：一是实现Runnable接口，然后将它传递给Thread的构造函数，创建一个Thread对象；二是直接继承Thread类；三是实现Callable接口。

5. 有哪些不同的线程生命周期？
> 当我们在Java程序中新建一个线程时，它的状态是New。当我们调用线程的start()方法时，状态被改变为Runnable。线程调度器会为Runnable线程池中的线程分配CPU时间并且讲它们的状态改变为Running。其他的线程状态还有Waiting，Blocked 和Dead。读这篇文章可以了解更多关于线程生命周期的知识。

6. 可以直接调用Thread类的run()方法么？
> 当然可以，但是如果我们调用了Thread的run()方法，它的行为就会和普通的方法一样，为了在新的线程中执行我们的代码，必须使用Thread.start()方法。

7. 如何让正在运行的线程暂停一段时间？
> 我们可以使用Thread类的Sleep()方法让线程暂停一段时间。需要注意的是，这并不会让线程终止，一旦从休眠中唤醒线程，线程的状态将会被改变为Runnable，并且根据线程调度，它将得到执行。

8. 你对线程优先级的理解是什么？
> 每一个线程都是有优先级的，一般来说，高优先级的线程在运行时会具有优先权，但这依赖于线程调度的实现，这个实现是和操作系统相关的(OS dependent)。我们可以定义线程的优先级，但是这并不能保证高优先级的线程会在低优先级的线程前执行。线程优先级是一个int变量(从1-10)，1代表最低优先级，10代表最高优先级。

9. 什么是线程调度器(Thread Scheduler)和时间分片(Time Slicing)？
> 线程调度器是一个操作系统服务，它负责为Runnable状态的线程分配CPU时间。一旦我们创建一个线程并启动它，它的执行便依赖于线程调度器的实现。时间分片是指将可用的CPU时间分配给可用的Runnable线程的过程。分配CPU时间可以基于线程优先级或者线程等待的时间。线程调度并不受到Java虚拟机控制，所以由应用程序来控制它是更好的选择（也就是说不要让你的程序依赖于线程的优先级）。

10. 在多线程中，什么是上下文切换(context-switching)？
> 上下文切换是存储和恢复CPU状态的过程，它使得线程执行能够从中断点恢复执行。上下文切换是多任务操作系统和多线程环境的基本特征。

11. 你如何确保main()方法所在的线程是Java程序最后结束的线程？
> 我们可以使用Thread类的joint()方法来确保所有程序创建的线程在main()方法退出前结束。这里有一篇文章关于Thread类的joint()方法。

12. 线程之间是如何通信的？
> 当线程间是可以共享资源时，线程间通信是协调它们的重要的手段。Object类中wait()\notify()\notifyAll()方法可以用于线程间通信关于资源的锁的状态。点击这里有更多关于线程wait, notify和notifyAll.

13. 为什么线程通信的方法wait(), notify()和notifyAll()被定义在Object类里？
> Java的每个对象中都有一个锁(monitor，也可以成为监视器) 并且wait()，notify()等方法用于等待对象的锁或者通知其他线程对象的监视器可用。在Java的线程中并没有可供任何对象使用的锁和同步器。这就是为什么这些方法是Object类的一部分，这样Java的每一个类都有用于线程间通信的基本方法

14. 为什么wait(), notify()和notifyAll()必须在同步方法或者同步块中被调用？
> 当一个线程需要调用对象的wait()方法的时候，这个线程必须拥有该对象的锁，接着它就会释放这个对象锁并进入等待状态直到其他线程调用这个对象上的notify()方法。同样的，当一个线程需要调用对象的notify()方法时，它会释放这个对象的锁，以便其他在等待的线程就可以得到这个对象锁。由于所有的这些方法都需要线程持有对象的锁，这样就只能通过同步来实现，所以他们只能在同步方法或者同步块中被调用。

15. 为什么Thread类的sleep()和yield()方法是静态的？
> Thread类的sleep()和yield()方法将在当前正在执行的线程上运行。所以在其他处于等待状态的线程上调用这些方法是没有意义的。这就是为什么这些方法是静态的。它们可以在当前正在执行的线程中工作，并避免程序员错误的认为可以在其他非运行线程调用这些方法。

16. 如何确保线程安全？
> 在Java中可以有很多方法来保证线程安全——同步，使用原子类(atomic concurrent classes)，实现并发锁，使用volatile关键字，使用不变类和线程安全类。在线程安全教程中，你可以学到更多。

17. volatile关键字在Java中有什么作用？
> 当我们使用volatile关键字去修饰变量的时候，所以线程都会直接读取该变量并且不缓存它。这就确保了线程读取到的变量是同内存中是一致的。

18. 同步方法和同步块，哪个是更好的选择？
> 同步块是更好的选择，因为它不会锁住整个对象（当然你也可以让它锁住整个对象）。同步方法会锁住整个对象，哪怕这个类中有多个不相关联的同步块，这通常会导致他们停止执行并需要等待获得这个对象上的锁。

19. 如何创建守护线程？
> 使用Thread类的setDaemon(true)方法可以将线程设置为守护线程，需要注意的是，需要在调用start()方法前调用这个方法，否则会抛出IllegalThreadStateException异常。

20. 什么是ThreadLocal?
> ThreadLocal用于创建线程的本地变量，我们知道一个对象的所有线程会共享它的全局变量，所以这些变量不是线程安全的，我们可以使用同步技术。但是当我们不想使用同步的时候，我们可以选择ThreadLocal变量。每个线程都会拥有他们自己的Thread变量，它们可以使用get()\set()方法去获取他们的默认值或者在线程内部改变他们的值。ThreadLocal实例通常是希望它们同线程状态关联起来是private static属性。在ThreadLocal例子这篇文章中你可以看到一个关于ThreadLocal的小程序。

21. 什么是Thread Group？为什么建议不使用它？
> ThreadGroup是一个类，它的目的是提供关于线程组的信息。ThreadGroup API比较薄弱，它并没有比Thread提供了更多的功能。它有两个主要的功能：一是获取线程组中处于活跃状态线程的列表；二是设置为线程设置未捕获异常处理器(ncaught exception handler)。但在Java 1.5中Thread类也添加了setUncaughtExceptionHandler(UncaughtExceptionHandler eh) 方法，所以ThreadGroup是已经过时的，不建议继续使用。

22. 什么是Java线程转储(Thread Dump)，如何得到它？
> 线程转储是一个JVM活动线程的列表，它对于分析系统瓶颈和死锁非常有用。有很多方法可以获取线程转储——使用Profiler，Kill -3命令，jstack工具等等。我更喜欢jstack工具，因为它容易使用并且是JDK自带的。由于它是一个基于终端的工具，所以我们可以编写一些脚本去定时的产生线程转储以待分析。读这篇文档可以了解更多关于产生线程转储的知识。

23. 什么是死锁(Deadlock)？如何分析和避免死锁？
> 死锁是指两个以上的线程永远阻塞的情况，这种情况产生至少需要两个以上的线程和两个以上的资源。分析死锁，我们需要查看Java应用程序的线程转储。我们需要找出那些状态为BLOCKED的线程和他们等待的资源。每个资源都有一个唯一的id，用这个id我们可以找出哪些线程已经拥有了它的对象锁。避免嵌套锁，只在需要的地方使用锁和避免无限期等待是避免死锁的通常办法，阅读这篇文章去学习如何分析死锁。

24. 什么是Java Timer类？如何创建一个有特定时间间隔的任务？
> java.util.Timer是一个工具类，可以用于安排一个线程在未来的某个特定时间执行。Timer类可以用安排一次性任务或者周期任务。
> java.util.TimerTask是一个实现了Runnable接口的抽象类，我们需要去继承这个类来创建我们自己的定时任务并使用Timer去安排它的执行。



25. 什么是线程池？如何创建一个Java线程池？
> 一个线程池管理了一组工作线程，同时它还包括了一个用于放置等待执行的任务的队列。java.util.concurrent.Executors提供了一个 java.util.concurrent.Executor接口的实现用于创建线程池。线程池例子展现了如何创建和使用线程池，或者阅读ScheduledThreadPoolExecutor例子，了解如何创建一个周期任务。

### Java并发面试问题

1. 什么是原子操作？在Java Concurrency API中有哪些原子类(atomic classes)？
> 原子操作是指一个不受其他操作影响的操作任务单元。原子操作是在多线程环境下避免数据不一致必须的手段。int++并不是一个原子操作，所以当一个线程读取它的值并加1时，另外一个线程有可能会读到之前的值，这就会引发错误。为了解决这个问题，必须保证增加操作是原子的，在JDK1.5之前我们可以使用同步技术来做到这一点。到JDK1.5，java.util.concurrent.atomic包提供了int和long类型的装类，它们可以自动的保证对于他们的操作是原子的并且不需要使用同步。可以阅读这篇文章来了解Java的atomic类。

2. Java Concurrency API中的Lock接口(Lock interface)是什么？对比同步它有什么优势？
> Lock接口比同步方法和同步块提供了更具扩展性的锁操作。他们允许更灵活的结构，可以具有完全不同的性质，并且可以支持多个相关类的条件对象。它的优势有：
> 可以使锁更公平;
> 可以使线程在等待锁的时候响应中断;
> 可以让线程尝试获取锁，并在无法获取锁的时候立即返回或者等待一段时间;
> 可以在不同的范围，以不同的顺序获取和释放锁;

3. 什么是Executors框架？
>  Executor框架同java.util.concurrent.Executor 接口在Java 5中被引入。Executor框架是一个根据一组执行策略调用，调度，执行和控制的异步任务的框架。无限制的创建线程会引起应用程序内存溢出。所以创建一个线程池是个更好的的解决方案，因为可以限制线程的数量并且可以回收再利用这些线程。利用Executors框架可以非常方便的创建一个线程池，阅读这篇文章可以了解如何使用Executor框架创建一个线程池。

4. 什么是阻塞队列？如何使用阻塞队列来实现生产者-消费者模型？
> java.util.concurrent.BlockingQueue的特性是：当队列是空的时，从队列中获取或删除元素的操作将会被阻塞，或者当队列是满时，往队列里添加元素的操作会被阻塞。
> 阻塞队列不接受空值，当你尝试向队列中添加空值的时候，它会抛出NullPointerException。阻塞队列的实现都是线程安全的，所有的查询方法都是原子的并且使用了内部锁或者其他形式的并发控制。BlockingQueue 接口是java collections框架的一部分，它主要用于实现生产者-消费者问题。


5. 什么是Callable和Future?
> Java 5在concurrency包中引入了java.util.concurrent.Callable 接口，它和Runnable接口很相似，但它可以返回一个对象或者抛出一个异常。
> Callable接口使用泛型去定义它的返回类型。Executors类提供了一些有用的方法去在线程池中执行Callable内的任务。由于Callable任务是并行的，我们必须等待它返回的结果。java.util.concurrent.Future对象为我们解决了这个问题。在线程池提交Callable任务后返回了一个Future对象，使用它我们可以知道Callable任务的状态和得到Callable返回的执行结果。Future提供了get()方法让我们可以等待Callable结束并获取它的执行结果。


6. 什么是FutureTask?
> FutureTask是Future的一个基础实现，我们可以将它同Executors使用处理异步任务。通常我们不需要使用FutureTask类，单当我们打算重写Future接口的一些方法并保持原来基础的实现是，它就变得非常有用。我们可以仅仅继承于它并重写我们需要的方法。阅读Java FutureTask例子，学习如何使用它。

7. 什么是并发容器的实现？
> Java集合类都是快速失败的，这就意味着当集合被改变且一个线程在使用迭代器遍历集合的时候，迭代器的next()方法将抛出ConcurrentModificationException异常。并发容器支持并发的遍历和并发的更新。主要的类有ConcurrentHashMap, CopyOnWriteArrayList 和CopyOnWriteArraySet，阅读这篇文章了解如何避免ConcurrentModificationException。


8. Executors类是什么？
> Executors为Executor，ExecutorService，ScheduledExecutorService，ThreadFactory和Callable类提供了一些工具方法。Executors可以用于方便的创建线程池。

----
###  Java的锁


#### 自旋锁

自旋锁是采用让当前线程不停地的在循环体内执行实现的，当循环的条件被其他线程改变时 才能进入临界区

在自旋锁中 另有三种常见的锁形式:TicketLock ，CLHlock 和MCSlock

1. Ticket锁主要解决的是访问顺序的问题，主要的问题是在多核cpu上
1. CLHlock是不停的查询前驱变量， 导致不适合在NUMA 架构下使用（在这种结构下，每个线程分布在不同的物理内存区域）
1. MCSLock则是对本地变量的节点进行循环。

#### 阻塞锁

可以说是让线程进入阻塞状态进行等待，当获得相应的信号（唤醒，时间） 时，才可以进入线程的准备就绪状态，准备就绪状态的所有线程，通过竞争，进入运行状态。
JAVA中，能够进入\退出、阻塞状态或包含阻塞锁的方法有 ，synchronized 关键字（其中的重量锁），ReentrantLock，Object.wait()\notify(),LockSupport.park()/unpart()(j.u.c经常使用)


阻塞锁的优势在于，阻塞的线程不会占用cpu时间， 不会导致 CPu占用率过高，但进入时间以及恢复时间都要比自旋锁略慢。

在竞争激烈的情况下 阻塞锁的性能要明显高于 自旋锁。

理想的情况则是; 在线程竞争不激烈的情况下，使用自旋锁，竞争激烈的情况下使用，阻塞锁。

#### 可重入锁

广义上的可重入锁，不单指JAVA下的ReentrantLock。

可重入锁，也叫做递归锁，指的是同一线程 外层函数获得锁之后 ，内层递归函数仍然有获取该锁的代码，但不受影响。
在JAVA环境下 ReentrantLock 和synchronized 都是 可重入锁

#### 其它
互斥锁 、悲观锁、乐观锁、公平锁、非公平锁、偏向锁

同步 ： 自己亲自出马持银行卡到银行取钱（使用同步IO时，Java自己处理IO读写）。
异步 ： 委托一小弟拿银行卡到银行取钱，然后给你（使用异步IO时，Java将IO读写委托给OS处理，需要将数据缓冲区地址和大小传给OS(银行卡和密码)，OS需要支持异步IO操作API）。
阻塞 ： ATM排队取款，你只能等待（使用阻塞IO时，Java调用会一直阻塞到读写完成才返回）。
非阻塞 ： 柜台取款，取个号，然后坐在椅子上做其它事，等号广播会通知你办理，没到号你就不能去，你可以不断问大堂经理排到了没有，大堂经理如果说还没到你就不能去（使用非阻塞IO时，如果不能读写Java调用会马上返回，当IO事件分发器会通知可读写时再继续进行读写，不断循环直到读写完成）。

----

### Java BIO、NIO、AIO

#### Java对BIO、NIO、AIO的支持：

1. Java BIO ： 同步并阻塞，服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销，当然可以通过线程池机制改善。
1. Java NIO ： 同步非阻塞，服务器实现模式为一个请求一个线程，即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有I/O请求时才启动一个线程进行处理。
1. Java AIO(NIO.2) ： 异步非阻塞，服务器实现模式为一个有效请求一个线程，客户端的I/O请求都是由OS先完成了再通知服务器应用去启动线程进行处理，

#### BIO、NIO、AIO适用场景分析:

1. BIO方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4以前的唯一选择，但程序直观简单易理解。
1. NIO方式适用于连接数目多且连接比较短（轻操作）的架构，比如聊天服务器，并发局限于应用中，编程比较复杂，JDK1.4开始支持。
1. AIO方式使用于连接数目多且连接比较长（重操作）的架构，比如相册服务器，充分调用OS参与并发操作，编程比较复杂，JDK7开始支持。

另外，I/O属于底层操作，需要操作系统支持，并发也需要操作系统的支持，所以性能方面不同操作系统差异会比较明显。

----

###  Redis

最为常用的数据类型主要有以下五种：

1. String
1. Hash
1. List
1. Set
1. Sorted set

----


### Spring

#### 什么是spring?
>  Spring 是个java企业级应用的开源开发框架。Spring主要用来开发Java应用，但是有些扩展是针对构建J2EE平台的web应用。Spring 框架目标是简化Java企业级应用开发，并通过POJO为基础的编程模型促进良好的编程习惯。


#### 使用Spring框架的好处是什么？
1.  轻量：Spring 是轻量的，基本的版本大约2MB。
1.  控制反转：Spring通过控制反转实现了松散耦合，对象们给出它们的依赖，而不是创建或查找依赖的对象们。
1.  面向切面的编程(AOP)：Spring支持面向切面的编程，并且把应用业务逻辑和系统服务分开。
容器：Spring 包含并管理应用中对象的生命周期和配置。
1.  MVC框架：Spring的WEB框架是个精心设计的框架，是Web框架的一个很好的替代品。
1.  事务管理：Spring 提供一个持续的事务管理接口，可以扩展到上至本地事务下至全局事务（JTA）。
1.  异常处理：Spring 提供方便的API把具体技术相关的异常（比如由JDBC，Hibernate or JDO抛出的）转化为一致的unchecked 异常。

----------------------------


####   Spring由哪些模块组成?
>  以下是Spring 框架的基本模块：

1. Core module、
1. Bean module、
1. Context module、
1. Expression Language module、
1. JDBC module、
1. ORM module、
1. OXM module、
1. Java Messaging Service(JMS) module、
1. Transaction module、
1. Web module、
1. Web-Servlet module、
1. Web-Struts module、
1. Web-Portlet module、

#### 核心容器（应用上下文) 模块。
>  这是基本的Spring模块，提供spring 框架的基础功能，BeanFactory 是 任何以spring为基础的应用的核心。Spring 框架建立在此模块之上，它使Spring成为一个容器。

#### BeanFactory – BeanFactory 实现举例。
>  Bean 工厂是工厂模式的一个实现，提供了控制反转功能，用来把应用的配置和依赖从正真的应用代码中分离。最常用的BeanFactory 实现是XmlBeanFactory 类。

#### XMLBeanFactory 
  最常用的就是org.springframework.beans.factory.xml.XmlBeanFactory ，它根据XML文件中的定义加载beans。该容器从XML 文件读取配置元数据并用它去创建一个完全配置的系统或应用。

#### 解释AOP模块
>  AOP模块用于发给我们的Spring应用做面向切面的开发， 很多支持由AOP联盟提供，这样就确保了Spring和其他AOP框架的共通性。这个模块将元数据编程引入Spring。

#### 解释JDBC抽象和DAO模块。
>  通过使用JDBC抽象和DAO模块，保证数据库代码的简洁，并能避免数据库资源错误关闭导致的问题，它在各种不同的数据库的错误信息之上，提供了一个统一的异常访问层。它还利用Spring的AOP 模块给Spring应用中的对象提供事务管理服务。

#### 解释对象/关系映射集成模块。
>  Spring 通过提供ORM模块，支持我们在直接JDBC之上使用一个对象/关系映射映射(ORM)工具，Spring 支持集成主流的ORM框架，如Hiberate,JDO和 iBATIS SQL Maps。Spring的事务管理同样支持以上所有ORM框架及JDBC。

####  解释WEB 模块。
>  Spring的WEB模块是构建在application context 模块基础之上，提供一个适合web应用的上下文。这个模块也包括支持多种面向web的任务，如透明地处理多个文件上传请求和程序级请求参数的绑定到你的业务对象。它也有对Jakarta Struts的支持。

####  Spring配置文件
>  Spring配置文件是个XML 文件，这个文件包含了类信息，描述了如何配置它们，以及如何相互调用。

####  什么是Spring IOC 容器？
>  Spring IOC 负责创建对象，管理对象（通过依赖注入（DI），装配对象，配置对象，并且管理这些对象的整个生命周期。

#### IOC的优点是什么？
>  IOC 或 依赖注入把应用的代码量降到最低。它使应用容易测试，单元测试不再需要单例和JNDI查找机制。最小的代价和最小的侵入性使松散耦合得以实现。IOC容器支持加载服务时的饿汉式初始化和懒加载。

#### ApplicationContext通常的实现是什么?
1.  FileSystemXmlApplicationContext ：此容器从一个XML文件中加载beans的定义，XML Bean 配置文件的全路径名必须提供给它的构造函数。
1. ClassPathXmlApplicationContext：此容器也从一个XML文件中加载beans的定义，这里，你需要正确设置classpath因为这个容器将在classpath里找bean配置。
1. WebXmlApplicationContext：此容器加载一个XML文件，此文件定义了一个WEB应用的所有bean。

#### Bean 工厂和 Application contexts  有什么区别？

Application contexts提供一种方法处理文本消息，一个通常的做法是加载文件资源（比如镜像），它们可以向注册为监听器的bean发布事件。另外，在容器或容器内的对象上执行的那些不得不由bean工厂以程序化方式处理的操作，可以在Application contexts中以声明的方式处理。Application contexts实现了MessageSource接口，该接口的实现以可插拔的方式提供获取本地化消息的方法。

#### 一个Spring的应用看起来象什么？

1. 一个定义了一些功能的接口。
这实现包括属性，它的Setter ， getter 方法和函数等。
Spring AOP。
Spring 的XML 配置文件。
使用以上功能的客户端程序。
依赖注入

#### 什么是Spring的依赖注入？

依赖注入，是IOC的一个方面，是个通常的概念，它有多种解释。这概念是说你不用创建对象，而只需要描述它如何被创建。你不在代码里直接组装你的组件和服务，但是要在配置文件里描述哪些组件需要哪些服务，之后一个容器（IOC容器）负责把他们组装起来。

####  有哪些不同类型的IOC（依赖注入）方式？

构造器依赖注入：构造器依赖注入通过容器触发一个类的构造器来实现的，该类有一系列参数，每个参数代表一个对其他类的依赖。
Setter方法注入：Setter方法注入是容器通过调用无参构造器或无参static工厂 方法实例化bean之后，调用该bean的setter方法，即实现了基于setter的依赖注入。

#### 哪种依赖注入方式你建议使用，构造器注入，还是 Setter方法注入？

你两种依赖方式都可以使用，构造器注入和Setter方法注入。最好的解决方案是用构造器参数实现强制依赖，setter方法实现可选依赖。


#### 什么是Spring beans?

Spring beans 是那些形成Spring应用的主干的java对象。它们被Spring IOC容器初始化，装配，和管理。这些beans通过容器中配置的元数据创建。比如，以XML文件中<bean/> 的形式定义。

Spring 框架定义的beans都是单件beans。在bean tag中有个属性”singleton”，如果它被赋为TRUE，bean 就是单件，否则就是一个 prototype bean。默认是TRUE，所以所有在Spring框架中的beans 缺省都是单件。

#### 一个 Spring Bean 定义 包含什么？

一个Spring Bean 的定义包含容器必知的所有配置元数据，包括如何创建一个bean，它的生命周期详情及它的依赖。

#### 如何给Spring 容器提供配置元数据?

这里有三种重要的方法给Spring 容器提供配置元数据。

1. XML配置文件。
1. 基于注解的配置。
1. 基于java的配置。

#### 你怎样定义类的作用域? 

当定义一个<bean> 在Spring里，我们还能给这个bean声明一个作用域。它可以通过bean 定义中的scope属性来定义。如，当Spring要在需要的时候每次生产一个新的bean实例，bean的scope属性被指定为prototype。另一方面，一个bean每次使用的时候必须返回同一个实例，这个bean的scope 属性 必须设为 singleton。

####  解释Spring支持的几种bean的作用域。

Spring框架支持以下五种bean的作用域：

1. singleton : bean在每个Spring ioc 容器中只有一个实例。
1. prototype：一个bean的定义可以有多个实例。
1. request：每次http请求都会创建一个bean，该作用域仅在基于web的Spring  ApplicationContext情形下有效。
1. session：在一个HTTP Session中，一个bean定义对应一个实例。该作用域仅在基于web的Spring ApplicationContext情形下有效。
1. global-session：在一个全局的HTTP Session中，一个bean定义对应一个实例。该作用域仅在基于web的Spring ApplicationContext情形下有效。
1.  缺省的Spring bean 的作用域是Singleton.

####  Spring框架中的单例bean是线程安全的吗?

不，Spring框架中的单例bean不是线程安全的。

####  解释Spring框架中bean的生命周期。

1. Spring容器 从XML 文件中读取bean的定义，并实例化bean。
1. Spring根据bean的定义填充所有的属性。
1. 如果bean实现了BeanNameAware 接口，Spring 传递bean 的ID 到 setBeanName方法。
如果Bean 实现了 BeanFactoryAware 接口， Spring传递beanfactory 给setBeanFactory 方法。
1. 如果有任何与bean相关联的BeanPostProcessors，Spring会在postProcesserBeforeInitialization()方法内调用它们。
1. 如果bean实现IntializingBean了，调用它的afterPropertySet方法，如果bean声明了初始化方法，调用此初始化方法。
1. 如果有BeanPostProcessors 和bean 关联，这些bean的1. postProcessAfterInitialization() 方法将被调用。
1. 如果bean实现了 DisposableBean，它将调用destroy()方法。

#### 哪些是重要的bean生命周期方法？ 你能重载它们吗？

有两个重要的bean 生命周期方法，第一个是setup ， 它是在容器加载bean的时候被调用。第二个方法是 teardown  它是在容器卸载类的时候被调用。

The bean 标签有两个重要的属性（init-method和destroy-method）。用它们你可以自己定制初始化和注销方法。它们也有相应的注解（@PostConstruct和@PreDestroy）。

#### 什么是Spring的内部bean？

当一个bean仅被用作另一个bean的属性时，它能被声明为一个内部bean，为了定义inner bean，在Spring 的 基于XML的 配置元数据中，可以在 <property/>或 <constructor-arg/> 元素内使用<bean/> 元素，内部bean通常是匿名的，它们的Scope一般是prototype。

#### 在 Spring中如何注入一个java集合？

Spring提供以下几种集合的配置元素：

1. <list>类型用于注入一列值，允许有相同的值。
1. <set> 类型用于注入一组值，不允许有相同的值。
1. <map> 类型用于注入一组键值对，键和值都可以为任意类型。
1. <props>类型用于注入一组键值对，键和值都只能为String类型。


####  什么是bean装配? 

装配，或bean 装配是指在Spring 容器中把bean组装到一起，前提是容器需要知道bean的依赖关系，如何通过依赖注入来把它们装配到一起。

#### 什么是bean的自动装配？

Spring 容器能够自动装配相互合作的bean，这意味着容器不需要<constructor-arg>和<property>配置，能通过Bean工厂自动处理bean之间的协作。

#### 解释不同方式的自动装配 。

有五种自动装配的方式，可以用来指导Spring容器用自动装配方式来进行依赖注入。

1. no：默认的方式是不进行自动装配，通过显式设置ref 属性来进行装配。
1. byName：通过参数名 自动装配，Spring容器在配置文件中发现bean的autowire属性被设置成byname，之后容器试图匹配、装配和该bean的属性具有相同名字的bean。
1. byType:：通过参数类型自动装配，Spring容器在配置文件中发现bean的autowire属性被设置成byType，之后容器试图匹配、装配和该bean的属性具有相同类型的bean。如果有多个bean符合条件，则抛出错误。
1. constructor：这个方式类似于byType， 但是要提供给构造器参数，如果没有确定的带参数的构造器参数类型，将会抛出异常。
1. autodetect：首先尝试使用constructor来自动装配，如果无法工作，则使用byType方式。

#### 自动装配有哪些局限性 ?

自动装配的局限性是：

重写： 你仍需用 <constructor-arg>和 <property> 配置来定义依赖，意味着总要重写自动装配。
基本数据类型：你不能自动装配简单的属性，如基本数据类型，String字符串，和类。
模糊特性：自动装配不如显式装配精确，如果有可能，建议使用显式装配。


#### 你可以在Spring中注入一个null 和一个空字符串吗？

可以。


#### 什么是基于Java的Spring注解配置? 给一些注解的例子.

基于Java的配置，允许你在少量的Java注解的帮助下，进行你的大部分Spring配置而非通过XML文件。

以@Configuration 注解为例，它用来标记类可以当做一个bean的定义，被Spring IOC容器使用。另一个例子是@Bean注解，它表示此方法将要返回一个对象，作为一个bean注册进Spring应用上下文。

#### 什么是基于注解的容器配置?

相对于XML文件，注解型的配置依赖于通过字节码元数据装配组件，而非尖括号的声明。

开发者通过在相应的类，方法或属性上使用注解的方式，直接组件类中进行配置，而不是使用xml表述bean的装配关系。

####  怎样开启注解装配？

注解装配在默认情况下是不开启的，为了使用注解装配，我们必须在Spring配置文件中配置 <context:annotation-config/>元素。

#### @Required  注解

这个注解表明bean的属性必须在配置的时候设置，通过一个bean定义的显式的属性值或通过自动装配，若@Required注解的bean属性未被设置，容器将抛出BeanInitializationException。

#### @Autowired 注解

@Autowired 注解提供了更细粒度的控制，包括在何处以及如何完成自动装配。它的用法和@Required一样，修饰setter方法、构造器、属性或者具有任意名称和/或多个参数的PN方法。

#### @Qualifier 注解

当有多个相同类型的bean却只有一个需要自动装配时，将@Qualifier 注解和@Autowire 注解结合使用以消除这种混淆，指定需要装配的确切的bean。



#### 在Spring框架中如何更有效地使用JDBC? 

使用SpringJDBC 框架，资源管理和错误处理的代价都会被减轻。所以开发者只需写statements 和 queries从数据存取数据，JDBC也可以在Spring框架提供的模板类的帮助下更有效地被使用，这个模板叫JdbcTemplate （例子见这里here）

#### JdbcTemplate

JdbcTemplate 类提供了很多便利的方法解决诸如把数据库数据转变成基本数据类型或对象，执行写好的或可调用的数据库操作语句，提供自定义的数据错误处理。

#### Spring对DAO的支持

Spring对数据访问对象（DAO）的支持旨在简化它和数据访问技术如JDBC，Hibernate or JDO 结合使用。这使我们可以方便切换持久层。编码时也不用担心会捕获每种技术特有的异常。

#### 使用Spring通过什么方式访问Hibernate? 

在Spring中有两种方式访问Hibernate：
控制反转  Hibernate Template和 Callback。
继承 HibernateDAOSupport提供一个AOP 拦截器。

#### Spring支持的ORM

Spring支持以下ORM：

Hibernate
iBatis
JPA (Java Persistence API)
TopLink
JDO (Java Data Objects)
OJB

#### 如何通过HibernateDaoSupport将Spring和Hibernate结合起来？

用Spring的 SessionFactory 调用 LocalSessionFactory。集成过程分三步：

配置the Hibernate SessionFactory。
继承HibernateDaoSupport实现一个DAO。
在AOP支持的事务中装配。

#### Spring支持的事务管理类型

Spring支持两种类型的事务管理：

1. 编程式事务管理：这意味你通过编程的方式管理事务，给你带来极大的灵活性，但是难维护。
1. 声明式事务管理：这意味着你可以将业务代码和事务管理分离，你只需用注解和XML配置来管理事务。

#### Spring框架的事务管理有哪些优点？

1. 它为不同的事务API  如 JTA，JDBC，Hibernate，JPA 和JDO，提供一个不变的编程模式。
1. 它为编程式事务管理提供了一套简单的API而不是一些复杂的事务API如
1. 它支持声明式事务管理。
1. 它和Spring各种数据访问抽象层很好得集成。

#### 你更倾向用那种事务管理类型？

大多数Spring框架的用户选择声明式事务管理，因为它对应用代码的影响最小，因此更符合一个无侵入的轻量级容器的思想。声明式事务管理要优于编程式事务管理，虽然比编程式事务管理（这种方式允许你通过代码控制事务）少了一点灵活性。


####  解释AOP

面向切面的编程，或AOP， 是一种编程技术，允许程序模块化横向切割关注点，或横切典型的责任划分，如日志和事务管理。

#### Aspect 切面

AOP核心就是切面，它将多个类的通用行为封装成可重用的模块，该模块含有一组API提供横切功能。比如，一个日志模块可以被称作日志的AOP切面。根据需求的不同，一个应用程序可以有若干切面。在Spring AOP中，切面通过带有@Aspect注解的类实现。

#### 在Spring AOP 中，关注点和横切关注的区别是什么？

关注点是应用中一个模块的行为，一个关注点可能会被定义成一个我们想实现的一个功能。
横切关注点是一个关注点，此关注点是整个应用都会使用的功能，并影响整个应用，比如日志，安全和数据传输，几乎应用的每个模块都需要的功能。因此这些都属于横切关注点。

#### 连接点

连接点代表一个应用程序的某个位置，在这个位置我们可以插入一个AOP切面，它实际上是个应用程序执行Spring AOP的位置。

#### 通知

通知是个在方法执行前或执行后要做的动作，实际上是程序执行时要通过SpringAOP框架触发的代码段。

Spring切面可以应用五种类型的通知：

1. before：前置通知，在一个方法执行前被调用。
1. after: 在方法执行之后调用的通知，无论方法执行是否成功。
1. after-returning: 仅当方法成功完成后执行的通知。
1. after-throwing: 在方法抛出异常退出时执行的通知。
1. around: 在方法执行之前和之后调用的通知。

#### 切点

切入点是一个或一组连接点，通知将在这些位置执行。可以通过表达式或匹配的方式指明切入点。

#### 什么是引入? 

引入允许我们在已存在的类中增加新的方法和属性。

#### 什么是目标对象? 

被一个或者多个切面所通知的对象。它通常是一个代理对象。也指被通知（advised）对象。

#### 什么是代理?

代理是通知目标对象后创建的对象。从客户端的角度看，代理对象和目标对象是一样的。

#### 有几种不同类型的自动代理？

BeanNameAutoProxyCreator

DefaultAdvisorAutoProxyCreator

Metadata autoproxying

#### 什么是织入。什么是织入应用的不同点？

织入是将切面和到其他应用类型或对象连接或创建一个被通知对象的过程。

织入可以在编译时，类加载时，或运行时完成。

#### 解释基于XML Schema方式的切面实现。

在这种情况下，切面由常规类以及基于XML的配置实现。

#### 解释基于注解的切面实现

在这种情况下(基于@AspectJ的实现)，涉及到的切面声明的风格与带有java5标注的普通java类一致。


#### 什么是Spring的MVC框架？

Spring 配备构建Web 应用的全功能MVC框架。Spring可以很便捷地和其他MVC框架集成，如Struts，Spring 的MVC框架用控制反转把业务对象和控制逻辑清晰地隔离。它也允许以声明的方式把请求参数和业务对象绑定。

#### DispatcherServlet

Spring的MVC框架是围绕DispatcherServlet来设计的，它用来处理所有的HTTP请求和响应。

#### WebApplicationContext

WebApplicationContext 继承了ApplicationContext  并增加了一些WEB应用必备的特有功能，它不同于一般的ApplicationContext ，因为它能处理主题，并找到被关联的servlet。

#### 什么是Spring MVC框架的控制器？

控制器提供一个访问应用程序的行为，此行为通常通过服务接口实现。控制器解析用户输入并将其转换为一个由视图呈现给用户的模型。Spring用一个非常抽象的方式实现了一个控制层，允许用户创建多种用途的控制器。

#### @Controller 注解

该注解表明该类扮演控制器的角色，Spring不需要你继承任何其他控制器基类或引用Servlet API。

#### @RequestMapping 注解

该注解是用来映射一个URL到一个类或一个特定的方处理法上。

----
### Mybatis基础
#### #{...} 和 ${...} 的区别
MyBatis将 #{…} 解释为JDBC prepared statement 的一个参数标记。而将 ${…} 解释为字符串替换。前者可以防止SQL注入。


----

### session创建，session作用

### https 协议(注意是S)


### 加密签名、证书等 (反正就是与通信安全有关的)

### 前端的jQuery+ajax等与后台数据传输