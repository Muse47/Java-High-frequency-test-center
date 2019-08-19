# Java 并发



[TOC]



## 1、synchronized 的实现原理以及锁优化？

### 1.synchronized内部实现原理

synchronized关键字在应用层的语义是可以把任何一个非null对象作为锁，当synchronized作用在方法上时，锁住的是对象实例(this),作用在静态方法上锁住的就是对象对应的Classs实例，由于Class实例存在于永久代，因此静态方法锁相当于类的一个全局锁，当synchronized作用在一个对象实例上，锁住的就是一个代码块
ps：在HotSpot JVM中 锁被称作对象监视器

当有多个线程同时请求某个对象监视器时，对象监视器会设置几种状态来区分请求的线程：

> Contention List：所有请求锁的线程被首先放置在该竞争队列中
> Entry List：Contention List 中有机会获得锁的线程被放置到Entry List
> Wait Set：调用wait()方法被阻塞的线程被放置到Wait Set中
> OnDeck：任何一个时候只能有一个线程竞争锁 该线程称作OnDeck
> Owner：获得锁的线程成为Owner
> !Owner：释放锁的线程

转换关系如下图：

![](http://static.open-open.com/lib/uploadImg/20121109/20121109112521_220.jpg)

新请求锁的线程被首先加入到Contention List中，当某个拥有锁定线程(Owner状态)调用unlock之后，如果发现Entry List为空就从ContentionList中移动线程到Entry List中
Contention List和Entry List的实现方式

**1.Contention List虚拟队列**

Contention List并不是一个真正的Queue，而是一个虚拟队列，原因是Contention List是由Node和next指针逻辑构成，并不存在一个Queue的数据结构。Contention List是一个后进先出的队列，每次添加Node时都会在队头进行，通过CAS改变第一个节点的指针在新增节点，同时设置新增节点的next指向后继节点，而取线程操作发生在队尾。

只有Owner线程才能从队尾取元素，线程出队操作无竞争，避免CAS的ABA问题

![](http://static.open-open.com/lib/uploadImg/20121109/20121109112521_981.jpg)

**2.Entry List**

EntryList与ContentionList逻辑上同属等待队列，ContentionList会被线程并发访问，为了降低对 ContentionList队尾的争用，而建立EntryList。Owner线程在unlock时会从Contention List中迁移线程到Entry List，并会指定Entry List中某个线程(一般是第一个)为OnDeck线程。Owner线程并不是把锁传递给 OnDeck线程，只是把竞争锁的权利交给OnDeck，OnDeck线程需要重新竞争锁。这样做虽然牺牲了一定的公平性，但极大的提高了整体吞吐量，在 Hotspot中把OnDeck的选择行为称之为“竞争切换”

OnDeck线程在获得锁后变成Owner线程，无法获得锁则会继续留在Entry List中，但是在Entry List中的位置不会发生改变。如果Owner线程被wait方法阻塞，就转移到WaitSet队列；如果在某个时刻被notify/notifyAll唤醒就会再次被转移到Entry List中

### 2.sychronized中的锁机制

介绍锁机制之前先介绍看一下同步的原理

**同步的原理**

JVM规范规定JVM基于进入和退出Monitor对象来实现方法同步和代码块同步，但两者的实现细节不一样。

代码块同步是使用monitorenter和monitorexit指令实现，而方法同步是使用另外一种方式实现的，细节在JVM规范里并没有详细说明，但是方法的同步同样可以使用这两个指令来实现。

monitorenter指令是在编译后插入到同步代码块的开始位置，而monitorexit是插入到方法结束处和异常处， JVM要保证每个monitorenter必须有对应的monitorexit与之配对。

任何对象都有一个 monitor 与之关联，当且一个monitor 被持有后，它将处于锁定状态。线程执行到 monitorenter 指令时，将会尝试获取对象所对应的 monitor 的所有权，即尝试获得对象的锁。

**java对象头的概念**

java对象的内存布局包括对象头，数据和填充数据

数组类型的对象头使用3个字宽存储，非数组类型使用2个字宽存储，一个字宽等于四字节（32位）

![](https://img-blog.csdn.net/20170916092254610?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvVGhvdXNhX0hv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

Java对象头里的Mark Word里默认存储对象的HashCode，分代年龄和锁标记位

![](https://img-blog.csdn.net/20170916092334618?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvVGhvdXNhX0hv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

在运行期间随着锁标志位的变化存储的数据也会变化

![](https://img-blog.csdn.net/20170916092431891?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvVGhvdXNhX0hv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

64位JVM下Mark Word大小的64位的 存储结构如下

![](https://img-blog.csdn.net/20170916092509603?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvVGhvdXNhX0hv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

偏向锁的锁标记位和无锁是一样的，都是01，但是有单独一位偏向标记设置是否偏向锁。

轻量级锁00，重量级锁10，GC标记11，无锁 01.

**1.自旋锁 Spin Lock**

处于Contention List，Entry List和Wait Set中的线程均属于阻塞状态，阻塞操作由操作系统完成（在Linux系统下通过pthread_mutex_lock函数），线程被阻塞后进入内核调度状态，这个会导致在用户态和内核态之间来回切换，严重影响锁的性能。

解决上述问题的方法就是自旋，原理是：
当发生争用时，若Owner线程能在很短的时间内释放锁，则那些正在争用线程可以稍微等等（自旋），在Owner线程释放锁之后，争用线程可能会立即获得锁，避免了系统阻塞

但是Owner运行的时间可能会超出临界值，争用线程自旋一段时间无法获得锁的话会停止自旋进入阻塞状态。
因此自旋锁对于执行时间很短的代码块有性能提高。

线程自旋的时候可以执行几次for循环，可以执行几条空的汇编指令，目的是占着CPU不放，等待获取锁的机会。
因此自旋的时间很重要，如果过长会影响整体性能，过短达不到延迟阻塞的目的。HotSpot认为最佳的时间是一个线程上下文切换的时间，但是目前只实现了通过汇编暂停集合CPU周期。

其他自旋锁的优化：

> 如果平均负载小于CPU的个数则一直自旋
> 如果超过CPU个数一半个线程正在自旋，则后面的线程会直接阻塞
> 如果正在自旋的线程发现Owner发生了变化则延迟自旋时间（自旋计数）或进入阻塞
> 如果CPU处于节点模式就停止自旋
> 自旋时间的最坏情况是CPU的存储延迟（CPU A存储了一个数据，到CPU B得知这个数据直接的时间差）
> 自旋时会适当放弃线程优先级之间的差异

那synchronized实现何时使用了自旋锁？
答案是在线程进入ContentionList时，也即第一步操作前。
线程在进入等待队列时首先进行自旋尝试获得锁，如果不成功再进入等待队列。这对那些已经在等待队列中的线程来说，稍微显得不公平。
还有一个不公平的地方是自旋线程可能会抢占了Ready线程的锁。自旋锁由每个监视对象维护，每个监视对象一个

**2.偏向锁(Biased Lock)**

主要解决无竞争下的锁性能问题

按照之前HotSpot设计，每次加锁/解锁都会涉及到一些CAS操作（比如等待队列的CAS操作）。CAS操作会延迟本地调用，因此偏向锁的想法是一旦线程第一次获得了监视对象，之后让监视对象偏向这个线程，之后的多次调用可以避免CAS操作。如果在运行过程中，遇到了其他线程抢占锁，则持有偏向锁的线程会被挂起，JVM会尝试消除它身上的偏向锁，把锁恢复到标准的轻量级锁。

流程是这样的 偏向锁->轻量级锁->重量级锁

简单的加锁机制：

每个锁都关联一个请求计数器和一个占有她的线程，当请求计数器为0时，这个锁可以被认为是unheld的，当一个线程请求一个unheld的锁时，JVM记录锁的拥有者，并把锁的请求计数加1，如果同一个线程再次请求这个锁是，请求计数器就会加一，当线程退出synchronized块时，计数器减一。当计数器为0时，释放锁。

偏向锁的流程

在锁对象的对象头中有一个ThreadId字段，如果字段是空，第一次获取锁的时候就把自身的ThreadId写入到锁的ThreadId字段内，把锁内的是否是偏向锁状态位置设置为1。下次获取锁的时候，直接查看ThreadId是否和自身线程Id一致，如果一致就认为当前线程已经取得了锁无需再次获取锁，略过了轻量级锁和重量级锁的加锁阶段，提高了效率。

但是偏向锁也有一个问题，就是当锁有竞争关系的时候，需要解除偏向锁，使锁进入竞争的状态

![](https://images2015.cnblogs.com/blog/899685/201610/899685-20161025102843468-151954717.png)

对于偏向锁的抢占问题，一旦偏向锁冲突，双方都会升级会轻量级锁。之后就会进入轻量级的锁状态

![](https://images2015.cnblogs.com/blog/899685/201610/899685-20161025103757531-1715338870.jpg)

偏向锁使用的是一种等到竞争出现才释放锁的机制，所以在其他线程尝试获取竞争偏向锁时，持有偏向锁的线程才会释放锁，释放锁需要等到全局安全点(在该时间点上没有字节码在执行)

消除偏向锁的过程是：
先暂停偏向锁的线程，尝试直接切换，如果不成功，就继续运行，并且标记对象不适合偏向锁，锁升级成轻量级锁。

关闭偏向锁：
偏向锁在jdk6和7中是默认开启的，但是总是在程序启动几秒钟后才激活
可以使用JVM参数来关闭延迟-XX:BiasedLockingStartupDelay=0
同时也可以使用参数来关闭偏向锁-XX:-UseBiasedLocking=false

**轻量级锁**

加锁流程：

线程在执行同步块之前，JVM会在当前线程的栈帧中创建用于存储锁记录的空间，并把对象头中的Mark Word复制到锁记录空间中，官方称为Displaced Mark Word，然后线程尝试持有CAS把对象头中的Mark Word替换成指向锁记录的指针，如果成功，当前线程获得锁，如果失败表示有其他线程竞争锁，当前线程尝试使用自旋来获取锁，获取失败就升级成重量级锁。

解锁流程：

会使用CAS操作来把Displaced Mark Word替换回对象头，如果成功表示没有竞争发生，如果失败，升级成重量级锁。

![](https://images2015.cnblogs.com/blog/899685/201610/899685-20161025111821296-1590704619.png)

**锁的优缺点比较**

偏向锁：
优点：加锁解锁不需要额外的消耗，和执行非同步方法比仅存在纳秒级的差距
缺点：如果线程间存在锁竞争，会带来额外的锁撤销的消耗
适用场景：适合只有一个线程访问同步快的场景

轻量级锁：
优点：竞争的线程不会阻塞，提高程序的响应速度
缺点：如果始终得不到锁竞争的线程使用自旋会消耗CPU
适用场景：追求响应时间 同步块执行速度非常快

重量级锁：
优点：线程竞争不适用自旋 不会消耗CPU
缺点：线程阻塞 响应时间缓慢
适用场景：追求吞吐量 同步块执行时间长

**ps:偏向锁和轻量级锁理念上的区别：**

    轻量级锁：在无竞争的情况下使用CAS操作去消除同步使用的互斥量
    偏向锁：在无竞争的情况下把整个同步都消除掉，连CAS操作都不做了

![](https://images2015.cnblogs.com/blog/899685/201610/899685-20161025114738015-1389910806.jpg)

### 总结

在jdk1.6中对锁的实现引入了大量的优化，如锁粗化（Lock Coarsening）、锁消除（Lock Elimination）、轻量级锁（Lightweight Locking）、
偏向锁（Biased Locking）、适应性自旋（Adaptive Spinning）等技术来减少锁操作的开销。

> 锁粗化(Lock Coarsening) 减少不必要的紧连在一起的unlock,lock操作，将多个连续的锁扩展成一个范围更大的锁
> 锁消除(Lock Elimination) 通过运行时JIT编译器的逃逸分析来消除一些没有在当前同步快以外被其他线程共享的数据的锁的保护，通过逃逸分析也可以在线程本地Stack上进行对象空间的分配（减少堆上上GC开销）
> 偏向锁(Biased Locking) 是为了在无锁竞争的情况下避免在锁获取过程中执行不必要的CAS原子指令，因为CAS原子指令虽然相对于重量级锁来说开销比较小但还是存在非常可观的本地延迟
> 轻量级锁 (Lightweight Locking) 这种锁实现的背后基于这样一种假设，即在真实的情况下我们程序中的大部分同步代码一般都处于无锁竞争状态（即单线程执行环境），在无锁竞争的情况下完全可以避免调用操作系统层面的重量级互斥锁，取而代之的是在monitorenter和monitorexit中只需要依靠一条CAS原子指令就可以完成锁的获取及释放。当存在锁竞争的情况下，执行CAS指令失败的线程将调用操作系统互斥锁进入到阻塞状态，当锁被释放的时候被唤醒
> 适应性自旋(Adaptive Spinning) 当线程在获取轻量级锁的过程中执行CAS操作失败时，在进入与monitor相关联的操作系统重量级锁
> （mutex semaphore）前会进入忙等待（Spinning）然后再次尝试，当尝试一定的次数后如果仍然没有成功则调用与该monitor关联的semaphore（即互斥锁），进入到阻塞状态。

对于synchronized加锁的完整过程描述：

> 检查Mark Word里存放的是否是自身的ThreadId，如果是，表示当前线程处于偏向锁，无需加锁就可获取临界资源
> 如果不是自身的ThreadId，锁升级，使用CAS来进行切换，新的线程根据MarkWord里现有的ThreadId，通知之前线程暂停，之前线程把MarkWord的内容设置为空
> 两个线程都把对象的HashCode复制到自己新建的用于存储锁的记录空间，接着开始通过CAS操作把共享对象的MarkWord的内容修改为自己新建的记录空间的地址的方式竞争MarkWord
> 成功执行CAS的获得资源，失败的进入自旋
> 自旋在线程在自旋过程中，成功获得资源则整个状态依然处于轻量级的锁状态
> 如果自旋失败进入重量级锁的状态，自旋的线程进行阻塞，等待之前的线程完成并唤醒自己

最后说一下CAS操作：

CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。

下面是sun.misc.Unsafe类的compareAndSwapInt()方法的源代码：

public final native boolean compareAndSwapInt(Object o, long offset,  int expected,int x);

​    

可以看到这是个本地方法调用。这个本地方法在openjdk中依次调用的c++代码为：unsafe.cpp，atomic.cpp和atomicwindowsx86.inline.hpp。

对于32位/64位的操作应该是原子的：

奔腾6和最新的处理器能自动保证单处理器对同一个缓存行里进行16/32/64位的操作是原子的，但是复杂的内存操作处理器不能自动保证其原子性，比如跨总线宽度，跨多个缓存行，跨页表的访问。但是处理器提供总线锁定和缓存锁定两个机制来保证复杂内存操作的原子性。

缺点主要是ABA问题，循环时间开销大和只能保证一个共享变量

1.ABA问题。因为CAS需要在操作值的时候检查下值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。

ABA问题的解决思路就是使用版本号
在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么A－B－A 就会变成1A-2B－3A

从Java1.5开始JDK的atomic包里提供了一个类AtomicStampedReference来解决ABA问题。
这个类的compareAndSet方法作用是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，
则以原子方式将该引用和该标志的值设置为给定的更新值。

2.循环时间开销大 自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。

如果JVM能支持处理器提供的pause指令那么效率会有一定的提升，pause指令有两个作用，第一它可以延迟流水线执行指令（de-pipeline）,使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零。
第二它可以避免在退出循环的时候因内存顺序冲突（memory order violation）而引起CPU流水线被清空（CPU pipeline flush），从而提高CPU的执行效率。

3.只能保证一个共享变量的原子操作

解决方法：
把多个共享变量合并成一个共享变量来操作。比如有两个共享变量i＝2,j=a，合并一下ij=2a，然后用CAS来操作ij。
从Java1.5开始JDK提供了AtomicReference类来保证引用对象之间的原子性，也可以把多个变量放在一个对象里来进行CAS操作。

## 2、volatile 的实现原理？

通过前面一章我们了解了synchronized是一个重量级的锁，虽然JVM对它做了很多优化，而下面介绍的volatile则是轻量级的synchronized。如果一个变量使用volatile，则它比使用synchronized的成本更加低，因为它不会引起线程上下文的切换和调度。Java语言规范对volatile的定义如下：

> Java编程语言允许线程访问共享变量，为了确保共享变量能被准确和一致地更新，线程应该确保通过排他锁单独获得这个变量。

上面比较绕口，通俗点讲就是说一个变量如果用volatile修饰了，则Java可以确保所有线程看到这个变量的值是一致的，如果某个线程对volatile修饰的共享变量进行更新，那么其他线程可以立马看到这个更新，这就是所谓的线程可见性。

volatile虽然看起来比较简单，使用起来无非就是在一个变量前面加上volatile即可，但是要用好并不容易（LZ承认我至今仍然使用不好，在使用时仍然是模棱两可）。
内存模型相关概念

理解volatile其实还是有点儿难度的，它与Java的内存模型有关，所以在理解volatile之前我们需要先了解有关Java内存模型的概念，这里只做初步的介绍，后续LZ会详细介绍Java内存模型。
操作系统语义

计算机在运行程序时，每条指令都是在CPU中执行的，在执行过程中势必会涉及到数据的读写。我们知道程序运行的数据是存储在主存中，这时就会有一个问题，读写主存中的数据没有CPU中执行指令的速度快，如果任何的交互都需要与主存打交道则会大大影响效率，所以就有了CPU高速缓存。CPU高速缓存为某个CPU独有，只与在该CPU运行的线程有关。

有了CPU高速缓存虽然解决了效率问题，但是它会带来一个新的问题：数据一致性。在程序运行中，会将运行所需要的数据复制一份到CPU高速缓存中，在进行运算时CPU不再也主存打交道，而是直接从高速缓存中读写数据，只有当运行结束后才会将数据刷新到主存中。举一个简单的例子：


    i++i++

当线程运行这段代码时，首先会从主存中读取i( i = 1)，然后复制一份到CPU高速缓存中，然后CPU执行 + 1 （2）的操作，然后将数据（2）写入到告诉缓存中，最后刷新到主存中。其实这样做在单线程中是没有问题的，有问题的是在多线程中。如下：

假如有两个线程A、B都执行这个操作（i++），按照我们正常的逻辑思维主存中的i值应该=3，但事实是这样么？分析如下：

两个线程从主存中读取i的值（1）到各自的高速缓存中，然后线程A执行+1操作并将结果写入高速缓存中，最后写入主存中，此时主存i==2,线程B做同样的操作，主存中的i仍然=2。所以最终结果为2并不是3。这种现象就是缓存一致性问题。

解决缓存一致性方案有两种：

> 通过在总线加LOCK#锁的方式
> 通过缓存一致性协议

但是方案1存在一个问题，它是采用一种独占的方式来实现的，即总线加LOCK#锁的话，只能有一个CPU能够运行，其他CPU都得阻塞，效率较为低下。

第二种方案，缓存一致性协议（MESI协议）它确保每个缓存中使用的共享变量的副本是一致的。其核心思想如下：当某个CPU在写数据时，如果发现操作的变量是共享变量，则会通知其他CPU告知该变量的缓存行是无效的，因此其他CPU在读取该变量时，发现其无效会重新从主存中加载数据。

![](https://images2015.cnblogs.com/blog/381060/201702/381060-20170208174530432-687253404.jpg)
Java内存模型

上面从操作系统层次阐述了如何保证数据一致性，下面我们来看一下Java内存模型，稍微研究一下Java内存模型为我们提供了哪些保证以及在Java中提供了哪些方法和机制来让我们在进行多线程编程时能够保证程序执行的正确性。

在并发编程中我们一般都会遇到这三个基本概念：原子性、可见性、有序性。我们稍微看下volatile
原子性

> 原子性：即一个操作或者多个操作 要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。

原子性就像数据库里面的事务一样，他们是一个团队，同生共死。其实理解原子性非常简单，我们看下面一个简单的例子即可：


​    	
​    i = 0;---1
​    j = i ;---2
​    i++;---3
​    i = j + 1;---4

上面四个操作，有哪个几个是原子操作，那几个不是？如果不是很理解，可能会认为都是原子性操作，其实只有1才是原子操作，其余均不是。

    1—在Java中，对基本数据类型的变量和赋值操作都是原子性操作；
    2—包含了两个操作：读取i，将i值赋值给j
    3—包含了三个操作：读取i值、i + 1 、将+1结果赋值给i；
    4—同三一样

在单线程环境下我们可以认为整个步骤都是原子性操作，但是在多线程环境下则不同，Java只保证了基本数据类型的变量和赋值操作才是原子性的（注：在32位的JDK环境下，对64位数据的读取不是原子性操作*，如long、double）。要想在多线程环境下保证原子性，则可以通过锁、synchronized来确保。

> volatile是无法保证复合操作的原子性

可见性

> 可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

在上面已经分析了，在多线程环境下，一个线程对共享变量的操作对其他线程是不可见的。

Java提供了volatile来保证可见性。

当一个变量被volatile修饰后，表示着线程本地内存无效，当一个线程修改共享变量后他会立即被更新到主内存中，当其他线程读取共享变量时，它会直接从主内存中读取。
当然，synchronize和锁都可以保证可见性。
有序性

> 
> 有序性：即程序执行的顺序按照代码的先后顺序执行。

在Java内存模型中，为了效率是允许编译器和处理器对指令进行重排序，当然重排序它不会影响单线程的运行结果，但是对多线程会有影响。

Java提供volatile来保证一定的有序性。最著名的例子就是单例模式里面的DCL（双重检查锁）。这里LZ就不再阐述了。
剖析volatile原理

JMM比较庞大，不是上面一点点就能够阐述的。上面简单地介绍都是为了volatile做铺垫的。

> volatile可以保证线程可见性且提供了一定的有序性，但是无法保证原子性。在JVM底层volatile是采用“内存屏障”来实现的。

上面那段话，有两层语义
> 
> 保证可见性、不保证原子性
> 禁止指令重排序

第一层语义就不做介绍了，下面重点介绍指令重排序。

在执行程序时为了提高性能，编译器和处理器通常会对指令做重排序：

> 编译器重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序；
> 处理器重排序。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序；

指令重排序对单线程没有什么影响，他不会影响程序的运行结果，但是会影响多线程的正确性。既然指令重排序会影响到多线程执行的正确性，那么我们就需要禁止重排序。那么JVM是如何禁止重排序的呢？这个问题稍后回答，我们先看另一个原则happens-before，happen-before原则保证了程序的“有序性”，它规定如果两个操作的执行顺序无法从happens-before原则中推到出来，那么他们就不能保证有序性，可以随意进行重排序。其定义如下：

> 同一个线程中的，前面的操作 happen-before 后续的操作。（即单线程内按代码顺序执行。但是，在不影响在单线程环境执行结果的前提下，编译器和处理器可以进行重排序，这是合法的。换句话说，这一是规则无法保证编译重排和指令重排）。
> 监视器上的解锁操作 happen-before 其后续的加锁操作。（Synchronized 规则）
> 对volatile变量的写操作 happen-before 后续的读操作。（volatile 规则）
> 线程的start() 方法 happen-before 该线程所有的后续操作。（线程启动规则）
> 线程所有的操作 happen-before 其他线程在该线程上调用 join 返回成功后的操作。
> 如果 a happen-before b，b happen-before c，则a happen-before c（传递性）。

我们着重看第三点volatile规则：对volatile变量的写操作 happen-before 后续的读操作。为了实现volatile内存语义，JMM会重排序，其规则如下：

对happen-before原则有了稍微的了解，我们再来回答这个问题JVM是如何禁止重排序的？

![](https://images2015.cnblogs.com/blog/381060/201702/381060-20170208174536119-1181208627.jpg)

观察加入volatile关键字和没有加入volatile关键字时所生成的汇编代码发现，加入volatile关键字时，会多出一个lock前缀指令。lock前缀指令其实就相当于一个内存屏障。内存屏障是一组处理指令，用来实现对内存操作的顺序限制。volatile的底层就是通过内存屏障来实现的。下图是完成上述规则所需要的内存屏障：

volatile暂且下分析到这里，JMM体系较为庞大，不是三言两语能够说清楚的，后面会结合JMM再一次对volatile深入分析。

![](https://images2015.cnblogs.com/blog/381060/201702/381060-20170208174537963-1251333114.jpg)

### 总结

volatile看起来简单，但是要想理解它还是比较难的，这里只是对其进行基本的了解。volatile相对于synchronized稍微轻量些，在某些场合它可以替代synchronized，但是又不能完全取代synchronized，只有在某些场合才能够使用volatile。使用它必须满足如下两个条件：

> 对变量的写操作不依赖当前值；
> 该变量没有包含在具有其他变量的不变式中。
> 
> volatile经常用于两个两个场景：状态标记两、double check



## 3、Java 的信号灯？

### 1、Semaphore概念

Semaphore是Java1.5之后提供的一种同步工具，Semaphore可以维护访问自身线程个数，并提供了同步机制。使用Semaphore可以控制同时访问资源的线程个数，通过 acquire() 获取一个许可，如果没有就等待，而release() 释放一个许可。

Semaphore实现的功能就类似厕所有5个坑，假如有10个人要上厕所，那么同时只能有多少个人去上厕所呢？同时只能有5个人能够占用，当5个人中 的任何一个人让开后，其中等待的另外5个人中又有一个人可以去占用了。另外等待的5个人中可以是随机获得优先机会，也可以是按照先来后到的顺序获得机会，这取决于构造Semaphore对象时传入的参数选项。单个信号量的Semaphore对象可以实现互斥锁的功能，并且可以是由一个线程获得了“锁”，再由另一个线程释放“锁”，这可应用于死锁恢复的一些场合，所以单个信号量的Semaphore对象的功能就和synchronized实现互斥是共同的

### 2、功能扩展

1、Semaphore往往结合线程池使用，比如建立一个固定大小为10的线程池，最多线程并发数为10个，当你提交20个任务到线程池的时候，线程池会安排这10个线程优先接待20个任务中的10个，当优先安排的10个中有完成的会去接待剩下的10个任务中的某一个任务，直到执行完所有任务为止。但是如果此时引入了Semaphore对象，所传的值是5的时候，那么这线程池中10个线程只有5个能够并发执行，此时就做到了限量访问的作用。

2、当Semaphore构造方法中传入的参数是1的时候，此时线程并发数最多是1个，即是线程安全的，这种方式也可以做到现场互斥。Java实现互斥线程同步有三种方式synchronized、lock 、单Semaphore


3、Semaphore的使用demo如下

    public class SemahoreDemo {
    	
    	public static void main(String[] args) {
    		ExecutorService executorService = Executors.newCachedThreadPool();
    		
    		final Semaphore semaphore = new Semaphore(1,true);
    		
    		for(int i=0;i<10;i++){
    			Runnable runnable = new Runnable() {
    				@Override
    				public void run() {
    					try {
    						semaphore.acquire();
    					} catch (InterruptedException e) {
    						e.printStackTrace();
    					}
    					
    					System.err.println("线程"+Thread.currentThread().getName()+"进入，已有"+(3-semaphore.availablePermits())+"并发");
                        
    					try {
    						Thread.sleep((long)(Math.random()*1000));
    					} catch (InterruptedException e) {
    						e.printStackTrace();
    					}
    					
    					System.out.println("线程"+Thread.currentThread().getName()+"即将离开");
    					
    					semaphore.release();
    					
    					System.err.println("线程"+Thread.currentThread().getName()+"已经离开"+"当前并发数："+(3-semaphore.availablePermits()));
    				}
    			};
    			
    			executorService.execute(runnable);
    		}
    		
    		executorService.shutdown();
    		
    	}
     
    }

## 4、synchronized 在静态方法和普通方法的区别？

synchronized是Java中的关键字，是一种同步锁。它修饰的对象有以下几种：

> 1. 修饰一个代码块，被修饰的代码块称为同步语句块，其作用的范围是大括号{}括起来的代码，作用的对象是调用这个代码块的对象；
> 2. 修饰一个方法，被修饰的方法称为同步方法，其作用的范围是整个方法，作用的对象是调用这个方法的对象；
> 3. 修改一个静态的方法，其作用的范围是整个静态方法，作用的对象是这个类的所有对象；
> 4. 修改一个类，其作用的范围是synchronized后面括号括起来的部分，作用主的对象是这个类的所有对象。


​     
​    public class SynchronizedTest {
​     
​    private int num;
​     
​    public synchronized void method01(String arg) {
​    try {
​    if("a".equals(arg)){
​    num = 100;
​    System.out.println("tag a set number over");
​    Thread.sleep(1000);
​    }else{
​    num = 200;
​    System.out.println("tag b set number over");
​    }
​     
​    System.out.println("tag = "+ arg + ";num ="+ num);
​    } catch (InterruptedException e) {
​    e.printStackTrace();
​    }
​    }
​     
​    public static void main(String[] args) {
​    final SynchronizedTest m1 = new SynchronizedTest();
​    final SynchronizedTest m2 = new SynchronizedTest();
​     
​    Thread t1 = new Thread(new Runnable() {
​     
    @Override
    public void run() {
    m1.method01("a");
    }
    });
    t1.start();
     
    Thread t2 = new Thread(new Runnable() {
     
    @Override
    public void run() {
    m2.method01("b");
    }
    });
    t2.start();
     
    }
    }

执行输出：


    tag a set number over
    tag b set number over
    tag = b;num =200
    tag = a;num =100

可以看出，两个不同的对象m1和m2的method01()方法执行并没有互斥，因为这里synchronized是分别持有两个对象的锁。如果要想m1,m2两个对象竞争同一个锁，则需要在method01()上加上static修饰。如下：



    public class SynchronizedTest {
     
    private static int  num;
     
    public static synchronized void method01(String arg) {
    try {
    if("a".equals(arg)){
    num = 100;
    System.out.println("tag a set number over");
    Thread.sleep(1000);
    }else{
    num = 200;
    System.out.println("tag b set number over");
    }
     
    System.out.println("tag = "+ arg + ";num ="+ num);
    } catch (InterruptedException e) {
    e.printStackTrace();
    }
    }
     
    public static void main(String[] args) {
    final SynchronizedTest m1 = new SynchronizedTest();
    final SynchronizedTest m2 = new SynchronizedTest();
     
    Thread t1 = new Thread(new Runnable() {
     
    @Override
    public void run() {
    m1.method01("a");
    }
    });
    t1.start();
     
    Thread t2 = new Thread(new Runnable() {
     
    @Override
    public void run() {
    m2.method01("b");
    }
    });
    t2.start();
     
    }
    }

输出结果：


    tag a set number over
    tag = a;num =100
    tag b set number over
    tag = b;num =200

###小结：

synchronized修饰不加static的方法，锁是加在单个对象上，不同的对象没有竞争关系；修饰加了static的方法，锁是加载类上，这个类所有的对象竞争一把锁。

## 5、怎么实现所有线程在等待某个事件的发生才会去执行？

java里面实现这个有两个办法，**countdownlatch和cyclicbarrier。**

cyclicbarrier可以重复使用，它允许一组线程相互等待，直到达到某个公共屏障点。

cyclicbarrier不会阻塞主线程，只会阻塞子线程。

countdownlatch不可以重复使用，会阻塞主线程。主线程调用await方法，主线程阻塞。子线程调用countdown方法，触发计数。
countdownlatch内部是实现了AQS，初始化的时候，new  CountDownLatch(n);将AQS的state设置为n。

await方法->acquireSharedInterruptibly->doAcquireSharedInterruptibly->shouldParkAfterFailedAcquire&parkAndCheckInterrupt->LockSupport.park.根据此调用链，会将当前线程阻塞。shouldParkAfterFailedAcquire方法相当于重新构造阻塞队列，对于前驱节点的waitstate大于0的删除，不是SIGNAL的赋值为SIGNAL。

countdown是个解锁的过程，每个子线程执行一次countdown，state就减一，state等于0时，就会执行如下代码

![](https://upload-images.jianshu.io/upload_images/6256495-f426de30459aee64.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)


<br>

![](https://upload-images.jianshu.io/upload_images/6256495-8327af9baf306907.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

unparkSuccessor方法的参数是头结点，此时的队列应该是只有一个等待节点，就是主线程。s不等于null，s是主线程。

cyclicbarrier内部使用Lock，每一个子线程执行await，计数减一，当最后一个子线程的计数为0时，会执行cyclicbarrier构造函数中的Runable参数的run方法。

**可以使用信号量(Seamphone)**

Windows 和 Linux 的基本概念是一样的。

信号量相当于一个原子计数器，等待的线程数就是计数器的最大数。等待线程等待时尝试让计数器减1，成功就继续执行，失败就等待。

执行线程在需要唤醒等到线程时，让计数器等于等待线程数(release操作），这样每个等待的线程都可以成功减1，进而继续执行了。

所有线程都等待（wait)这个信号量，一旦某个事件发生，则执行线程就释放这个信号量（release)。 

## 6、CAS？CAS 有什么缺陷，如何解决？

前言
CAS（Compare and Swap），即比较并替换，实现并发算法时常用到的一种技术，Doug lea大神在java同步器中大量使用了CAS技术，鬼斧神工的实现了多线程执行的安全性。
CAS的思想很简单：三个参数，一个当前内存值V、旧的预期值A、即将更新的值B，当且仅当预期值A和内存值V相同时，将内存值修改为B并返回true，否则什么都不做，并返回false。
问题
一个n++的问题。

    public class Case {
    
    public volatile int n;
    
    public void add() {
    n++;
    }
    }

通过javap -verbose Case看看add方法的字节码指令

    public void add();
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=1, args_size=1
     0: aload_0   
     1: dup   
     2: getfield  #2  // Field n:I
     5: iconst_1  
     6: iadd  
     7: putfield  #2  // Field n:I
    10: return    

n++被拆分成了几个指令：

    执行getfield拿到原始n；
    执行iadd进行加1操作；
    执行putfield写把累加后的值写回n；

通过volatile修饰的变量可以保证线程之间的可见性，但并不能保证这3个指令的原子执行，在多线程并发执行下，无法做到线程安全，得到正确的结果，那么应该如何解决呢？
如何解决
在add方法加上synchronized修饰解决。

    public class Case {
    
    public volatile int n;
    
    public synchronized void add() {
    n++;
    }
    }

这个方案当然可行，但是性能上差了点，还有其它方案么？
再来看一段代码

    public int a = 1;
    public boolean compareAndSwapInt(int b) {
    if (a == 1) {
    a = b;
    return true;
    }
    return false;
    }

如果这段代码在并发下执行，会发生什么？

假设线程1和线程2都过了a==1的检测，都准备执行对a进行赋值，结果就是两个线程同时修改了变量a，显然这种结果是无法符合预期的，无法确定a的最终值。

解决方法也同样暴力，在compareAndSwapInt方法加锁同步，变成一个原子操作，同一时刻只有一个线程才能修改变量a。

除了低性能的加锁方案，我们还可以使用JDK自带的CAS方案，在CAS中，比较和替换是一组原子操作，不会被外部打断，且在性能上更占有优势。

下面以AtomicInteger的实现为例，分析一下CAS是如何实现的。

    public class AtomicInteger extends Number implements java.io.Serializable {
    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;
    
    static {
    try {
    valueOffset = unsafe.objectFieldOffset
    (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
    }
    
    private volatile int value;
    public final int get() {return value;}
    }


> Unsafe，是CAS的核心类，由于Java方法无法直接访问底层系统，需要通过本地（native）方法来访问，Unsafe相当于一个后门，基于该类可以直接操作特定内存的数据。
> 
> 变量valueOffset，表示该变量值在内存中的偏移地址，因为Unsafe就是根据内存偏移地址获取数据的。
> 
> 变量value用volatile修饰，保证了多线程之间的内存可见性。

看看AtomicInteger如何实现并发下的累加操作：

    public final int getAndAdd(int delta) {
    return unsafe.getAndAddInt(this, valueOffset, delta);
    }
    
    //unsafe.getAndAddInt
    public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
    var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
    return var5;
    }

假设线程A和线程B同时执行getAndAdd操作（分别跑在不同CPU上）：

> AtomicInteger里面的value原始值为3，即主内存中AtomicInteger的value为3，根据Java内存模型，线程A和线程B各自持有一份value的副本，值为3。
> 
> 线程A通过getIntVolatile(var1, var2)拿到value值3，这时线程A被挂起。
> 
> 线程B也通过getIntVolatile(var1, var2)方法获取到value值3，运气好，线程B没有被挂起，并执行compareAndSwapInt方法比较内存值也为3，成功修改内存值为2。
> 
> 这时线程A恢复，执行compareAndSwapInt方法比较，发现自己手里的值(3)和内存的值(2)不一致，说明该值已经被其它线程提前修改过了，那只能重新来一遍了。
> 
> 重新获取value值，因为变量value被volatile修饰，所以其它线程对它的修改，线程A总是能够看到，线程A继续执行compareAndSwapInt进行比较替换，直到成功。

整个过程中，利用CAS保证了对于value的修改的并发安全，继续深入看看Unsafe类中的compareAndSwapInt方法实现。


    public final native boolean compareAndSwapInt(Object paramObject, long paramLong, int paramInt1, int paramInt2);

Unsafe类中的compareAndSwapInt，是一个本地方法，该方法的实现位于unsafe.cpp中
    
    UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))
      UnsafeWrapper("Unsafe_CompareAndSwapInt");
      oop p = JNIHandles::resolve(obj);
      jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);
      return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
    UNSAFE_END


先想办法拿到变量value在内存中的地址。
通过Atomic::cmpxchg实现比较替换，其中参数x是即将更新的值，参数e是原内存的值。

如果是Linux的x86，Atomic::cmpxchg方法的实现如下：

    inline jint Atomic::cmpxchg (jint exchange_value, volatile jint* dest, jint compare_value) {
      int mp = os::is_MP();
      __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"
    : "=a" (exchange_value)
    : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
    : "cc", "memory");
      return exchange_value;
    }

看到这汇编，内心崩溃 😖

> __asm__表示汇编的开始

> volatile表示禁止编译器优化
> 
> LOCK_IF_MP是个内联函数

    #define LOCK_IF_MP(mp) "cmp $0, " #mp "; je 1f; lock; 1: "

Window的x86实现如下：

    inline jint Atomic::cmpxchg (jint exchange_value, volatile jint* dest, jint compare_value) {
    int mp = os::isMP(); //判断是否是多处理器
    _asm {
    mov edx, dest
    mov ecx, exchange_value
    mov eax, compare_value
    LOCK_IF_MP(mp)
    cmpxchg dword ptr [edx], ecx
    }
    }
    
    // Adding a lock prefix to an instruction on MP machine
    // VC++ doesn't like the lock prefix to be on a single line
    // so we can't insert a label after the lock prefix.
    // By emitting a lock prefix, we can define a label after it.
    #define LOCK_IF_MP(mp) __asm cmp mp, 0  \
       __asm je L0  \
       __asm _emit 0xF0 \
       __asm L0:

> LOCK_IF_MP根据当前系统是否为多核处理器决定是否为cmpxchg指令添加lock前缀。

> 如果是多处理器，为cmpxchg指令添加lock前缀。
> 
> 反之，就省略lock前缀。（单处理器会不需要lock前缀提供的内存屏障效果）

intel手册对lock前缀的说明如下：

> 确保后续指令执行的原子性。
> 在Pentium及之前的处理器中，带有lock前缀的指令在执行期间会锁住总线，使得其它处理器暂时无法通过总线访问内存，很显然，这个开销很大。在新的处理器中，Intel使用缓存锁定来保证指令执行的原子性，缓存锁定将大大降低lock前缀指令的执行开销。
> 
> 禁止该指令与前面和后面的读写指令重排序。
> 
把写缓冲区的所有数据刷新到内存中。

上面的第2点和第3点所具有的内存屏障效果，保证了CAS同时具有volatile读和volatile写的内存语义。

**CAS缺点**

CAS存在一个很明显的问题，即ABA问题。

问题：如果变量V初次读取的时候是A，并且在准备赋值的时候检查到它仍然是A，那能说明它的值没有被其他线程修改过了吗？

如果在这段期间曾经被改成B，然后又改回A，那CAS操作就会误认为它从来没有被修改过。针对这种情况，java并发包中提供了一个带有标记的原子引用类AtomicStampedReference，它可以通过控制变量值的版本来保证CAS的正确性。

## 7、synchronized 和 lock 有什么区别？

1）Lock不是Java语言内置的，synchronized是Java语言的关键字，因此是内置特性。Lock是一个类，通过这个类可以实现同步访问；

2）Lock和synchronized有一点非常大的不同，采用synchronized不需要用户去手动释放锁，当synchronized方法或者synchronized代码块执行完之后，系统会自动让线程释放对锁的占用；而Lock则必须要用户去手动释放锁，如果没有主动释放锁，就有可能导致出现死锁现象。

**区别**

1.用法不一样。synchronized既可以加在方法上，也可以加载特定的代码块上，括号中表示需要锁的对象。而Lock需要显示地指定起始位置和终止位置。synchronzied是托管给jvm执行的，Lock锁定是通过代码实现的。 

2.在性能上来说，如果竞争资源不激烈，两者的性能是差不多的，而当竞争资源非常激烈时（即有大量线程同时竞争），此时Lock的性能要远远优于synchronized。所以说，在具体使用时要根据适当情况选择。 

3.锁的机制不一样。synchronized获得锁和释放的方式都是在块结构中，而且是自动释放锁。而Lock则需要开发人员手动去释放，并且必须在finally块中释放，否则会引起死锁问题的发生。 

4.Lock是一个接口，而synchronized是Java中的关键字,synchronized是内置的语言实现； 

5.synchronized在发生异常时，会自动释放线程占有的锁，因此不会导致死锁现象发生；而Lock在发生异常时，如果没有主动通过unLock()去释放锁，则很可能造成死锁现象，因此使用Lock时需要在finally块中释放锁； 

6.Lock可以让等待锁的线程响应中断，而synchronized却不行，使用synchronized时，等待的线程会一直等待下去，不能够响应中断；通过Lock可以知道有没有成功获取锁，而synchronized却无法办到。Lock可以提高多个线程进行读操作的效率。



## 8、Hashtable 是怎么加锁的 ？

线程安全就是多线程访问时（WEB网页多用户访问一个页面时），采用了加锁机制，当一个线程访问该类的某个数据时，进行保护，其他线程不能进行访问直到该线程读取完，其他线程才可使用。不会出现数据不一致或者数据污染。

Hashtable 表示键/值对的集合，这些键/值对根据键的哈希代码进行组织,它的Key不能为null,Value可以为null，这一点与Hashmap不同（本身不是线程安全的），对于Hashtable它是实现了IDictionary和ICollection接口的，它的key与value都是object类型的，不支持泛型，进行类型转换成需要装箱与拆箱（boxing,unboxing），这在性能肯定会有一些影响，所以，微软这边给出了支持泛型的键值对集合Dictionary，而Dictionary本身也不是线程安全的，我们需要对它加锁(lock)，才能避免多线程环境下产生的一些错误。

下面我们来看一下线程安全的Hashtable代码片断：

            Hashtable ht = Hashtable.Synchronized(new Hashtable());
            ht.Add("ok", null);
            Console.WriteLine(ht["ok"]);

我们在来看一下Dictionary对象，可以使它基类提供的SyncRoot属性，来实现它内部对象的线程安全　　

            lock ((dic as ICollection).SyncRoot)
            {
                dic.Add("ok", "ok value");
            }

下面我们来做一个实例，还是Dictionary的线程安全问题，我们有两个线程，t1和t2，当我们为它加lock之后，t1纯种在进行dic.Ad操作时，t2并不能进行访问

当t1完成add操作后，t2线程才进行执行，这时它就可以改变dic 元素的值了，程序运行正常，但如果没有lock锁机制，t1与 t2线程谁先执行就不确定了，这时，

如果t1先执行，当然没有问题，但如果t2先操作了，程序出现异常，因为dic元素没有被add，所以无法改变其值。

看代码：



            Dictionary<string, string> dic = new Dictionary<string, string>();
    
            Thread t1 = new Thread(() =>
            {
                lock ((dic as ICollection).SyncRoot) //dic对象被保存，处于临界区
                {
                    dic.Add("ok1", "ok value1");//这句先向字典添加
                }
            });
    
            Thread t2 = new Thread(() =>
            {
                lock ((dic as ICollection).SyncRoot)
                {
                    dic["ok1"] = "ok value2";
                }
            });


            t1.Start();
            t2.Start();
            Thread.Sleep(2000);



 

而对于Hashtable来说，如果希望对它进行写加锁，读不加锁，也可以通过lock在代码段时去实现
复制代码

                  Thread t1 = new Thread(() =>
                    {
                        lock (ht.SyncRoot)
                        {
    
                            ht.Add(i, i);
                        }
                    });

## 9、HashMap 的并发问题？

概念1：Rehash的概念？ 
Rehash 是HashMap在扩容时候的一个步骤。

HashMap的容量是有限的。当经过多次元素插入，使得HashMap达到一定饱和度时，Key映射位置发生冲突的几率会逐渐提高。

这时候，HashMap需要扩展它的长度，也就是进行Resize

![](https://img-blog.csdn.net/20171212102003329?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGd1dGxpYW5neHVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

影响发生Resize的因素有两个： 
1.Capacity（HashMap的当前长度–容量） 
HashMap的当前长度。上一期曾经说过，HashMap的长度是2的幂。

2.LoadFactor（负载因子） 
HashMap负载因子，默认值为0.75f。

衡量HashMap是否进行Resize的条件如下： 
HashMap.Size >= Capacity * LoadFactor （默认情况下是 == 原来长度 * 0.75）

HashMap的Resize方法具体做了什么事情？

1.扩容 
创建一个新的Entry空数组，长度是原数组的2倍。

2.ReHash 
遍历原Entry数组，把所有的Entry重新Hash到新数组。为什么要重新Hash呢？因为长度扩大以后，Hash的规则也随之改变。

让我们回顾一下Hash公式： 
index = HashCode（Key） & （Length - 1）

当原数组长度为8时，Hash运算是和111B（代表二进制的7）做与运算；新数组长度为16，Hash运算是和1111B（代表二进制的15）做与运算。Hash结果显然不同

![](https://img-blog.csdn.net/20171212102614687?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGd1dGxpYW5neHVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![](https://img-blog.csdn.net/20171212102634781?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGd1dGxpYW5neHVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

ReHash的Java代码如下：

    /**
     * Transfers all entries from current table to newTable.
     */
    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }
    
    1


注意HashMap在多线程下的Rehash可能会出现什么样的问题呢？

假设一个HashMap已经到了Resize的临界点。此时有两个线程A和B，在同一时刻对HashMap进行Put操作： 
![](https://img-blog.csdn.net/20171212102954873?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGd1dGxpYW5neHVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

此时达到Resize条件，两个线程各自进行Rezie的第一步，也就是扩容：


这时候，两个线程都走到了ReHash的步骤。让我们回顾一下ReHash的代码： 
![](https://img-blog.csdn.net/20171212103144133?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGd1dGxpYW5neHVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

假如此时线程B遍历到Entry3对象，刚执行完红框里的这行代码，线程就被挂起。 
对于线程B来说 
： e = Entry3 next =Entry2 
这时候线程A畅通无阻地进行着Rehash，当ReHash完成后，结果如下（图中的e和next，代表线程B的两个引用）： 
![](https://img-blog.csdn.net/20171212103418062?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGd1dGxpYW5neHVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

直到这一步，看起来没什么毛病。接下来线程B恢复，继续执行属于它自己的ReHash。 
线程B刚才的状态是：e = Entry3 next = Entry2 
![](https://img-blog.csdn.net/20171212103541482?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGd1dGxpYW5neHVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast) 

当执行到上面这一行时，显然 i = 3，因为刚才线程A对于Entry3的hash结果也是3。 

![](https://img-blog.csdn.net/20171212103844512?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGd1dGxpYW5neHVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> 我们继续执行到这两行，Entry3放入了线程B的数组下标为3的位置，并且e指向了Entry2。 
> 此时e和next的指向如下： 
> e =Entry2 next = Entry2 
> 整体情况如下图所示： 
> 
> ![](https://img-blog.csdn.net/20171212104110867?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGd1dGxpYW5neHVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
> 
> 接着是新一轮循环，又执行到红框内的代码行： 
> 
> ![](https://img-blog.csdn.net/20171212104217057?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGd1dGxpYW5neHVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
> 
>     e = Entry2 
>     next = Entry3 
> 
> *注：网上写1.7版本HashMap并发产生环形链表问题都没有考虑内存可见性，如果多线程情况下内存不可见，自然也不会有环形链表问题。 HashMap中没有定义volatile变量，无法保证多线程情况下内存可见性，所以正常来说A线程看到的就是自己本地内存中的链表：3->7而不是线程B中的链表7->3。这里是因为循环过程中在某些情况下调用了hash方法，hash方法调用了sun.misc.Hashing.stringHash32方法，强迫线程A读取主存，从而保持了和线程B的一致。后面才会有环形链表的故事。所以环形链表发生的根本原因是内存可见造成的。（JDK8中对HashMap作出调整，在尾节点添加元素。1.7及以前都是头结点添加元素。）*
> 
> 
> 整体情况如图所示： 
> 
> ![](https://img-blog.csdn.net/20171212104448938?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGd1dGxpYW5neHVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
> 
> 接下来执行下面的三行，用头插法把Entry2插入到了线程B的数组的头结点： 
> 
> ![](https://img-blog.csdn.net/20171212104528287?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGd1dGxpYW5neHVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
> 
> 整体情况如图所示： 
> 
> ![](https://img-blog.csdn.net/20171212104554256?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGd1dGxpYW5neHVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
> 
> 第三次循环开始，又执行到红框的代码： 
> 
> ![](https://img-blog.csdn.net/20171212104633678?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGd1dGxpYW5neHVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
> 
>     e = Entry3 
>     next = Entry3.next = null
> 
> 最后一步，当我们执行下面这一行的时候，见证奇迹的时刻来临了 
> ![](https://img-blog.csdn.net/20171212104750687?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGd1dGxpYW5neHVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
> 
>     newTable[i] = Entry2 
>     e = Entry3 
>     Entry2.next = Entry3 
>     Entry3.next = Entry2 
> 链表出现了环形！ 
> 整体情况如图所示： 
> ![](https://img-blog.csdn.net/20171212104939923?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGd1dGxpYW5neHVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
> 
> 此时，问题还没有直接产生。当调用Get查找一个不存在的Key， 
> 而这个Key的Hash结果恰好等于3的时候，由于位置3带有环形链表，所以程序将会进入死循环！

这种情况，不禁让人联想到一道经典的面试题： 
漫画算法：如何判断链表有环？ 
如何杜绝这种情况？ 
![](https://img-blog.csdn.net/20171212105147056?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGd1dGxpYW5neHVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

总结如下

    1.Hashmap在插入元素过多的时候需要进行Resize， 
    Resize的条件是 HashMap.Size >= Capacity * LoadFactor。
    
    2.Hashmap的Resize包含扩容和ReHash两个步骤，ReHash在并发的情况下可能会形成链表环。
    
    3.Hashmap在并发情况下还会造成元素丢失（这与内存可见性有关，当e.next==null的时候就会结束循环）
    
    4.Hashmap在并发情况下还会造成size不准确（因为在判断是否需要扩容之前会做size++，其实这个时候size实际可能只是增加了1，现在确增加了2）

链表头插法的会颠倒原来一个散列桶里面链表的顺序。在并发的时候原来的顺序被另外一个线程a颠倒了，而被挂起线程b恢复后拿扩容前的节点和顺序继续完成第一次循环后，又遵循a线程扩容后的链表顺序重新排列链表中的顺序，最终形成了环。



## 10、ConcurrenHashMap 介绍？1.8 中为什么要用红黑树？

### ConcurrentHashMap

我们关注的操作有：get，put，remove 这3个操作。

对于哈希表，Java中采用链表的方式来解决hash冲突的。一个HashMap的数据结构看起来类似下图：





![img](https:////upload-images.jianshu.io/upload_images/4594052-57321ba2a9fa9e5a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/580)

[1](http://images.cnitblog.com/blog/587773/201409/062059382035540.jpg)



实现了同步的HashTable也是这样的结构，它的同步使用锁来保证的，并且所有同步操作使用的是同一个锁对象。这样若有n个线程同时在get时，这n个线程要串行的等待来获取锁。

ConcurrentHashMap中对这个数据结构，针对并发稍微做了一点调整。它把区间按照并发级别(concurrentLevel)，分成了若干个segment。默认情况下内部按并发级别为16来创建。对于每个segment的容量，默认情况也是16。当然并发级别(concurrentLevel)和每个段(segment)的初始容量都是可以通过构造函数设定的。

创建好默认的ConcurrentHashMap之后，它的结构大致如下图：





![img](https:////upload-images.jianshu.io/upload_images/4594052-3dca3ebf6c104347.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/580)

[1](http://images.cnitblog.com/blog/587773/201409/062059451104363.jpg)



看起来只是把以前HashTable的一个hash bucket创建了16份而已。有什么特别的吗？没啥特别的。

继续看每个segment是怎么定义的：

```
static final class Segment<K,V> extends ReentrantLock implements Serializable
```

Segment继承了ReentrantLock，表明每个segment都可以当做一个锁。（ReentrantLock前文已经提到，不了解的话就把当做synchronized的替代者吧）这样对每个segment中的数据需要同步操作的话都是使用每个segment容器对象自身的锁来实现。只有对全局需要改变时锁定的是所有的segment。

面的这种做法，就称之为**“分离锁（lock striping）”**。有必要对**“分拆锁”**和**“分离锁”**的概念描述一下：

分拆锁(lock spliting)就是若原先的程序中多处逻辑都采用同一个锁，但各个逻辑之间又相互独立，就可以拆(Spliting)为使用多个锁，每个锁守护不同的逻辑。
 分拆锁有时候可以被扩展，分成可大可小加锁块的集合，并且它们归属于相互独立的对象，这样的情况就是分离锁(lock striping)。（摘自《Java并发编程实践》）</pre>

看上去，单是这样就已经能大大提高多线程并发的性能了。还没完，继续看我们关注的get,put,remove这三个函数怎么保证数据同步的。

### 先看get方法

public V get(Object key) { int hash = hash(key); // throws NullPointerException if key null
 return segmentFor(hash).get(key, hash);
 }</pre>

它没有使用同步控制，交给segment去找，再看Segment中的get方法：

```
V get(Object key, int hash) { if (count != 0) { // read-volatile // ①
            HashEntry<K,V> e = getFirst(hash); while (e != null) { if (e.hash == hash && key.equals(e.key)) {
                    V v = e.value; if (v != null)  // ② 注意这里
                        return v; return readValueUnderLock(e); // recheck
 }
                e = e.next;
            }
        } return null;
}
```

它也没有使用锁来同步，只是判断获取的entry的value是否为null，为null时才使用加锁的方式再次去获取。

这个实现很微妙，没有锁同步的话，靠什么保证同步呢？我们一步步分析。

第一步，先判断一下 count != 0；count变量表示segment中存在entry的个数。如果为0就不用找了。
 假设这个时候恰好另一个线程put或者remove了这个segment中的一个entry，会不会导致两个线程看到的count值不一致呢？
 看一下count变量的定义： `transient volatile int count;`
 它使用了volatile来修改。我们前文说过，Java5之后，JMM实现了对volatile的保证：对volatile域的写入操作happens-before于每一个后续对同一个域的读写操作。
 所以，每次判断count变量的时候，即使恰好其他线程改变了segment也会体现出来。

第二步，获取到要该key所在segment中的索引地址，如果该地址有相同的hash对象，顺着链表一直比较下去找到该entry。当找到entry的时候，先做了一次比较： `if(v != null)` 我们用红色注释的地方。
 这是为何呢？

考虑一下，如果这个时候，另一个线程恰好新增/删除了entry，或者改变了entry的value，会如何？

先看一下HashEntry类结构。

```
static final class HashEntry<K,V> { final K key; final int hash; volatile V value; final HashEntry<K,V> next;
    。。。
}
```

除了 value，其它成员都是final修饰的，也就是说value可以被改变，其它都不可以改变，包括指向下一个HashEntry的next也不能被改变。（那删除一个entry时怎么办？后续会讲到。）

**1) 在get代码的①和②之间，另一个线程新增了一个entry**
 如果另一个线程新增的这个entry又恰好是我们要get的，这事儿就比较微妙了。

下图大致描述了put 一个新的entry的过程。





![img](https:////upload-images.jianshu.io/upload_images/4594052-08753882c2f93c24.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/580)

[1](http://images.cnitblog.com/blog/587773/201409/062059532193187.jpg)



因为每个HashEntry中的next也是final的，没法对链表最后一个元素增加一个后续entry所以新增一个entry的实现方式只能通过头结点来插入了。

newEntry对象是通过 `new HashEntry(K k , V v, HashEntry next)` 来创建的。如果另一个线程刚好new 这个对象时，当前线程来get它。因为没有同步，就可能会出现当前线程得到的newEntry对象是一个没有完全构造好的对象引用。

回想一下我们之前讨论的DCL的问题，这里也一样，没有锁同步的话，new 一个对象对于多线程看到这个对象的状态是没有保障的，这里同样有可能一个线程new这个对象的时候还没有执行完构造函数就被另一个线程得到这个对象引用。
 所以才需要判断一下：`if (v != null)` 如果确实是一个不完整的对象，则使用锁的方式再次get一次。

有没有可能会put进一个value为null的entry？ 不会的，已经做了检查，这种情况会抛出异常，所以 ②处的判断完全是出于对多线程下访问一个new出来的对象的状态检测。

**2) 在get代码的①和②之间，另一个线程修改了一个entry的value**

value是用volitale修饰的，可以保证读取时获取到的是修改后的值。

**3) 在get代码的①之后，另一个线程删除了一个entry**

假设我们的链表元素是：e1-> e2 -> e3 -> e4 我们要删除 e3这个entry，因为HashEntry中next的不可变，所以我们无法直接把e2的next指向e4，而是将要删除的节点之前的节点复制一份，形成新的链表。

它的实现大致如下图所示：





![img](https:////upload-images.jianshu.io/upload_images/4594052-0f8374a3f61df036.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/580)

[1](http://images.cnitblog.com/blog/587773/201409/062059580163855.jpg)



如果我们get的也恰巧是e3，可能我们顺着链表刚找到e1，这时另一个线程就执行了删除e3的操作，而我们线程还会继续沿着旧的链表找到e3返回。这里没有办法实时保证了。

我们第①处就判断了count变量，它保障了在 ①处能看到其他线程修改后的。①之后到②之间，如果再次发生了其他线程再删除了entry节点，就没法保证看到最新的了。

不过这也没什么关系，即使我们返回e3的时候，它被其他线程删除了，暴漏出去的e3也不会对我们新的链表造成影响。

这其实是一种乐观设计，设计者假设 ①之后到②之间 发生被其它线程增、删、改的操作可能性很小，所以不采用同步设计，而是采用了事后（其它线程这期间也来操作，并且可能发生非安全事件）弥补的方式。
 而因为其他线程的“改”和“删”对我们的数据都不会造成影响，所以只有对“新增”操作进行了安全检查，就是②处的非null检查，如果确认不安全事件发生，则采用加锁的方式再次get。

这样做减少了使用互斥锁对并发性能的影响。可能有人怀疑remove操作中复制链表的方式是否代价太大，这里我没有深入比较，不过既然Java5中这么实现，我想new一个对象的代价应该已经没有早期认为的那么严重。

我们基本分析完了get操作。对于put和remove操作，是使用锁同步来进行的，不过是用的ReentrantLock而不是synchronized，性能上要更高一些。它们的实现前文都已经提到过，就没什么可分析的了。

我们还需要知道一点，ConcurrentHashMap的迭代器不是Fast-Fail的方式，所以在迭代的过程中别其他线程添加/删除了元素，不会抛出异常，也不能体现出元素的改动。但也没有关系，因为每个entry的成员除了value都是final修饰的，暴漏出去也不会对其他元素造成影响。

### 加深

```
ConcurrentHashMap<String, Boolean> map = new ...;
Thread a = new Thread { void run() {
        map.put("first", true);
        map.put("second", true);
    }
};

Thread b = new Thread { void run() {
        map.clear();
    }
};

a.start();
b.start();
a.join();
b.join();
```

结果：

```
Map("first" -> true, "second" -> true)
Map("second" -> true)
Map()

ConcurrentHashMap<String, Boolean> map = new ...;
List<String> myKeys = new ...;

Thread a = new Thread { void run() {
        map.put("first", true); // more stuff
        map.remove("first");
        map.put("second", true);
    }
};

Thread b = new Thread { void run() {
        Set<String> keys = map.keySet(); for (String key : keys) {
            myKeys.add(key);
        }
    }
};

a.start();
b.start();
a.join();
b.join();</pre>
```

结果：

```
List()
List("first")
List("second")
List("first", "second")
```

解释：
 对于这两个现象的解释：`ConcurrentHashMap`中的`clear`方法：

```
public void clear() { for (int i = 0; i < segments.length; ++i)
        segments[i].clear();
}
```

如果线程b先执行了clear，清空了一部分segment的时候，线程a执行了put且正好把“first”放入了“清空过”的segment中，而把“second”放到了还没有清空过的segment中，就会出现上面的情况。

第二段代码，如果线程b执行了迭代遍历到first，而此时线程a还没有remove掉first，那么即使后续删除了first，迭代器里不会反应出来，也不抛出异常，这种迭代器被称为“弱一致性”(weakly consistent)迭代器。



### 为什么加入红黑树

在jdk1.8版本后，java对HashMap做了改进，在链表长度大于8的时候，将后面的数据存在红黑树中，以加快检索速度。

### 为什么链表长度大于8的时候才变为红黑树

因为红黑树需要进行左旋，右旋操作， 而单链表不需要，
以下都是单链表与红黑树结构对比。
如果元素小于8个，查询成本高，新增成本低
如果元素大于8个，查询成本低，新增成本高



## 11、AQS



AQS是AbstractQueuedSynchronizer的简称。AQS提供了一种实现阻塞锁和一系列依赖FIFO等待队列的同步器的框架，如下图所示。AQS为一系列同步器依赖于一个单独的原子变量（state）的同步器提供了一个非常有用的基础。子类们必须定义改变state变量的protected方法，这些方法定义了state是如何被获取或释放的。鉴于此，本类中的其他方法执行所有的排队和阻塞机制。子类也可以维护其他的state变量，但是为了保证同步，必须原子地操作这些变量。




![img](https:////upload-images.jianshu.io/upload_images/53727-ae36db58241c256b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/852)

AQS.png


    AbstractQueuedSynchronizer中对state的操作是原子的，且不能被继承。所有的同步机制的实现均依赖于对改变量的原子操作。为了实现不同的同步机制，我们需要创建一个非共有的（non-public internal）扩展了AQS类的内部辅助类来实现相应的同步逻辑。AbstractQueuedSynchronizer并不实现任何同步接口，它提供了一些可以被具体实现类直接调用的一些原子操作方法来重写相应的同步逻辑。AQS同时提供了互斥模式（exclusive）和共享模式（shared）两种不同的同步逻辑。一般情况下，子类只需要根据需求实现其中一种模式，当然也有同时实现两种模式的同步类，如。接下来将详细介绍AbstractQueuedSynchronizer的提供的一些具体实现方法。

#### state状态

  AbstractQueuedSynchronizer维护了一个volatile int类型的变量，用户表示当前同步状态。volatile虽然不能保证操作的原子性，但是保证了当前变量state的可见性。至于[volatile](https://www.jianshu.com/p/14fc9d34de33)的具体语义，可以参考相关文章。state的访问方式有三种:

> - getState()
> - setState()
> - compareAndSetState()

  这三种叫做均是原子操作，其中compareAndSetState的实现依赖于Unsafe的compareAndSwapInt()方法。代码实现如下：

```
    /**
     * The synchronization state.
     */
    private volatile int state;
  
    /**
     * Returns the current value of synchronization state.
     * This operation has memory semantics of a {@code volatile} read.
     * @return current state value
     */
    protected final int getState() {
        return state;
    }

    /**
     * Sets the value of synchronization state.
     * This operation has memory semantics of a {@code volatile} write.
     * @param newState the new state value
     */
    protected final void setState(int newState) {
        state = newState;
    }

    /**
     * Atomically sets synchronization state to the given updated
     * value if the current state value equals the expected value.
     * This operation has memory semantics of a {@code volatile} read
     * and write.
     *
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful. False return indicates that the actual
     *         value was not equal to the expected value.
     */
    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```

#### 自定义资源共享方式

  AQS定义两种资源共享方式：Exclusive（独占，只有一个线程能执行，如ReentrantLock）和Share（共享，多个线程可同时执行，如Semaphore/CountDownLatch）。
   不同的自定义同步器争用共享资源的方式也不同。自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了。自定义同步器实现时主要实现以下几种方法：

> - isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。
> - tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false。
> - tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败则返回false。
> - tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
> - tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

#### 源码实现

  接下来我们开始开始讲解AQS的源码实现。依照acquire-release、acquireShared-releaseShared的次序来。

#### 1. acquire(int)

    acquire是一种以独占方式获取资源，如果获取到资源，线程直接返回，否则进入等待队列，直到获取到资源为止，且整个过程忽略中断的影响。该方法是独占模式下线程获取共享资源的顶层入口。获取到资源后，线程就可以去执行其临界区代码了。下面是acquire()的源码：

```
/**
     * Acquires in exclusive mode, ignoring interrupts.  Implemented
     * by invoking at least once {@link #tryAcquire},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquire} until success.  This method can be used
     * to implement method {@link Lock#lock}.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquire} but is otherwise uninterpreted and
     *        can represent anything you like.
     */
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

  通过注释我们知道，acquire方法是一种互斥模式，且忽略中断。该方法至少执行一次`tryAcquire(int)`方法，如果tryAcquire(int)方法返回true，则acquire直接返回，否则当前线程需要进入队列进行排队。函数流程如下：

> 1. tryAcquire()尝试直接去获取资源，如果成功则直接返回；
> 2. addWaiter()将该线程加入等待队列的尾部，并标记为独占模式；
> 3. acquireQueued()使线程在等待队列中获取资源，一直获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
> 4. 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。

接下来一次介绍相关方法。

##### 1.1 tryAcquire(int)

   tryAcquire尝试以独占的方式获取资源，如果获取成功，则直接返回true，否则直接返回false。该方法可以用于实现Lock中的tryLock()方法。该方法的默认实现是抛出`UnsupportedOperationException`，具体实现由自定义的扩展了AQS的同步类来实现。AQS在这里只负责定义了一个公共的方法框架。这里之所以没有定义成abstract，是因为独占模式下只用实现tryAcquire-tryRelease，而共享模式下只用实现tryAcquireShared-tryReleaseShared。如果都定义成abstract，那么每个模式也要去实现另一模式下的接口。

```
/**
     * Attempts to acquire in exclusive mode. This method should query
     * if the state of the object permits it to be acquired in the
     * exclusive mode, and if so to acquire it.
     *
     * <p>This method is always invoked by the thread performing
     * acquire.  If this method reports failure, the acquire method
     * may queue the thread, if it is not already queued, until it is
     * signalled by a release from some other thread. This can be used
     * to implement method {@link Lock#tryLock()}.
     *
     * <p>The default
     * implementation throws {@link UnsupportedOperationException}.
     *
     * @param arg the acquire argument. This value is always the one
     *        passed to an acquire method, or is the value saved on entry
     *        to a condition wait.  The value is otherwise uninterpreted
     *        and can represent anything you like.
     * @return {@code true} if successful. Upon success, this object has
     *         been acquired.
     * @throws IllegalMonitorStateException if acquiring would place this
     *         synchronizer in an illegal state. This exception must be
     *         thrown in a consistent fashion for synchronization to work
     *         correctly.
     * @throws UnsupportedOperationException if exclusive mode is not supported
     */
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
```

##### 1.2 addWaiter(Node)

  该方法用于将当前线程根据不同的模式（`Node.EXCLUSIVE`互斥模式、`Node.SHARED`共享模式）加入到等待队列的队尾，并返回当前线程所在的结点。如果队列不为空，则以通过`compareAndSetTail`方法以CAS的方式将当前线程节点加入到等待队列的末尾。否则，通过enq(node)方法初始化一个等待队列，并返回当前节点。源码如下：

```
/**
     * Creates and enqueues node for current thread and given mode.
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
     * @return the new node
     */
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

##### 1.2.1 enq(node)

  `enq(node)`用于将当前节点插入等待队列，如果队列为空，则初始化当前队列。整个过程以CAS自旋的方式进行，直到成功加入队尾为止。源码如下：

```
/**
     * Inserts node into queue, initializing if necessary. See picture above.
     * @param node the node to insert
     * @return node's predecessor
     */
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

##### 1.3  acquireQueued(Node, int)

  `acquireQueued()`用于队列中的线程自旋地以独占且不可中断的方式获取同步状态（acquire），直到拿到锁之后再返回。该方法的实现分成两部分：如果当前节点已经成为头结点，尝试获取锁（tryAcquire）成功，然后返回；否则检查当前节点是否应该被park，然后将该线程park并且检查当前线程是否被可以被中断。

```
/**
     * Acquires in exclusive uninterruptible mode for thread already in
     * queue. Used by condition wait methods as well as acquire.
     *
     * @param node the node
     * @param arg the acquire argument
     * @return {@code true} if interrupted while waiting
     */
    final boolean acquireQueued(final Node node, int arg) {
        //标记是否成功拿到资源，默认false
        boolean failed = true;
        try {
            boolean interrupted = false;//标记等待过程中是否被中断过
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

##### 1.3.1  shouldParkAfterFailedAcquire(Node, Node)

  shouldParkAfterFailedAcquire方法通过对当前节点的前一个节点的状态进行判断，对当前节点做出不同的操作，至于每个Node的状态表示，可以参考接口文档。

```
/**
     * Checks and updates status for a node that failed to acquire.
     * Returns true if thread should block. This is the main signal
     * control in all acquire loops.  Requires that pred == node.prev.
     *
     * @param pred node's predecessor holding status
     * @param node the node
     * @return {@code true} if thread should block
     */
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

##### 1.3.2  parkAndCheckInterrupt()

  该方法让线程去休息，真正进入等待状态。park()会让当前线程进入waiting状态。在此状态下，有两种途径可以唤醒该线程：1）被unpark()；2）被interrupt()。需要注意的是，Thread.interrupted()会清除当前线程的中断标记位。

```
/**
     * Convenience method to park and then check if interrupted
     *
     * @return {@code true} if interrupted
     */
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

  我们再回到acquireQueued()，总结下该函数的具体流程：

> 1. 结点进入队尾后，检查状态，找到安全休息点；
> 2. 调用park()进入waiting状态，等待unpark()或interrupt()唤醒自己；
> 3. 被唤醒后，看自己是不是有资格能拿到号。如果拿到，head指向当前结点，并返回从入队到拿到号的整个过程中是否被中断过；如果没拿到，继续流程1。

最后，总结一下acquire()的流程：

> 1. 调用自定义同步器的tryAcquire()尝试直接去获取资源，如果成功则直接返回；
> 2. 没成功，则addWaiter()将该线程加入等待队列的尾部，并标记为独占模式；
> 3. acquireQueued()使线程在等待队列中休息，有机会时（轮到自己，会被unpark()）会去尝试获取资源。获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
> 4. 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。

#### 2. release(int)

  `release(int)`方法是独占模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果彻底释放了（即state=0）,它会唤醒等待队列里的其他线程来获取资源。这也正是unlock()的语义，当然不仅仅只限于unlock()。下面是release()的源码：

```
/**
     * Releases in exclusive mode.  Implemented by unblocking one or
     * more threads if {@link #tryRelease} returns true.
     * This method can be used to implement method {@link Lock#unlock}.
     *
     * @param arg the release argument.  This value is conveyed to
     *        {@link #tryRelease} but is otherwise uninterpreted and
     *        can represent anything you like.
     * @return the value returned from {@link #tryRelease}
     */
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

/**
     * Attempts to set the state to reflect a release in exclusive
     * mode.
     *
     * <p>This method is always invoked by the thread performing release.
     *
     * <p>The default implementation throws
     * {@link UnsupportedOperationException}.
     *
     * @param arg the release argument. This value is always the one
     *        passed to a release method, or the current state value upon
     *        entry to a condition wait.  The value is otherwise
     *        uninterpreted and can represent anything you like.
     * @return {@code true} if this object is now in a fully released
     *         state, so that any waiting threads may attempt to acquire;
     *         and {@code false} otherwise.
     * @throws IllegalMonitorStateException if releasing would place this
     *         synchronizer in an illegal state. This exception must be
     *         thrown in a consistent fashion for synchronization to work
     *         correctly.
     * @throws UnsupportedOperationException if exclusive mode is not supported
     */
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }

/**
     * Wakes up node's successor, if one exists.
     *
     * @param node the node
     */
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

  与acquire()方法中的tryAcquire()类似，tryRelease()方法也是需要独占模式的自定义同步器去实现的。正常来说，tryRelease()都会成功的，因为这是独占模式，该线程来释放资源，那么它肯定已经拿到独占资源了，直接减掉相应量的资源即可(state-=arg)，也不需要考虑线程安全的问题。但要注意它的返回值，上面已经提到了，release()是根据tryRelease()的返回值来判断该线程是否已经完成释放掉资源了！所以自义定同步器在实现时，如果已经彻底释放资源(state=0)，要返回true，否则返回false。
   `unparkSuccessor(Node)`方法用于唤醒等待队列中下一个线程。这里要注意的是，下一个线程并不一定是当前节点的next节点，而是下一个可以用来唤醒的线程，如果这个节点存在，调用`unpark()`方法唤醒。
   总之，release()是独占模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果彻底释放了（即state=0）,它会唤醒等待队列里的其他线程来获取资源。

#### 3. acquireShared(int)

  `acquireShared(int)`方法是共享模式下线程获取共享资源的顶层入口。它会获取指定量的资源，获取成功则直接返回，获取失败则进入等待队列，直到获取到资源为止，整个过程忽略中断。下面是acquireShared()的源码：

```
/**
     * Acquires in shared mode, ignoring interrupts.  Implemented by
     * first invoking at least once {@link #tryAcquireShared},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquireShared} until success.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquireShared} but is otherwise uninterpreted
     *        and can represent anything you like.
     */
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

##### 3.1 doAcquireShared(int)

  将当前线程加入等待队列尾部休息，直到其他线程释放资源唤醒自己，自己成功拿到相应量的资源后才返回。源码如下：

```
 /**
     * Acquires in shared uninterruptible mode.
     * @param arg the acquire argument
     */
    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

  跟独占模式比，还有一点需要注意的是，这里只有线程是head.next时（“老二”），才会去尝试获取资源，有剩余的话还会唤醒之后的队友。那么问题就来了，假如老大用完后释放了5个资源，而老二需要6个，老三需要1个，老四需要2个。老大先唤醒老二，老二一看资源不够，他是把资源让给老三呢，还是不让？答案是否定的！老二会继续park()等待其他线程释放资源，也更不会去唤醒老三和老四了。独占模式，同一时刻只有一个线程去执行，这样做未尝不可；但共享模式下，多个线程是可以同时执行的，现在因为老二的资源需求量大，而把后面量小的老三和老四也都卡住了。当然，这并不是问题，只是AQS保证严格按照入队顺序唤醒罢了（保证公平，但降低了并发）。实现如下：

```
/**
     * Sets head of queue, and checks if successor may be waiting
     * in shared mode, if so propagating if either propagate > 0 or
     * PROPAGATE status was set.
     *
     * @param node the node
     * @param propagate the return value from a tryAcquireShared
     */
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```

  此方法在setHead()的基础上多了一步，就是自己苏醒的同时，如果条件符合（比如还有剩余资源），还会去唤醒后继结点，毕竟是共享模式！至此，acquireShared()也要告一段落了。让我们再梳理一下它的流程：

> 1. tryAcquireShared()尝试获取资源，成功则直接返回；
> 2. 失败则通过doAcquireShared()进入等待队列park()，直到被unpark()/interrupt()并成功获取到资源才返回。整个等待过程也是忽略中断的。

#### 4. releaseShared(int)

  `releaseShared(int)`方法是共享模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果成功释放且允许唤醒等待线程，它会唤醒等待队列里的其他线程来获取资源。下面是releaseShared()的源码：

```
/**
     * Releases in shared mode.  Implemented by unblocking one or more
     * threads if {@link #tryReleaseShared} returns true.
     *
     * @param arg the release argument.  This value is conveyed to
     *        {@link #tryReleaseShared} but is otherwise uninterpreted
     *        and can represent anything you like.
     * @return the value returned from {@link #tryReleaseShared}
     */
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

  此方法的流程也比较简单，一句话：释放掉资源后，唤醒后继。跟独占模式下的release()相似，但有一点稍微需要注意：独占模式下的tryRelease()在完全释放掉资源（state=0）后，才会返回true去唤醒其他线程，这主要是基于独占下可重入的考量；而共享模式下的releaseShared()则没有这种要求，共享模式实质就是控制一定量的线程并发执行，那么拥有资源的线程在释放掉部分资源时就可以唤醒后继等待结点。

```
/**
     * Release action for shared mode -- signals successor and ensures
     * propagation. (Note: For exclusive mode, release just amounts
     * to calling unparkSuccessor of head if it needs signal.)
     */
    private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```



## 12、如何检测死锁？怎么预防死锁？



### 死锁是什么 

所谓死锁：是指两个或两个以上的进程在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程称为死锁进程。由于资源占用是互斥的，当某个进程提出申请资源后，使得有关进程在无外力协助下，永远分配不到必需的资源而无法继续运行，这就产生了一种特殊现象死锁。

### 死锁的必要条件 

虽然进程在运行过程中，可能发生死锁，但死锁的发生也必须具备一定的条件，死锁的发生必须具备以下四个必要条件。

**1）互斥条件：**指进程对所分配到的资源进行排它性使用，即在一段时间内某资源只由一个进程占用。如果此时还有其它进程请求资源，则请求者只能等待，直至占有资源的进程用毕释放。

**2）请求和保持条件：**指进程已经保持至少一个资源，但又提出了新的资源请求，而该资源已被其它进程占有，此时请求进程阻塞，但又对自己已获得的其它资源保持不放。

**3）不剥夺条件：**指进程已获得的资源，在未使用完之前，不能被剥夺，只能在使用完时由自己释放。

**4）环路等待条件：**指在发生死锁时，必然存在一个进程——资源的环形链，即进程集合{P0，P1，P2，···，Pn}中的P0正在等待一个P1占用的资源；P1正在等待P2占用的资源，……，Pn正在等待已被P0占用的资源。

在系统中已经出现死锁后，应该及时检测到死锁的发生，并采取适当的措施来解除死锁。目前处理死锁的方法可归结为以下四种：

### **预防死锁**

　　这是一种较简单和直观的事先预防的方法。方法是通过设置某些限制条件，去破坏产生死锁的四个必要条件中的一个或者几个，来预防发生死锁。预防死锁是一种较易实现的方法，已被广泛使用。但是由于所施加的限制条件往往太严格，可能会导致系统资源利用率和系统吞吐量降低。

###  **避免死锁**

　　该方法同样是属于事先预防的策略，但它并不须事先采取各种限制措施去破坏产生死锁的的四个必要条件，而是在资源的动态分配过程中，用某种方法去防止系统进入不安全状态，从而避免发生死锁。（安全状态、银行家算法）

### **检测死锁**

　　这种方法并不须事先采取任何限制性措施，也不必检查系统是否已经进入不安全区，此方法允许系统在运行过程中发生死锁。但可通过系统所设置的检测机构，及时地检测出死锁的发生，并精确地确定与死锁有关的进程和资源，然后采取适当措施，从系统中将已发生的死锁清除掉。（死锁定理化简资源分配图）

### **解除死锁**

　　这是与检测死锁相配套的一种措施。当检测到系统中已发生死锁时，须将进程从死锁状态中解脱出来。常用的实施方法是撤销或挂起一些进程，以便回收一些资源，再将这些资源分配给已处于阻塞状态的进程，使之转为就绪状态，以继续运行。死锁的检测和解除措施，有可能使系统获得较好的资源利用率和吞吐量，但在实现上难度也最大。（资源剥夺法，撤销进程法，进程回退法）

用两个线程请求被对方占用的资源，实现线程死锁

     /**
     * 用两个线程请求被对方占用的资源，实现线程死锁
     */
    public class DeadLockThread implements Runnable {
        private static final Object objectA = new Object();
        private static final Object objectB = new Object();
        private boolean flag;
     
        @Override
        public void run() {
            String threadName = Thread.currentThread().getName();
            System.out.println("当前线程 为：" 
            + threadName + "\tflag = " + flag);
            if (flag) {
                synchronized (objectA) {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(threadName 
                            + "已进入同步代码块objectA，准备进入objectB");
                    synchronized (objectB) {
                        System.out.println(threadName 
                                + "已经进入同步代码块objectB");
                    }
                }
     
            } else {
                synchronized (objectB) {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(threadName 
                            + "已进入同步代码块objectB，准备进入objectA");
                    synchronized (objectA) {
                        System.out.println(threadName 
                                + "已经进入同步代码块objectA");
                    }
                }
            }
        }
     
        public static void main(String[] args) {
            DeadLockThread deadlock1 = new DeadLockThread();
            DeadLockThread deadlock2 = new DeadLockThread();
            deadlock1.flag = true;
            deadlock2.flag = false;
            Thread thread1 = new Thread(deadlock1);
            Thread thread2 = new Thread(deadlock2);
            thread1.start();
            thread2.start();
     
        }
     
    }


## 13、Java 内存模型？

在Java中应为不同的目的可以将java划分为两种内存模型：gc内存模型。并发内存模型。

### gc内存模型

java与c++之间有一堵由内存动态分配与垃圾收集技术所围成的“高墙”。墙外面的人想进去，墙里面的人想出来。java在执行java程序的过程中会把它管理的内存划分若干个不同功能的数据管理区域。如图：

![img](https://images2015.cnblogs.com/blog/286989/201611/286989-20161124093427206-761806286.jpg)

![img](https://images2015.cnblogs.com/blog/286989/201701/286989-20170112150556791-818433561.png)

![img](https://images2015.cnblogs.com/blog/286989/201701/286989-20170112150607822-1924598543.jpg)

 

#### hotspot中的gc内存模型

整体上。分为三部分：栈，堆，程序计数器，他们每一部分有其各自的用途；虚拟机栈保存着每一条线程的执行程序调用堆栈；堆保存着类对象、数组的具体信息；程序计数器保存着每一条线程下一次执行指令位置。这三块区域中栈和程序计数器是线程私有的。也就是说每一个线程拥有其独立的栈和程序计数器。我们可以看看具体结构：

#### 虚拟机/本地方法栈

在栈中，会为每一个线程创建一个栈。线程越多，栈的内存使用越大。对于每一个线程栈。当一个方法在线程中执行的时候，会在线程栈中创建一个栈帧(stack   frame)，用于存放该方法的上下文(局部变量表、操作数栈、方法返回地址等等)。每一个方法从调用到执行完毕的过程，就是对应着一个栈帧入栈出栈的过程。

本地方法栈与虚拟机栈发挥的作用是类似的，他们之间的区别不过是虚拟机栈为虚拟机执行java(字节码)服务的，而本地方法栈是为虚拟机执行native方法服务的。

#### 方法区/堆

在hotspot的实现中，方法区就是在堆中称为永久代的堆区域。几乎所有的对象/数组的内存空间都在堆上(有少部分在栈上)。在gc管理中，将虚拟机堆分为永久代、老年代、新生代。通过名字我们可以知道一个对象新建一般在新生代。经过几轮的gc。还存活的对象会被移到老年代。永久代用来保存类信息、代码段等几乎不会变的数据。堆中的所有数据是线程共享的。

- 新生代：应为gc具体实现的优化的原因。hotspot又将新生代划分为一个eden区和两个survivor区。每一次新生代gc时候。只用到一个eden区，一个survivor区。新生代一般的gc策略为mark-copy。
- 老年代：当新生代中的对象经过若干轮gc后还存活/或survisor在gc内存不够的时候。会把当前对象移动到老年代。老年代一般gc策略为mark-compact。
- 永久代：永久代一般可以不参与gc。应为其中保存的是一些代码/常量数据/类信息。JDK 1.8 中已经不存在永久代。

JVM内存模型中分两大块，一块是 NEW Generation, 另一块是Old Generation. 在New  Generation中，有一个叫Eden的空间，主要是用来存放新生的对象，还有两个Survivor Spaces（from,to）,  它们用来存放每次垃圾回收后存活下来的对象。在Old Generation中，主要存放应用程序中生命周期长的内存对象，还有个Permanent  Generation，主要用来放JVM自己的反射对象，比如类对象和方法对象等。

#### 程序计数器

如同其名称一样。程序计数器用于记录某个线程下次执行指令位置。程序计数器也是线程私有的。

### 并发内存模型

java试图定义一个Java内存模型(Java memory model  jmm)来屏蔽掉各种硬件/操作系统的内存访问差异，以实现让java程序在各个平台下都能达到一致的内存访问效果。java内存模型主要目标是定义程序中各个变量的访问规则，即在虚拟机中将变量存储到内存和从内存中取出变量这样的底层细节。模型图如下：

![img](https://images2015.cnblogs.com/blog/286989/201611/286989-20161124093612487-2048767455.jpg)

#### java并发内存模型以及内存操作规则

java内存模型中规定了所有变量都存贮到主内存（如虚拟机物理内存中的一部分）中。每一个线程都有一个自己的工作内存(如cpu中的高速缓存)。线程中的工作内存保存了该线程使用到的变量的主内存的副本拷贝。线程对变量的所有操作（读取、赋值等）必须在该线程的工作内存中进行。不同线程之间无法直接访问对方工作内存中变量。线程间变量的值传递均需要通过主内存来完成。

关于主内存与工作内存之间的交互协议，即一个变量如何从主内存拷贝到工作内存。如何从工作内存同步到主内存中的实现细节。java内存模型定义了8种操作来完成。这8种操作每一种都是原子操作。8种操作如下：

- lock(锁定)：作用于主内存，它把一个变量标记为一条线程独占状态；
- unlock(解锁)：作用于主内存，它将一个处于锁定状态的变量释放出来，释放后的变量才能够被其他线程锁定；
- read(读取)：作用于主内存，它把变量值从主内存传送到线程的工作内存中，以便随后的load动作使用；
- load(载入)：作用于工作内存，它把read操作的值放入工作内存中的变量副本中；
- use(使用)：作用于工作内存，它把工作内存中的值传递给执行引擎，每当虚拟机遇到一个需要使用这个变量的指令时候，将会执行这个动作；
- assign(赋值)：作用于工作内存，它把从执行引擎获取的值赋值给工作内存中的变量，每当虚拟机遇到一个给变量赋值的指令时候，执行该操作；
- store(存储)：作用于工作内存，它把工作内存中的一个变量传送给主内存中，以备随后的write操作使用；
- write(写入)：作用于主内存，它把store传送值放到主内存中的变量中。

Java内存模型还规定了执行上述8种基本操作时必须满足如下规则:

- 不允许read和load、store和write操作之一单独出现，以上两个操作必须按顺序执行，但没有保证必须连续执行，也就是说，read与load之间、store与write之间是可插入其他指令的。
- 不允许一个线程丢弃它的最近的assign操作，即变量在工作内存中改变了之后必须把该变化同步回主内存。
- 不允许一个线程无原因地（没有发生过任何assign操作）把数据从线程的工作内存同步回主内存中。
- 一个新的变量只能从主内存中“诞生”，不允许在工作内存中直接使用一个未被初始化（load或assign）的变量，换句话说就是对一个变量实施use和store操作之前，必须先执行过了assign和load操作。
- 一个变量在同一个时刻只允许一条线程对其执行lock操作，但lock操作可以被同一个条线程重复执行多次，多次执行lock后，只有执行相同次数的unlock操作，变量才会被解锁。
- 如果对一个变量执行lock操作，将会清空工作内存中此变量的值，在执行引擎使用这个变量前，需要重新执行load或assign操作初始化变量的值。
- 如果一个变量实现没有被lock操作锁定，则不允许对它执行unlock操作，也不允许去unlock一个被其他线程锁定的变量。
- 对一个变量执行unlock操作之前，必须先把此变量同步回主内存（执行store和write操作）。

#### volatile型变量的特殊规则

关键字volatile可以说是Java虚拟机提供的最轻量级的同步机制，但是它并不容易完全被正确、完整的理解，以至于许多程序员都不习惯去使用它，遇到需要处理多线程的问题的时候一律使用synchronized来进行同步。了解volatile变量的语义对后面了解多线程操作的其他特性很有意义。Java内存模型对volatile专门定义了一些特殊的访问规则，当一个变量被定义成volatile之后，他将具备两种特性：

- 保证此变量对所有线程的可见性。第一保证此变量对所有线程的可见性，这里的“可见性”是指当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。而普通变量是做不到这点，普通变量的值在线程在线程间传递均需要通过住内存来完成，例如，线程A修改一个普通变量的值，然后向主内存进行会写，另外一个线程B在线程A回写完成了之后再从主内存进行读取操作，新变量值才会对线程B可见。另外，java里面的运算并非原子操作，会导致volatile变量的运算在并发下一样是不安全的。
- 禁止指令重排序优化。普通的变量仅仅会保证在该方法的执行过程中所有依赖赋值结果的地方都能获得正确的结果，而不能保证变量赋值操作的顺序与程序中的执行顺序一致，在单线程中，我们是无法感知这一点的。

由于volatile变量只能保证可见性，在不符合以下两条规则的运算场景中，我们仍然要通过加锁来保证原子性。

- 1.运算结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值。
- 2.变量不需要与其他的状态比阿尼浪共同参与不变约束。

### 原子性、可见性与有序性

Java内存模型是围绕着在并发过程中如何处理原子性、可见性和有序性这三个特征来建立的，我们逐个看下哪些操作实现了这三个特性。

- **原子性**（Atomicity）：由Java内存模型来直接保证的原子性变量包括read、load、assign、use、store和write，我们大致可以认为基本数据类型的访问读写是具备原子性的。如果应用场景需要一个更大方位的原子性保证，Java内存模型还提供了lock和unlock操作来满足这种需求，尽管虚拟机未把lock和unlock操作直接开放给用户使用，但是却提供了更高层次的字节码指令monitorenter和monitorexit来隐式的使用这两个操作，这两个字节码指令反应到Java代码中就是同步块--synchronized关键字，因此在synchronized块之间的操作也具备原子性。
- **可见性**（Visibility）：可见性是指当一个线程修改了共享变量的值，其他线程能够立即得知这个修改。上文在讲解volatile变量的时候我们已详细讨论过这一点。Java内存模型是通过在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值这种依赖主内存作为传递媒介的方式来实现可见性的，无论是普通变量还是volatile变量都是如此，普通变量与volatile变量的区别是，volatile的特殊规则保证了新值能立即同步到主内存，以及每次使用前立即从主内存刷新。因此，可以说volatile保证了多线程操作时变量的可见性，而普通变量则不能保证这一点。除了volatile之外，Java还有两个关键字能实现可见性，即synchronized和final.同步快的可见性是由“对一个变量执行unlock操作前，必须先把此变量同步回主内存”这条规则获得的，而final关键字的可见性是指：被final修饰的字段在构造器中一旦初始化完成，并且构造器没有把"this"的引用传递出去，那么在其他线程中就能看见final字段的值。
- **有序性**（Ordering）：Java内存模型的有序性在前面讲解volatile时也详细的讨论过了，Java程序中天然的有序性可以总结为一句话：如果在本线程内观察，所有的操作都是有序的：如果在一个线程中观察另外一个线程，所有的线程操作都是无序的。前半句是指“线程内表现为串行的语义”，后半句是指“指令重排序”现象和“工作内存与主内存同步延迟”现象。Java语言提供了volatile和synchronized两个关键字来保证线程之间操作的有序性，volatile关键字本身就包含了禁止指令重排序的语义，而synchronized则是由“一个变量在同一个时刻只允许一条线程对其进行lock操作”这条规则获得的，这条规则决定了持有同一个锁的两个同步块只能串行的进入。

## 14、如何保证多线程下 i++ 结果正确？



volatile解决的是多线程间共享变量的可见性问题，而保证不了多线程间共享变量原子性问题。
对于多线程的i++,++i,依然还是会存在多线程问题,volatile是无法解决的
比如：变量i=0,A线程更新i+1,B线程也更新i+1,经过2次自增操作之后，i可能不等于2，而是等于1；
原因是i++和++i并非原子操作,我们通过javaP查看字节码,会发现
    
    void f1() { i++; } 

的字节码‘如下：

    void f1();  
    Code:  
    0: aload_0  
    1: dup  
    2: getfield #2; //Field i:I  
    5: iconst_1  
    6: iadd  
    7: putfield #2; //Field i:I  
    10: return 
可见i++执行了多部操作,从变量i中读取读取i的值->值+1 ->将+1后的值写回i中,这样在多线程的时候执行情况就类似如下了

    Thread1 Thread2  
    r1 = i; r3 = i; //读取i值
    r2 = r1 + 1;r4 = r3 + 1;//i值加1
    i = r2; i = r4; //写回到i
这样会造成的问题就是 r1, r3读到的值都是 0,最后两个线程都将 1 写入 i, 最后 i等于 1,但是却进行了两次自增操作。
可知加了volatile和没加volatile都无法解决非原子操作的线程同步问题。

### **解决方法**

1、 使用循环CAS，实现i++原子操作
Java从JDK1.5开始提供了java.util.concurrent.atomic包来提供线程安全的原子操作类。这些原子操作类都是是用CAS来实现，i++的原子性操作。以AtomicInteger为例子，讲一下 public final int getAndIncrement(){} 方法的实现。

    public final int getAndIncrement() {
    for (;;) {
    int current = get();
    int next = current + 1;
    if (compareAndSet(current, next))
    return current;
    }
    }

2、使用锁机制，实现i++原子操作
    
    // 使用Lock实现，多线程的数据同步
    public static ReentrantLock lock = new ReentrantLock();
3、使用synchronized，实现i++原子操作
    
    for (int i = 0; i < times; i++) {
    // 进行自加的操作
    synchronized (SynchronizedTest.class) {
    count++;
    }
    }
## 15、线程池的种类，区别和使用场景？



### newCachedThreadPool：

- 底层：返回ThreadPoolExecutor实例，corePoolSize为0；maximumPoolSize为Integer.MAX_VALUE；keepAliveTime为60L；unit为TimeUnit.SECONDS；workQueue为SynchronousQueue(同步队列)
- 通俗：当有新任务到来，则插入到SynchronousQueue中，由于SynchronousQueue是同步队列，因此会在池中寻找可用线程来执行，若有可以线程则执行，若没有可用线程则创建一个线程来执行该任务；若池中线程空闲时间超过指定大小，则该线程会被销毁。
- 适用：执行很多短期异步的小程序或者负载较轻的服务器

### newFixedThreadPool：

- 底层：返回ThreadPoolExecutor实例，接收参数为所设定线程数量nThread，corePoolSize为nThread，maximumPoolSize为nThread；keepAliveTime为0L(不限时)；unit为：TimeUnit.MILLISECONDS；WorkQueue为：new LinkedBlockingQueue<Runnable>() 无解阻塞队列
- 通俗：创建可容纳固定数量线程的池子，每隔线程的存活时间是无限的，当池子满了就不在添加线程了；如果池中的所有线程均在繁忙状态，对于新任务会进入阻塞队列中(无界的阻塞队列)
- 适用：执行长期的任务，性能好很多

### newSingleThreadExecutor:

- 底层：FinalizableDelegatedExecutorService包装的ThreadPoolExecutor实例，corePoolSize为1；maximumPoolSize为1；keepAliveTime为0L；unit为：TimeUnit.MILLISECONDS；workQueue为：new LinkedBlockingQueue<Runnable>() 无解阻塞队列
- 通俗：创建只有一个线程的线程池，且线程的存活时间是无限的；当该线程正繁忙时，对于新任务会进入阻塞队列中(无界的阻塞队列)
- 适用：一个任务一个任务执行的场景

### NewScheduledThreadPool:

- 底层：创建ScheduledThreadPoolExecutor实例，corePoolSize为传递来的参数，maximumPoolSize为Integer.MAX_VALUE；keepAliveTime为0；unit为：TimeUnit.NANOSECONDS；workQueue为：new DelayedWorkQueue() 一个按超时时间升序排序的队列
- 通俗：创建一个固定大小的线程池，线程池内线程存活时间无限制，线程池可以支持定时及周期性任务执行，如果所有线程均处于繁忙状态，对于新任务会进入DelayedWorkQueue队列中，这是一种按照超时时间排序的队列结构
- 适用：周期性执行任务的场景

### 线程池任务执行流程：

1. 当线程池小于corePoolSize时，新提交任务将创建一个新线程执行任务，即使此时线程池中存在空闲线程。 

2. 当线程池达到corePoolSize时，新提交任务将被放入workQueue中，等待线程池中任务调度执行 

3. 当workQueue已满，且maximumPoolSize>corePoolSize时，新提交任务会创建新线程执行任务 

4. 当提交任务数超过maximumPoolSize时，新提交任务由RejectedExecutionHandler处理 

5. 当线程池中超过corePoolSize线程，空闲时间达到keepAliveTime时，关闭空闲线程 

6. 当设置allowCoreThreadTimeOut(true)时，线程池中corePoolSize线程空闲时间达到keepAliveTime也将关闭 

   

备注：

一般如果线程池任务队列采用LinkedBlockingQueue队列的话，那么不会拒绝任何任务（因为队列大小没有限制），这种情况下，ThreadPoolExecutor最多仅会按照最小线程数来创建线程，也就是说线程池大小被忽略了。

如果线程池任务队列采用ArrayBlockingQueue队列的话，那么ThreadPoolExecutor将会采取一个非常负责的算法，比如假定线程池的最小线程数为4，最大为8所用的ArrayBlockingQueue最大为10。随着任务到达并被放到队列中，线程池中最多运行4个线程（即最小线程数）。即使队列完全填满，也就是说有10个处于等待状态的任务，ThreadPoolExecutor也只会利用4个线程。如果队列已满，而又有新任务进来，此时才会启动一个新线程，这里不会因为队列已满而拒接该任务，相反会启动一个新线程。新线程会运行队列中的第一个任务，为新来的任务腾出空间。

这个算法背后的理念是：该池大部分时间仅使用核心线程（4个），即使有适量的任务在队列中等待运行。这时线程池就可以用作节流阀。如果挤压的请求变得非常多，这时该池就会尝试运行更多的线程来清理；这时第二个节流阀—最大线程数就起作用了。



## 16、分析线程池的实现原理和线程的调度过程？



### 1.关于线程池



#### 线程池的技术背景

在面向对象编程中，创建和销毁对象是很费时间的，因为创建一个对象要获取内存资源或者其它更多资源。在Java中更是如此，虚拟机将试图跟踪每一个对象，以便能够在对象销毁后进行垃圾回收。

所以提高服务程序效率的一个手段就是尽可能减少创建和销毁对象的次数，特别是一些很耗资源的对象创建和销毁。如何利用已有对象来服务就是一个需要解决的关键问题，其实这就是一些”池化资源”技术产生的原因。

例如Android中常见到的很多通用组件一般都离不开”池”的概念，如各种图片加载库，网络请求库，即使Android的消息传递机制中的Meaasge当使用Meaasge.obtain()就是使用的Meaasge池中的对象，因此这个概念很重要。本文将介绍的线程池技术同样符合这一思想。

线程池的优点:

- 重用线程池中的线程,减少因对象创建,销毁所带来的性能开销;
- 能有效的控制线程的最大并发数,提高系统资源利用率,同时避免过多的资源竞争,避免堵塞;
- 能够多线程进行简单的管理,使线程的使用简单、高效。

#### 线程池框架[Executor](http://www.codeceo.com/article/java-executor-learning.html)

java中的线程池是通过Executor框架实现的，Executor 框架包括类：Executor，Executors，ExecutorService，ThreadPoolExecutor ，[Callable和Future、FutureTask的使用](http://www.silencedut.com/2016/06/15/Callable和Future、FutureTask的使用/)等。

[![undefined](http://static.codeceo.com/images/2016/08/c6e8878b385f63044ccf27e10b577fbb.jpg)](http://static.codeceo.com/images/2016/08/c6e8878b385f63044ccf27e10b577fbb.jpg)

Executor: 所有线程池的接口,只有一个方法。

```java
public interface Executor {        
  void execute(Runnable command);      
}
```

ExecutorService: 增加Executor的行为，是Executor实现类的最直接接口。

Executors： 提供了一系列工厂方法用于创先线程池，返回的线程池都实现了ExecutorService 接口。

ThreadPoolExecutor：线程池的具体实现类,一般用的各种线程池都是基于这个类实现的。
构造方法如下:

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {

        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);

}

```

- corePoolSize：线程池的核心线程数,线程池中运行的线程数也永远不会超过 corePoolSize 个,默认情况下可以一直存活。可以通过设置allowCoreThreadTimeOut为True,此时 核心线程数就是0,此时keepAliveTime控制所有线程的超时时间。
- maximumPoolSize：线程池允许的最大线程数;
- keepAliveTime： 指的是空闲线程结束的超时时间;
- unit ：是一个枚举，表示 keepAliveTime 的单位;
- workQueue：表示存放任务的BlockingQueue<Runnable队列。
- BlockingQueue:阻塞队列（BlockingQueue）是java.util.concurrent下的主要用来控制线程同步的工具。如果BlockQueue是空的,从BlockingQueue取东西的操作将会被阻断进入等待状态,直到BlockingQueue进了东西才会被唤醒。同样,如果BlockingQueue是满的,任何试图往里存东西的操作也会被阻断进入等待状态,直到BlockingQueue里有空间才会被唤醒继续操作。
  阻塞队列常用于生产者和消费者的场景，生产者是往队列里添加元素的线程，消费者是从队列里拿元素的线程。阻塞队列就是生产者存放元素的容器，而消费者也只从容器里拿元素。具体的实现类有LinkedBlockingQueue,ArrayBlockingQueued等。一般其内部的都是通过Lock和Condition([显示锁（Lock）及Condition的学习与使用](http://www.silencedut.com/2016/06/12/显示锁（Lock）及Condition的学习与使用/))来实现阻塞和唤醒。

线程池的工作过程如下：

1. 线程池刚创建时，里面没有一个线程。任务队列是作为参数传进来的。不过，就算队列里面有任务，线程池也不会马上执行它们。
2. 当调用 execute() 方法添加一个任务时，线程池会做如下判断：
   - 如果正在运行的线程数量小于 corePoolSize，那么马上创建线程运行这个任务；
   - 如果正在运行的线程数量大于或等于 corePoolSize，那么将这个任务放入队列；
   - 如果这时候队列满了，而且正在运行的线程数量小于 maximumPoolSize，那么还是要创建非核心线程立刻运行这个任务；
   - 如果队列满了，而且正在运行的线程数量大于或等于 maximumPoolSize，那么线程池会抛出异常RejectExecutionException。
3. 当一个线程完成任务时，它会从队列中取下一个任务来执行。
4. 当一个线程无事可做，超过一定的时间（keepAliveTime）时，线程池会判断，如果当前运行的线程数大于 corePoolSize，那么这个线程就被停掉。所以线程池的所有任务完成后，它最终会收缩到 corePoolSize 的大小。

#### 线程池的创建和使用

生成线程池采用了工具类Executors的静态方法，以下是几种常见的线程池。

SingleThreadExecutor：单个后台线程 (其缓冲队列是无界的)

```java
public static ExecutorService newSingleThreadExecutor() {        
    return new FinalizableDelegatedExecutorService (
        new ThreadPoolExecutor(1, 1,                                    
        0L, TimeUnit.MILLISECONDS,                                    
        new LinkedBlockingQueue<Runnable>()));   
}
```

创建一个单线程的线程池。这个线程池只有一个核心线程在工作，也就是相当于单线程串行执行所有任务。如果这个唯一的线程因为异常结束，那么会有一个新的线程来替代它。此线程池保证所有任务的执行顺序按照任务的提交顺序执行。

FixedThreadPool：只有核心线程的线程池,大小固定 (其缓冲队列是无界的) 。

```java
public static ExecutorService newFixedThreadPool(int nThreads) {         
        return new ThreadPoolExecutor(nThreads, nThreads,                                       
            0L, TimeUnit.MILLISECONDS,                                         
            new LinkedBlockingQueue<Runnable>());     
}
```

创建固定大小的线程池。每次提交一个任务就创建一个线程，直到线程达到线程池的最大大小。线程池的大小一旦达到最大值就会保持不变，如果某个线程因为执行异常而结束，那么线程池会补充一个新线程。

CachedThreadPool：无界线程池，可以进行自动线程回收。

```java
public static ExecutorService newCachedThreadPool() {         
    return new ThreadPoolExecutor(0,Integer.MAX_VALUE,                                           
           60L, TimeUnit.SECONDS,                                       
           new SynchronousQueue<Runnable>());     
}
```

如果线程池的大小超过了处理任务所需要的线程，那么就会回收部分空闲（60秒不执行任务）的线程，当任务数增加时，此线程池又可以智能的添加新线程来处理任务。此线程池不会对线程池大小做限制，线程池大小完全依赖于操作系统（或者说JVM）能够创建的最大线程大小。SynchronousQueue是一个是缓冲区为1的阻塞队列。

ScheduledThreadPool：核心线程池固定，大小无限的线程池。此线程池支持定时以及周期性执行任务的需求。

```java
public static ExecutorService newScheduledThreadPool(int corePoolSize) {         
    return new ScheduledThreadPool(corePoolSize, 
              Integer.MAX_VALUE,                                                  
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,                                                    
              new DelayedWorkQueue());    
}
```

创建一个周期性执行任务的线程池。如果闲置,非核心线程池会在DEFAULT_KEEPALIVEMILLIS时间内回收。

线程池最常用的提交任务的方法有两种：

execute:

```java
ExecutorService.execute(Runnable runable)；
```

submit:

```java
FutureTask task = ExecutorService.submit(Runnable runnable); FutureTask<T> task = ExecutorService.submit(Runnable runnable,T Result); FutureTask<T> task = ExecutorService.submit(Callable<T> callable);
```

submit(Callable callable)的实现，submit(Runnable runnable)同理。

```java
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    FutureTask<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```

可以看出submit开启的是有返回结果的任务，会返回一个FutureTask对象，这样就能通过get()方法得到结果。submit最终调用的也是execute(Runnable   runable)，submit只是将Callable对象或Runnable封装成一个FutureTask对象，因为FutureTask是个Runnable，所以可以在execute中执行。关于Callable对象和Runnable怎么封装成FutureTask对象，见[Callable和Future、FutureTask的使用](http://www.silencedut.com/2016/06/15/Callable和Future、FutureTask的使用)。

#### 线程池实现的原理

如果只讲线程池的使用，那这篇博客没有什么大的价值，充其量也就是熟悉Executor相关API的过程。线程池的实现过程没有用到Synchronized关键字，用的都是[Volatile](http://www.codeceo.com/article/java-volatile-var.html),Lock和同步(阻塞)队列,Atomic相关类，FutureTask等等，因为后者的性能更优。理解的过程可以很好的学习源码中并发控制的思想。

在开篇提到过线程池的优点是可总结为以下三点：

1. 线程复用
2. 控制最大并发数
3. 管理线程

##### 1.线程复用过程

理解线程复用原理首先应了解线程生命周期。

[![undefined](http://static.codeceo.com/images/2016/08/395352415aa1dc9083d17390bd4e0817.jpg)](http://static.codeceo.com/images/2016/08/395352415aa1dc9083d17390bd4e0817.jpg)

在线程的生命周期中，它要经过新建(New)、就绪（Runnable）、运行（Running）、阻塞(Blocked)和死亡(Dead)5种状态。

Thread通过new来新建一个线程，这个过程是是初始化一些线程信息，如线程名，id,线程所属group等，可以认为只是个普通的对象。调用Thread的start()后Java虚拟机会为其创建方法调用栈和程序计数器，同时将hasBeenStarted为true,之后调用start方法就会有异常。

处于这个状态中的线程并没有开始运行，只是表示该线程可以运行了。至于该线程何时开始运行，取决于JVM里线程调度器的调度。当线程获取cpu后，run()方法会被调用。不要自己去调用Thread的run()方法。之后根据CPU的调度在就绪——运行——阻塞间切换，直到run()方法结束或其他方式停止线程，进入dead状态。

所以实现线程复用的原理应该就是要保持线程处于存活状态（就绪，运行或阻塞）。接下来来看下ThreadPoolExecutor是怎么实现线程复用的。

在ThreadPoolExecutor主要Worker类来控制线程的复用。看下Worker类简化后的代码，这样方便理解：

```java
private final class Worker implements Runnable {

    final Thread thread;

    Runnable firstTask;

    Worker(Runnable firstTask) {
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    public void run() {
        runWorker(this);
    }

    final void runWorker(Worker w) {
        Runnable task = w.firstTask;
        w.firstTask = null;
        while (task != null || (task = getTask()) != null){
        task.run();
    }
}
```

Worker是一个Runnable，同时拥有一个thread，这个thread就是要开启的线程，在新建Worker对象时同时新建一个Thread对象，同时将Worker自己作为参数传入TThread，这样当Thread的start()方法调用时，运行的实际上是Worker的run()方法，接着到runWorker()中,有个while循环，一直从getTask()里得到Runnable对象，顺序执行。getTask()又是怎么得到Runnable对象的呢？

依旧是简化后的代码：

```java
private Runnable getTask() {
    if(一些特殊情况) {
        return null;
    }

    Runnable r = workQueue.take();

    return r;
}
```

这个workQueue就是初始化ThreadPoolExecutor时存放任务的BlockingQueue队列，这个队列里的存放的都是将要执行的Runnable任务。因为BlockingQueue是个阻塞队列，BlockingQueue.take()得到如果是空，则进入等待状态直到BlockingQueue有新的对象被加入时唤醒阻塞的线程。所以一般情况Thread的run()方法就不会结束,而是不断执行从workQueue里的Runnable任务，这就达到了线程复用的原理了。

##### 2.控制最大并发数

那Runnable是什么时候放入workQueue？Worker又是什么时候创建，Worker里的Thread的又是什么时候调用start()开启新线程来执行Worker的run()方法的呢？有上面的分析看出Worker里的runWorker()执行任务时是一个接一个，串行进行的，那并发是怎么体现的呢？

很容易想到是在execute(Runnable runnable)时会做上面的一些任务。看下execute里是怎么做的。

execute:

简化后的代码

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();

     int c = ctl.get();
    // 当前线程数 < corePoolSize
    if (workerCountOf(c) < corePoolSize) {
        // 直接启动新的线程。
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }

    // 活动线程数 >= corePoolSize
    // runState为RUNNING && 队列未满
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 再次检验是否为RUNNING状态
        // 非RUNNING状态 则从workQueue中移除任务并拒绝
        if (!isRunning(recheck) && remove(command))
            reject(command);// 采用线程池指定的策略拒绝任务
        // 两种情况：
        // 1.非RUNNING状态拒绝新的任务
        // 2.队列满了启动新的线程失败（workCount > maximumPoolSize）
    } else if (!addWorker(command, false))
        reject(command);
}
```

addWorker:

简化后的代码

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();

     int c = ctl.get();
    // 当前线程数 < corePoolSize
    if (workerCountOf(c) < corePoolSize) {
        // 直接启动新的线程。
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }

    // 活动线程数 >= corePoolSize
    // runState为RUNNING && 队列未满
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 再次检验是否为RUNNING状态
        // 非RUNNING状态 则从workQueue中移除任务并拒绝
        if (!isRunning(recheck) && remove(command))
            reject(command);// 采用线程池指定的策略拒绝任务
        // 两种情况：
        // 1.非RUNNING状态拒绝新的任务
        // 2.队列满了启动新的线程失败（workCount > maximumPoolSize）
    } else if (!addWorker(command, false))
        reject(command);
}
```

根据代码再来看上面提到的线程池工作过程中的添加任务的情况：

```
* 如果正在运行的线程数量小于 corePoolSize，那么马上创建线程运行这个任务；   
* 如果正在运行的线程数量大于或等于 corePoolSize，那么将这个任务放入队列；
* 如果这时候队列满了，而且正在运行的线程数量小于 maximumPoolSize，那么还是要创建非核心线程立刻运行这个任务；
* 如果队列满了，而且正在运行的线程数量大于或等于 maximumPoolSize，那么线程池会抛出异常RejectExecutionException。
```

> 这就是Android的AsyncTask在并行执行是在超出最大任务数是抛出RejectExecutionException的原因所在，详见[基于最新版本的AsyncTask源码解读及AsyncTask的黑暗面](http://www.silencedut.com/2016/07/08/基于最新版本的AsyncTask源码解读及AsyncTask的黑暗面)

通过addWorker如果成功创建新的线程成功，则通过start()开启新线程，同时将firstTask作为这个Worker里的run()中执行的第一个任务。

虽然每个Worker的任务是串行处理，但如果创建了多个Worker，因为共用一个workQueue，所以就会并行处理了。

所以根据corePoolSize和maximumPoolSize来控制最大并发数。大致过程可用下图表示。

[![undefined](http://static.codeceo.com/images/2016/08/28aab795f4aad4b940b6d7f983b79493.jpg)](http://static.codeceo.com/images/2016/08/28aab795f4aad4b940b6d7f983b79493.jpg)

上面的讲解和图来可以很好的理解的这个过程。

如果是做Android开发的，并且对Handler原理比较熟悉，你可能会觉得这个图挺熟悉，其中的一些过程和Handler，Looper，Meaasge使用中，很相似。Handler.send(Message)相当于execute(Runnuble)，Looper中维护的Meaasge队列相当于BlockingQueue，只不过需要自己通过同步来维护这个队列，Looper中的loop()函数循环从Meaasge队列取Meaasge和Worker中的runWork()不断从BlockingQueue取Runnable是同样的道理。

##### 3.管理线程

通过线程池可以很好的管理线程的复用，控制并发数，以及销毁等过程,线程的复用和控制并发上面已经讲了，而线程的管理过程已经穿插在其中了，也很好理解。

在ThreadPoolExecutor有个ctl的AtomicInteger变量。通过这一个变量保存了两个内容：

- 所有线程的数量
- 每个线程所处的状态

其中低29位存线程数，高3位存runState，通过位运算来得到不同的值。

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

//得到线程的状态
private static int runStateOf(int c) {
    return c & ~CAPACITY;
}

//得到Worker的的数量
private static int workerCountOf(int c) {
    return c & CAPACITY;
}

// 判断线程是否在运行
private static boolean isRunning(int c) {
    return c < SHUTDOWN;
}
```

这里主要通过shutdown和shutdownNow()来分析线程池的关闭过程。首先线程池有五种状态来控制任务添加与执行。主要介绍以下三种：

- RUNNING状态：线程池正常运行，可以接受新的任务并处理队列中的任务；
- SHUTDOWN状态：不再接受新的任务，但是会执行队列中的任务；
- STOP状态：不再接受新任务，不处理队列中的任务

shutdown这个方法会将runState置为SHUTDOWN，会终止所有空闲的线程，而仍在工作的线程不受影响，所以队列中的任务人会被执行。shutdownNow方法将runState置为STOP。和shutdown方法的区别，这个方法会终止所有的线程，所以队列中的任务也不会被执行了。

### 总结

通过对ThreadPoolExecutor源码的分析，从总体上了解了线程池的创建，任务的添加，执行等过程，熟悉这些过程，使用线程池就会更轻松了。

而从中学到的一些对并发控制，以及生产者——消费者模型任务处理的使用，对以后理解或解决其他相关问题会有很大的帮助。比如Android中的Handler机制，而Looper中的Messager队列用一个BlookQueue来处理同样是可以的,这写就是读源码的收获吧。



### 深入理解线程池及其原理





我们使用线程的时候就去创建一个线程，这样实现起来非常简便，但是就会有一个问题：

如果并发的线程数量很多，并且每个线程都是执行一个时间很短的任务就结束了，这样频繁创建线程就会大大降低系统的效率，因为频繁创建线程和销毁线程需要时间。

那么有没有一种办法使得线程可以复用，就是执行完一个任务，并不被销毁，而是可以继续执行其他的任务？

在Java中可以通过线程池来达到这样的效果。今天我们就来详细讲解一下Java的线程池，首先我们从最核心的ThreadPool[Executor](http://www.codeceo.com/article/java-executor-learning.html)类中的方法讲起，然后再讲述它的实现原理，接着给出了它的使用示例，最后讨论了一下如何合理配置线程池的大小。

以下是本文的目录大纲：

- 一.Java中的ThreadPoolExecutor类
- 二.深入剖析线程池实现原理
- 三.使用示例
- 四.如何合理配置线程池的大小　

若有不正之处请多多谅解，并欢迎批评指正。

#### 一.Java中的ThreadPoolExecutor类

java.uitl.concurrent.ThreadPoolExecutor类是线程池中最核心的一个类，因此如果要透彻地了解Java中的线程池，必须先了解这个类。下面我们来看一下ThreadPoolExecutor类的具体实现源码。

在ThreadPoolExecutor类中提供了四个构造方法：

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    .....
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue);

    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory);

    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);

    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
        BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler);
    ...
}
```

从上面的代码可以得知，ThreadPoolExecutor继承了AbstractExecutorService类，并提供了四个构造器，事实上，通过观察每个构造器的源码具体实现，发现前面三个构造器都是调用的第四个构造器进行的初始化工作。

下面解释下一下构造器中各个参数的含义：

- corePoolSize：核心池的大小，这个参数跟后面讲述的线程池的实现原理有非常大的关系。在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，除非调用了prestartAllCoreThreads()或者prestartCoreThread()方法，从这2个方法的名字就可以看出，是预创建线程的意思，即在没有任务到来之前就创建corePoolSize个线程或者一个线程。默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中；
- maximumPoolSize：线程池最大线程数，这个参数也是一个非常重要的参数，它表示在线程池中最多能创建多少个线程；
- keepAliveTime：表示线程没有任务执行时最多保持多久时间会终止。默认情况下，只有当线程池中的线程数大于corePoolSize时，keepAliveTime才会起作用，直到线程池中的线程数不大于corePoolSize，即当线程池中的线程数大于corePoolSize时，如果一个线程空闲的时间达到keepAliveTime，则会终止，直到线程池中的线程数不超过corePoolSize。但是如果调用了allowCoreThreadTimeOut(boolean)方法，在线程池中的线程数不大于corePoolSize时，keepAliveTime参数也会起作用，直到线程池中的线程数为0；
- unit：参数keepAliveTime的时间单位，有7种取值，在TimeUnit类中有7种静态属性：

```
TimeUnit.DAYS;               //天
TimeUnit.HOURS;             //小时
TimeUnit.MINUTES;           //分钟
TimeUnit.SECONDS;           //秒
TimeUnit.MILLISECONDS;      //毫秒
TimeUnit.MICROSECONDS;      //微妙
TimeUnit.NANOSECONDS;       //纳秒
```

- workQueue：一个阻塞队列，用来存储等待执行的任务，这个参数的选择也很重要，会对线程池的运行过程产生重大影响，一般来说，这里的阻塞队列有以下几种选择：

```
ArrayBlockingQueue;
LinkedBlockingQueue;
SynchronousQueue;
```

ArrayBlockingQueue和PriorityBlockingQueue使用较少，一般使用LinkedBlockingQueue和Synchronous。线程池的排队策略与BlockingQueue有关。

- threadFactory：线程工厂，主要用来创建线程；
- handler：表示当拒绝处理任务时的策略，有以下四种取值：

```css
ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。 ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。 ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务
```

具体参数的配置与线程池的关系将在下一节讲述。

从上面给出的ThreadPoolExecutor类的代码可以知道，ThreadPoolExecutor继承了AbstractExecutorService，我们来看一下AbstractExecutorService的实现：

```java
public abstract class AbstractExecutorService implements ExecutorService {

    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) { };
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) { };
    public Future<?> submit(Runnable task) {};
    public <T> Future<T> submit(Runnable task, T result) { };
    public <T> Future<T> submit(Callable<T> task) { };
    private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,
                            boolean timed, long nanos)
        throws InterruptedException, ExecutionException, TimeoutException {
    };
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException {
    };
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                           long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
    };
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
    };
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                         long timeout, TimeUnit unit)
        throws InterruptedException {
    };
}
```

AbstractExecutorService是一个抽象类，它实现了ExecutorService接口。

我们接着看ExecutorService接口的实现：

```java
public interface ExecutorService extends Executor {

    void shutdown();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
    <T> Future<T> submit(Callable<T> task);
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

而ExecutorService又是继承了Executor接口，我们看一下Executor接口的实现：

```java
public interface Executor {
    void execute(Runnable command);
}
```

到这里，大家应该明白了ThreadPoolExecutor、AbstractExecutorService、ExecutorService和Executor几个之间的关系了。

Executor是一个顶层接口，在它里面只声明了一个方法execute(Runnable)，返回值为void，参数为Runnable类型，从字面意思可以理解，就是用来执行传进去的任务的；

然后ExecutorService接口继承了Executor接口，并声明了一些方法：submit、invokeAll、invokeAny以及shutDown等；

抽象类AbstractExecutorService实现了ExecutorService接口，基本实现了ExecutorService中声明的所有方法；

然后ThreadPoolExecutor继承了类AbstractExecutorService。

在ThreadPoolExecutor类中有几个非常重要的方法：

```sql
execute()
submit()
shutdown()
shutdownNow()
```

execute()方法实际上是Executor中声明的方法，在ThreadPoolExecutor进行了具体的实现，这个方法是ThreadPoolExecutor的核心方法，通过这个方法可以向线程池提交一个任务，交由线程池去执行。

submit()方法是在ExecutorService中声明的方法，在AbstractExecutorService就已经有了具体的实现，在ThreadPoolExecutor中并没有对其进行重写，这个方法也是用来向线程池提交任务的，但是它和execute()方法不同，它能够返回任务执行的结果，去看submit()方法的实现，会发现它实际上还是调用的execute()方法，只不过它利用了Future来获取任务执行结果（Future相关内容将在下一篇讲述）。

shutdown()和shutdownNow()是用来关闭线程池的。

还有很多其他的方法：

比如：getQueue() 、getPoolSize() 、getActiveCount()、getCompletedTaskCount()等获取与线程池相关属性的方法，有兴趣的朋友可以自行查阅API。

#### 二.深入剖析线程池实现原理

在上一节我们从宏观上介绍了ThreadPoolExecutor，下面我们来深入解析一下线程池的具体实现原理，将从下面几个方面讲解：

- 1.线程池状态
- 2.任务的执行
- 3.线程池中的线程初始化
- 4.任务缓存队列及排队策略
- 5.任务拒绝策略
- 6.线程池的关闭
- 7.线程池容量的动态调整

1.线程池状态

在ThreadPoolExecutor中定义了一个[Volatile](http://www.codeceo.com/article/java-volatile-var.html)变量，另外定义了几个static final变量表示线程池的各个状态：

```java
volatile int runState;
static final int RUNNING    = 0;
static final int SHUTDOWN   = 1;
static final int STOP       = 2;
static final int TERMINATED = 3;
```

runState表示当前线程池的状态，它是一个volatile变量用来保证线程之间的可见性；

下面的几个static final变量表示runState可能的几个取值。

当创建线程池后，初始时，线程池处于RUNNING状态；

如果调用了shutdown()方法，则线程池处于SHUTDOWN状态，此时线程池不能够接受新的任务，它会等待所有任务执行完毕；

如果调用了shutdownNow()方法，则线程池处于STOP状态，此时线程池不能接受新的任务，并且会去尝试终止正在执行的任务；

当线程池处于SHUTDOWN或STOP状态，并且所有工作线程已经销毁，任务缓存队列已经清空或执行结束后，线程池被设置为TERMINATED状态。

2.任务的执行

在了解将任务提交给线程池到任务执行完毕整个过程之前，我们先来看一下ThreadPoolExecutor类中其他的一些比较重要成员变量：

```java
private final BlockingQueue<Runnable> workQueue;              //任务缓存队列，用来存放等待执行的任务
private final ReentrantLock mainLock = new ReentrantLock();   //线程池的主要状态锁，对线程池状态（比如线程池大小
                                                              //、runState等）的改变都要使用这个锁
private final HashSet<Worker> workers = new HashSet<Worker>();  //用来存放工作集

private volatile long  keepAliveTime;    //线程存活时间   
private volatile boolean allowCoreThreadTimeOut;   //是否允许为核心线程设置存活时间
private volatile int   corePoolSize;     //核心池的大小（即线程池中的线程数目大于这个参数时，提交的任务会被放进任务缓存队列）
private volatile int   maximumPoolSize;   //线程池最大能容忍的线程数

private volatile int   poolSize;       //线程池中当前的线程数

private volatile RejectedExecutionHandler handler; //任务拒绝策略

private volatile ThreadFactory threadFactory;   //线程工厂，用来创建线程

private int largestPoolSize;   //用来记录线程池中曾经出现过的最大线程数

private long completedTaskCount;   //用来记录已经执行完毕的任务个数
```

每个变量的作用都已经标明出来了，这里要重点解释一下corePoolSize、maximumPoolSize、largestPoolSize三个变量。

corePoolSize在很多地方被翻译成核心池大小，其实我的理解这个就是线程池的大小。举个简单的例子：

假如有一个工厂，工厂里面有10个工人，每个工人同时只能做一件任务。

因此只要当10个工人中有工人是空闲的，来了任务就分配给空闲的工人做；

当10个工人都有任务在做时，如果还来了任务，就把任务进行排队等待；

如果说新任务数目增长的速度远远大于工人做任务的速度，那么此时工厂主管可能会想补救措施，比如重新招4个临时工人进来；

然后就将任务也分配给这4个临时工人做；

如果说着14个工人做任务的速度还是不够，此时工厂主管可能就要考虑不再接收新的任务或者抛弃前面的一些任务了。

当这14个工人当中有人空闲时，而新任务增长的速度又比较缓慢，工厂主管可能就考虑辞掉4个临时工了，只保持原来的10个工人，毕竟请额外的工人是要花钱的。

这个例子中的corePoolSize就是10，而maximumPoolSize就是14（10+4）。

也就是说corePoolSize就是线程池大小，maximumPoolSize在我看来是线程池的一种补救措施，即任务量突然过大时的一种补救措施。

不过为了方便理解，在本文后面还是将corePoolSize翻译成核心池大小。

largestPoolSize只是一个用来起记录作用的变量，用来记录线程池中曾经有过的最大线程数目，跟线程池的容量没有任何关系。

下面我们进入正题，看一下任务从提交到最终执行完毕经历了哪些过程。

在ThreadPoolExecutor类中，最核心的任务提交方法是execute()方法，虽然通过submit也可以提交任务，但是实际上submit方法里面最终调用的还是execute()方法，所以我们只需要研究execute()方法的实现原理即可：

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command)) {
        if (runState == RUNNING && workQueue.offer(command)) {
            if (runState != RUNNING || poolSize == 0)
                ensureQueuedTaskHandled(command);
        }
        else if (!addIfUnderMaximumPoolSize(command))
            reject(command); // is shutdown or saturated
    }
}
```

上面的代码可能看起来不是那么容易理解，下面我们一句一句解释：

首先，判断提交的任务command是否为null，若是null，则抛出空指针异常；

接着是这句，这句要好好理解一下：

```ruby
if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command))
```

由于是或条件运算符，所以先计算前半部分的值，如果线程池中当前线程数不小于核心池大小，那么就会直接进入下面的if语句块了。

如果线程池中当前线程数小于核心池大小，则接着执行后半部分，也就是执行：

```
addIfUnderCorePoolSize(command)
```

如果执行完addIfUnderCorePoolSize这个方法返回false，则继续执行下面的if语句块，否则整个方法就直接执行完毕了。

如果执行完addIfUnderCorePoolSize这个方法返回false，然后接着判断：

```
if (runState == RUNNING && workQueue.offer(command))
```

如果当前线程池处于RUNNING状态，则将任务放入任务缓存队列；如果当前线程池不处于RUNNING状态或者任务放入缓存队列失败，则执行：

```
addIfUnderMaximumPoolSize(command)
```

如果执行addIfUnderMaximumPoolSize方法失败，则执行reject()方法进行任务拒绝处理。

回到前面：

```
if (runState == RUNNING && workQueue.offer(command))
```

这句的执行，如果说当前线程池处于RUNNING状态且将任务放入任务缓存队列成功，则继续进行判断：

```ruby
if (runState != RUNNING || poolSize == 0)
```

这句判断是为了防止在将此任务添加进任务缓存队列的同时其他线程突然调用shutdown或者shutdownNow方法关闭了线程池的一种应急措施。如果是这样就执行：

```
ensureQueuedTaskHandled(command)
```

进行应急处理，从名字可以看出是保证 添加到任务缓存队列中的任务得到处理。

我们接着看2个关键方法的实现：addIfUnderCorePoolSize和addIfUnderMaximumPoolSize：

```java
private boolean addIfUnderCorePoolSize(Runnable firstTask) {
    Thread t = null;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (poolSize < corePoolSize && runState == RUNNING)
            t = addThread(firstTask);        //创建线程去执行firstTask任务   
        } finally {
        mainLock.unlock();
    }
    if (t == null)
        return false;
    t.start();
    return true;
}
```

这个是addIfUnderCorePoolSize方法的具体实现，从名字可以看出它的意图就是当低于核心吃大小时执行的方法。下面看其具体实现，首先获取到锁，因为这地方涉及到线程池状态的变化，先通过if语句判断当前线程池中的线程数目是否小于核心池大小，有朋友也许会有疑问：前面在execute()方法中不是已经判断过了吗，只有线程池当前线程数目小于核心池大小才会执行addIfUnderCorePoolSize方法的，为何这地方还要继续判断？原因很简单，前面的判断过程中并没有加锁，因此可能在execute方法判断的时候poolSize小于corePoolSize，而判断完之后，在其他线程中又向线程池提交了任务，就可能导致poolSize不小于corePoolSize了，所以需要在这个地方继续判断。然后接着判断线程池的状态是否为RUNNING，原因也很简单，因为有可能在其他线程中调用了shutdown或者shutdownNow方法。然后就是执行

```
t = addThread(firstTask);
```

这个方法也非常关键，传进去的参数为提交的任务，返回值为Thread类型。然后接着在下面判断t是否为空，为空则表明创建线程失败（即poolSize>=corePoolSize或者runState不等于RUNNING），否则调用t.start()方法启动线程。

我们来看一下addThread方法的实现：

```java
private Thread addThread(Runnable firstTask) {
    Worker w = new Worker(firstTask);
    Thread t = threadFactory.newThread(w);  //创建一个线程，执行任务   
    if (t != null) {
        w.thread = t;            //将创建的线程的引用赋值为w的成员变量       
        workers.add(w);
        int nt = ++poolSize;     //当前线程数加1       
        if (nt > largestPoolSize)
            largestPoolSize = nt;
    }
    return t;
}
```

在addThread方法中，首先用提交的任务创建了一个Worker对象，然后调用线程工厂threadFactory创建了一个新的线程t，然后将线程t的引用赋值给了Worker对象的成员变量thread，接着通过workers.add(w)将Worker对象添加到工作集当中。

下面我们看一下Worker类的实现：

```java
private final class Worker implements Runnable {
    private final ReentrantLock runLock = new ReentrantLock();
    private Runnable firstTask;
    volatile long completedTasks;
    Thread thread;
    Worker(Runnable firstTask) {
        this.firstTask = firstTask;
    }
    boolean isActive() {
        return runLock.isLocked();
    }
    void interruptIfIdle() {
        final ReentrantLock runLock = this.runLock;
        if (runLock.tryLock()) {
            try {
        if (thread != Thread.currentThread())
        thread.interrupt();
            } finally {
                runLock.unlock();
            }
        }
    }
    void interruptNow() {
        thread.interrupt();
    }

    private void runTask(Runnable task) {
        final ReentrantLock runLock = this.runLock;
        runLock.lock();
        try {
            if (runState < STOP &&
                Thread.interrupted() &&
                runState >= STOP)
            boolean ran = false;
            beforeExecute(thread, task);   //beforeExecute方法是ThreadPoolExecutor类的一个方法，没有具体实现，用户可以根据
            //自己需要重载这个方法和后面的afterExecute方法来进行一些统计信息，比如某个任务的执行时间等           
            try {
                task.run();
                ran = true;
                afterExecute(task, null);
                ++completedTasks;
            } catch (RuntimeException ex) {
                if (!ran)
                    afterExecute(task, ex);
                throw ex;
            }
        } finally {
            runLock.unlock();
        }
    }

    public void run() {
        try {
            Runnable task = firstTask;
            firstTask = null;
            while (task != null || (task = getTask()) != null) {
                runTask(task);
                task = null;
            }
        } finally {
            workerDone(this);   //当任务队列中没有任务时，进行清理工作       
        }
    }
}
```

它实际上实现了Runnable接口，因此上面的Thread t = threadFactory.newThread(w);效果跟下面这句的效果基本一样：

```java
Thread t = new Thread(w);
```

相当于传进去了一个Runnable任务，在线程t中执行这个Runnable。

既然Worker实现了Runnable接口，那么自然最核心的方法便是run()方法了：

```java
private final class Worker implements Runnable {
    private final ReentrantLock runLock = new ReentrantLock();
    private Runnable firstTask;
    volatile long completedTasks;
    Thread thread;
    Worker(Runnable firstTask) {
        this.firstTask = firstTask;
    }
    boolean isActive() {
        return runLock.isLocked();
    }
    void interruptIfIdle() {
        final ReentrantLock runLock = this.runLock;
        if (runLock.tryLock()) {
            try {
        if (thread != Thread.currentThread())
        thread.interrupt();
            } finally {
                runLock.unlock();
            }
        }
    }
    void interruptNow() {
        thread.interrupt();
    }

    private void runTask(Runnable task) {
        final ReentrantLock runLock = this.runLock;
        runLock.lock();
        try {
            if (runState < STOP &&
                Thread.interrupted() &&
                runState >= STOP)
            boolean ran = false;
            beforeExecute(thread, task);   //beforeExecute方法是ThreadPoolExecutor类的一个方法，没有具体实现，用户可以根据
            //自己需要重载这个方法和后面的afterExecute方法来进行一些统计信息，比如某个任务的执行时间等           
            try {
                task.run();
                ran = true;
                afterExecute(task, null);
                ++completedTasks;
            } catch (RuntimeException ex) {
                if (!ran)
                    afterExecute(task, ex);
                throw ex;
            }
        } finally {
            runLock.unlock();
        }
    }

    public void run() {
        try {
            Runnable task = firstTask;
            firstTask = null;
            while (task != null || (task = getTask()) != null) {
                runTask(task);
                task = null;
            }
        } finally {
            workerDone(this);   //当任务队列中没有任务时，进行清理工作       
        }
    }
}
```

从run方法的实现可以看出，它首先执行的是通过构造器传进来的任务firstTask，在调用runTask()执行完firstTask之后，在while循环里面不断通过getTask()去取新的任务来执行，那么去哪里取呢？自然是从任务缓存队列里面去取，getTask是ThreadPoolExecutor类中的方法，并不是Worker类中的方法，下面是getTask方法的实现：

```java
Runnable getTask() {
    for (;;) {
        try {
            int state = runState;
            if (state > SHUTDOWN)
                return null;
            Runnable r;
            if (state == SHUTDOWN)  // Help drain queue
                r = workQueue.poll();
            else if (poolSize > corePoolSize || allowCoreThreadTimeOut) //如果线程数大于核心池大小或者允许为核心池线程设置空闲时间，
                //则通过poll取任务，若等待一定的时间取不到任务，则返回null
                r = workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS);
            else
                r = workQueue.take();
            if (r != null)
                return r;
            if (workerCanExit()) {    //如果没取到任务，即r为null，则判断当前的worker是否可以退出
                if (runState >= SHUTDOWN) // Wake up others
                    interruptIdleWorkers();   //中断处于空闲状态的worker
                return null;
            }
            // Else retry
        } catch (InterruptedException ie) {
            // On interruption, re-check runState
        }
    }
}
```

在getTask中，先判断当前线程池状态，如果runState大于SHUTDOWN（即为STOP或者TERMINATED），则直接返回null。

如果runState为SHUTDOWN或者RUNNING，则从任务缓存队列取任务。

如果当前线程池的线程数大于核心池大小corePoolSize或者允许为核心池中的线程设置空闲存活时间，则调用poll(time,timeUnit)来取任务，这个方法会等待一定的时间，如果取不到任务就返回null。

然后判断取到的任务r是否为null，为null则通过调用workerCanExit()方法来判断当前worker是否可以退出，我们看一下workerCanExit()的实现：

```java
private boolean workerCanExit() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    boolean canExit;
    //如果runState大于等于STOP，或者任务缓存队列为空了
    //或者  允许为核心池线程设置空闲存活时间并且线程池中的线程数目大于1
    try {
        canExit = runState >= STOP ||
            workQueue.isEmpty() ||
            (allowCoreThreadTimeOut &&
             poolSize > Math.max(1, corePoolSize));
    } finally {
        mainLock.unlock();
    }
    return canExit;
}
```

也就是说如果线程池处于STOP状态、或者任务队列已为空或者允许为核心池线程设置空闲存活时间并且线程数大于1时，允许worker退出。如果允许worker退出，则调用interruptIdleWorkers()中断处于空闲状态的worker，我们看一下interruptIdleWorkers()的实现：

```java
void interruptIdleWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers)  //实际上调用的是worker的interruptIfIdle()方法
            w.interruptIfIdle();
    } finally {
        mainLock.unlock();
    }
}
```

从实现可以看出，它实际上调用的是worker的interruptIfIdle()方法，在worker的interruptIfIdle()方法中：

```java
void interruptIfIdle() {
    final ReentrantLock runLock = this.runLock;
    if (runLock.tryLock()) {    //注意这里，是调用tryLock()来获取锁的，因为如果当前worker正在执行任务，锁已经被获取了，是无法获取到锁的
                                //如果成功获取了锁，说明当前worker处于空闲状态
        try {
    if (thread != Thread.currentThread())  
    thread.interrupt();
        } finally {
            runLock.unlock();
        }
    }
}
```

这里有一个非常巧妙的设计方式，假如我们来设计线程池，可能会有一个任务分派线程，当发现有线程空闲时，就从任务缓存队列中取一个任务交给空闲线程执行。但是在这里，并没有采用这样的方式，因为这样会要额外地对任务分派线程进行管理，无形地会增加难度和复杂度，这里直接让执行完任务的线程去任务缓存队列里面取任务来执行。

我们再看addIfUnderMaximumPoolSize方法的实现，这个方法的实现思想和addIfUnderCorePoolSize方法的实现思想非常相似，唯一的区别在于addIfUnderMaximumPoolSize方法是在线程池中的线程数达到了核心池大小并且往任务队列中添加任务失败的情况下执行的：

```java
private boolean addIfUnderMaximumPoolSize(Runnable firstTask) {
    Thread t = null;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (poolSize < maximumPoolSize && runState == RUNNING)
            t = addThread(firstTask);
    } finally {
        mainLock.unlock();
    }
    if (t == null)
        return false;
    t.start();
    return true;
}
```

看到没有，其实它和addIfUnderCorePoolSize方法的实现基本一模一样，只是if语句判断条件中的poolSize < maximumPoolSize不同而已。

到这里，大部分朋友应该对任务提交给线程池之后到被执行的整个过程有了一个基本的了解，下面总结一下：

1）首先，要清楚corePoolSize和maximumPoolSize的含义；

2）其次，要知道Worker是用来起到什么作用的；

3）要知道任务提交给线程池之后的处理策略，这里总结一下主要有4点：

- 如果当前线程池中的线程数目小于corePoolSize，则每来一个任务，就会创建一个线程去执行这个任务；
- 如果当前线程池中的线程数目>=corePoolSize，则每来一个任务，会尝试将其添加到任务缓存队列当中，若添加成功，则该任务会等待空闲线程将其取出去执行；若添加失败（一般来说是任务缓存队列已满），则会尝试创建新的线程去执行这个任务；
- 如果当前线程池中的线程数目达到maximumPoolSize，则会采取任务拒绝策略进行处理；
- 如果线程池中的线程数量大于   corePoolSize时，如果某线程空闲时间超过keepAliveTime，线程将被终止，直至线程池中的线程数目不大于corePoolSize；如果允许为核心池中的线程设置存活时间，那么核心池中的线程空闲时间超过keepAliveTime，线程也会被终止。

3.线程池中的线程初始化

默认情况下，创建线程池之后，线程池中是没有线程的，需要提交任务之后才会创建线程。

在实际中如果需要线程池创建之后立即创建线程，可以通过以下两个方法办到：

- prestartCoreThread()：初始化一个核心线程；
- prestartAllCoreThreads()：初始化所有核心线程

下面是这2个方法的实现：

```java
public boolean prestartCoreThread() {
    return addIfUnderCorePoolSize(null); //注意传进去的参数是null
}

public int prestartAllCoreThreads() {
    int n = 0;
    while (addIfUnderCorePoolSize(null))//注意传进去的参数是null
        ++n;
    return n;
}
```

注意上面传进去的参数是null，根据第2小节的分析可知如果传进去的参数为null，则最后执行线程会阻塞在getTask方法中的

```
r = workQueue.take();
```

即等待任务队列中有任务。

4.任务缓存队列及排队策略

在前面我们多次提到了任务缓存队列，即workQueue，它用来存放等待执行的任务。

workQueue的类型为BlockingQueue<Runnable>，通常可以取下面三种类型：

1）ArrayBlockingQueue：基于数组的先进先出队列，此队列创建时必须指定大小；

2）LinkedBlockingQueue：基于链表的先进先出队列，如果创建时没有指定此队列大小，则默认为Integer.MAX_VALUE；

3）synchronousQueue：这个队列比较特殊，它不会保存提交的任务，而是将直接新建一个线程来执行新来的任务。

5.任务拒绝策略

当线程池的任务缓存队列已满并且线程池中的线程数目达到maximumPoolSize，如果还有任务到来就会采取任务拒绝策略，通常有以下四种策略：

```css
ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务
```

6.线程池的关闭

ThreadPoolExecutor提供了两个方法，用于线程池的关闭，分别是shutdown()和shutdownNow()，其中：

- shutdown()：不会立即终止线程池，而是要等所有任务缓存队列中的任务都执行完后才终止，但再也不会接受新的任务
- shutdownNow()：立即终止线程池，并尝试打断正在执行的任务，并且清空任务缓存队列，返回尚未执行的任务

7.线程池容量的动态调整

ThreadPoolExecutor提供了动态调整线程池容量大小的方法：setCorePoolSize()和setMaximumPoolSize()，

- setCorePoolSize：设置核心池大小
- setMaximumPoolSize：设置线程池最大能创建的线程数目大小

当上述参数从小变大时，ThreadPoolExecutor进行线程赋值，还可能立即创建新的线程来执行任务。

#### 三.使用示例

前面我们讨论了关于线程池的实现原理，这一节我们来看一下它的具体使用：

```java
public class Test {
     public static void main(String[] args) {   
         ThreadPoolExecutor executor = new ThreadPoolExecutor(5, 10, 200, TimeUnit.MILLISECONDS,
                 new ArrayBlockingQueue<Runnable>(5));

         for(int i=0;i<15;i++){
             MyTask myTask = new MyTask(i);
             executor.execute(myTask);
             System.out.println("线程池中线程数目："+executor.getPoolSize()+"，队列中等待执行的任务数目："+
             executor.getQueue().size()+"，已执行玩别的任务数目："+executor.getCompletedTaskCount());
         }
         executor.shutdown();
     }
}

class MyTask implements Runnable {
    private int taskNum;

    public MyTask(int num) {
        this.taskNum = num;
    }

    @Override
    public void run() {
        System.out.println("正在执行task "+taskNum);
        try {
            Thread.currentThread().sleep(4000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("task "+taskNum+"执行完毕");
    }
}
```

执行结果：

```
正在执行task 0
线程池中线程数目：1，队列中等待执行的任务数目：0，已执行玩别的任务数目：0
线程池中线程数目：2，队列中等待执行的任务数目：0，已执行玩别的任务数目：0
正在执行task 1
线程池中线程数目：3，队列中等待执行的任务数目：0，已执行玩别的任务数目：0
正在执行task 2
线程池中线程数目：4，队列中等待执行的任务数目：0，已执行玩别的任务数目：0
正在执行task 3
线程池中线程数目：5，队列中等待执行的任务数目：0，已执行玩别的任务数目：0
正在执行task 4
线程池中线程数目：5，队列中等待执行的任务数目：1，已执行玩别的任务数目：0
线程池中线程数目：5，队列中等待执行的任务数目：2，已执行玩别的任务数目：0
线程池中线程数目：5，队列中等待执行的任务数目：3，已执行玩别的任务数目：0
线程池中线程数目：5，队列中等待执行的任务数目：4，已执行玩别的任务数目：0
线程池中线程数目：5，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
线程池中线程数目：6，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
正在执行task 10
线程池中线程数目：7，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
正在执行task 11
线程池中线程数目：8，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
正在执行task 12
线程池中线程数目：9，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
正在执行task 13
线程池中线程数目：10，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
正在执行task 14
task 3执行完毕
task 0执行完毕
task 2执行完毕
task 1执行完毕
正在执行task 8
正在执行task 7
正在执行task 6
正在执行task 5
task 4执行完毕
task 10执行完毕
task 11执行完毕
task 13执行完毕
task 12执行完毕
正在执行task 9
task 14执行完毕
task 8执行完毕
task 5执行完毕
task 7执行完毕
task 6执行完毕
task 9执行完毕
```

从执行结果可以看出，当线程池中线程的数目大于5时，便将任务放入任务缓存队列里面，当任务缓存队列满了之后，便创建新的线程。如果上面程序中，将for循环中改成执行20个任务，就会抛出任务拒绝异常了。

不过在java doc中，并不提倡我们直接使用ThreadPoolExecutor，而是使用Executors类中提供的几个静态方法来创建线程池：

```java
Executors.newCachedThreadPool();        //创建一个缓冲池，缓冲池容量大小为
Integer.MAX_VALUEExecutors.newSingleThreadExecutor();   //创建容量为1的缓冲池
Executors.newFixedThreadPool(int);    //创建固定容量大小的缓冲池
```

下面是这三个静态方法的具体实现：

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

从它们的具体实现来看，它们实际上也是调用了ThreadPoolExecutor，只不过参数都已配置好了。

newFixedThreadPool创建的线程池corePoolSize和maximumPoolSize值是相等的，它使用的LinkedBlockingQueue；

newSingleThreadExecutor将corePoolSize和maximumPoolSize都设置为1，也使用的LinkedBlockingQueue；

newCachedThreadPool将corePoolSize设置为0，将maximumPoolSize设置为Integer.MAX_VALUE，使用的SynchronousQueue，也就是说来了任务就创建线程运行，当线程空闲超过60秒，就销毁线程。

实际中，如果Executors提供的三个静态方法能满足要求，就尽量使用它提供的三个方法，因为自己去手动配置ThreadPoolExecutor的参数有点麻烦，要根据实际任务的类型和数量来进行配置。

另外，如果ThreadPoolExecutor达不到要求，可以自己继承ThreadPoolExecutor类进行重写。

#### 四.如何合理配置线程池的大小

本节来讨论一个比较重要的话题：如何合理配置线程池大小，仅供参考。

一般需要根据任务的类型来配置线程池大小：

如果是CPU密集型任务，就需要尽量压榨CPU，参考值可以设为 NCPU+1

如果是IO密集型任务，参考值可以设置为2*NCPU

当然，这只是一个参考值，具体的设置还需要根据实际情况进行调整，比如可以先将线程池大小设置为参考值，再观察任务运行情况和系统负载、资源利用率来进行适当调整。



## 17、线程池如何调优，最大数目如何确认？

在java中，几乎所有需要异步或者并发执行任务的程序都可以使用线程池。在开发过程中，合理的使用线程池能够带来3个好处

    首先是降低资源消耗。通过重复利用已创建的线程降低创建线程和销毁线程所带来的开销。
    提高相应速度。当任务到达时，任务可以不需要等待线程创建就立即执行。
    提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅消耗系统资源，同时降低系统的稳定性，使用线程池可以进行统一分配、调优和监控。

如何合理的使用线程池，如何合理的给出线程池的大小，是非常重要的。
对于线程池的大小不能过大，也不能过小。过大会有大量的线程在相对较少的CPU和内存上竞争，过小又会导致空闲的处理器无法工作，浪费资源，降低吞吐率。

对于线程池大小的设定，我们需要考虑的问题有：

    CPU个数
    内存大小
    任务类型，是计算密集型（CPU密集型）还是I/O密集型
    是否需要一些稀缺资源，像数据库连接这种等等
    等等

有种简单的估算方式，设N为CPU个数

    对于CPU密集型的应用，线程池的大小设置为N+1
    对于I/O密集型的应用，线程池的大小设置为2N+1
    这种设置方式适合于一台机器上的应用的类型是单一的，并且只有一个线程池，实际情况还需要根据实际的应用进行验证。

在I/O优化中，以下的估算公式可能更合理
最佳线程数量 = （（线程等待时间+线程CPU时间）／ 线程CPU时间）* CPU个数

由公式可得，线程等待时间所占比例越高，需要越多的线程。

**线程CPU时间所占比例越高，所需的线程数越少。**





## 18、ThreadLocal原理，用的时候需要注意什么？



### **一.对ThreadLocal的理解**

​        ThreadLocal，很多地方叫做线程本地变量，也有些地方叫做线程本地存储，其实意思差不多。可能很多朋友都知道ThreadLocal为变量在每个线程中都创建了一个副本，那么每个线程可以访问自己内部的副本变量。

​        这句话从字面上看起来很容易理解，但是真正理解并不是那么容易。

​        我们还是先来看一个例子：

    class ConnectionManager {  
       
     private static Connection connect = null;  
       
     public static Connection openConnection() {  
     if(connect == null){  
     connect = DriverManager.getConnection();  
     }  
     return connect;  
     }  
       
     public static void closeConnection() {  
     if(connect!=null)  
     connect.close();  
     }  
    }  


​         假设有这样一个数据库链接管理类，这段代码在单线程中使用是没有任何问题的，但是如果在多线程中使用呢？很显然，在多线程中使用会存在线程安全问题：第一，这里面的2个方法都没有进行同步，很可能在openConnection方法中会多次创建connect；第二，由于connect是共享变量，那么必然在调用connect的地方需要使用到同步来保障线程安全，因为很可能一个线程在使用connect进行数据库操作，而另外一个线程调用closeConnection关闭链接。

​        所以出于线程安全的考虑，必须将这段代码的两个方法进行同步处理，并且在调用connect的地方需要进行同步处理。

​        这样将会大大影响程序执行效率，因为一个线程在使用connect进行数据库操作的时候，其他线程只有等待。

​         那么大家来仔细分析一下这个问题，这地方到底需不需要将connect变量进行共享？事实上，是不需要的。假如每个线程中都有一个connect变量，各个线程之间对connect变量的访问实际上是没有依赖关系的，即一个线程不需要关心其他线程是否对这个connect进行了修改的。

​        到这里，可能会有朋友想到，既然不需要在线程之间共享这个变量，可以直接这样处理，在每个需要使用数据库连接的方法中具体使用时才创建数据库链接，然后在方法调用完毕再释放这个连接。比如下面这样：



    class ConnectionManager {  
       
     private Connection connect = null;  
       
     public Connection openConnection() {  
     if(connect == null){  
     connect = DriverManager.getConnection();  
     }  
     return connect;  
     }  
       
     public void closeConnection() {  
     if(connect!=null)  
     connect.close();  
     }  
    }  
       
    class Dao{  
     public void insert() {  
     ConnectionManager connectionManager = new ConnectionManager();  
     Connection connection = connectionManager.openConnection();  
       
     //使用connection进行操作  
       
     connectionManager.closeConnection();  
     }  
    }  
​         这样处理确实也没有任何问题，由于每次都是在方法内部创建的连接，那么线程之间自然不存在线程安全问题。但是这样会有一个致命的影响：导致服务器压力非常大，并且严重影响程序执行性能。由于在方法中需要频繁地开启和关闭数据库连接，这样不仅严重影响程序执行效率，还可能导致服务器压力巨大。

​         那么这种情况下使用ThreadLocal是再适合不过的了，因为ThreadLocal在每个线程中对该变量会创建一个副本，即每个线程内部都会有一个该变量，且在线程内部任何地方都可以使用，线程之间互不影响，这样一来就不存在线程安全问题，也不会严重影响程序执行性能。

​        但是要注意，虽然ThreadLocal能够解决上面说的问题，但是由于在每个线程中都创建了副本，所以要考虑它对资源的消耗，比如内存的占用会比不使用ThreadLocal要大。

 

### **二.深入解析ThreadLocal类**

​        在上面谈到了对ThreadLocal的一些理解，那我们下面来看一下具体ThreadLocal是如何实现的。

​        先了解一下ThreadLocal类提供的几个方法：

    public T get() { }  
    public void set(T value) { }  
    public void remove() { }  
    protected T initialValue() { }  
​         get()方法是用来获取ThreadLocal在当前线程中保存的变量副本，set()用来设置当前线程中变量的副本，remove()用来移除当前线程中变量的副本，initialValue()是一个protected方法，一般是用来在使用时进行重写的，它是一个延迟加载方法，下面会详细说明。

​        首先我们来看一下ThreadLocal类是如何为每个线程创建一个变量的副本的。

​        先看下get方法的实现：

![img](http://dl2.iteye.com/upload/attachment/0115/6037/199dbecb-799a-3416-a128-d28f33e452ad.jpg)
         第一句是取得当前线程，然后通过getMap(t)方法获取到一个map，map的类型为ThreadLocalMap。然后接着下面获取到<key,value>键值对，注意这里获取键值对传进去的是 this，而不是当前线程t。

​        如果获取成功，则返回value值。

​        如果map为空，则调用setInitialValue方法返回value。

​        我们上面的每一句来仔细分析：

​        首先看一下getMap方法中做了什么：

![img](http://dl2.iteye.com/upload/attachment/0115/6039/698102e7-3050-3d59-abeb-88ce6fe1af09.jpg)
         可能大家没有想到的是，在getMap中，是调用当期线程t，返回当前线程t中的一个成员变量threadLocals。

​        那么我们继续取Thread类中取看一下成员变量threadLocals是什么：

![img](http://dl2.iteye.com/upload/attachment/0115/6041/874880da-b5b7-3aac-a306-cc51290ad965.jpg)
         实际上就是一个ThreadLocalMap，这个类型是ThreadLocal类的一个内部类，我们继续看ThreadLocalMap的实现：

![img](http://dl2.iteye.com/upload/attachment/0115/6043/11057335-c0ad-3560-b166-1d486e5009d2.jpg)
         可以看到ThreadLocalMap的Entry继承了WeakReference，并且使用ThreadLocal作为键值。

​        然后再继续看setInitialValue方法的具体实现：

![img](http://dl2.iteye.com/upload/attachment/0115/6045/ca2fe71c-b286-3f1f-8159-9d325f6ba932.jpg)
         很容易了解，就是如果map不为空，就设置键值对，为空，再创建Map，看一下createMap的实现：

![img](http://dl2.iteye.com/upload/attachment/0115/6047/3a4b4822-beae-3e19-a15b-480c272917d0.jpg)
         至此，可能大部分朋友已经明白了ThreadLocal是如何为每个线程创建变量的副本的：

​         首先，在每个线程Thread内部有一个ThreadLocal.ThreadLocalMap类型的成员变量threadLocals，这个threadLocals就是用来存储实际的变量副本的，键值为当前ThreadLocal变量，value为变量副本（即T类型的变量）。

​         初始时，在Thread里面，threadLocals为空，当通过ThreadLocal变量调用get()方法或者set()方法，就会对Thread类中的threadLocals进行初始化，并且以当前ThreadLocal变量为键值，以ThreadLocal要保存的副本变量为value，存到threadLocals。

​        然后在当前线程里面，如果要使用副本变量，就可以通过get方法在threadLocals里面查找。

​        下面通过一个例子来证明通过ThreadLocal能达到在每个线程中创建变量副本的效果：

    package com.bijian.study;  
      
    public class Test {  
          
        ThreadLocal<Long> longLocal = new ThreadLocal<Long>();  
        ThreadLocal<String> stringLocal = new ThreadLocal<String>();  
      
        public void set() {  
            longLocal.set(Thread.currentThread().getId());  
            stringLocal.set(Thread.currentThread().getName());  
        }  
      
        public long getLong() {  
            return longLocal.get();  
        }  
      
        public String getString() {  
            return stringLocal.get();  
        }  
      
        public static void main(String[] args) throws InterruptedException {  
            final Test test = new Test();  
      
            test.set();  
            System.out.println(test.getLong());  
            System.out.println(test.getString());  
      
            Thread thread1 = new Thread() {  
                public void run() {  
                    test.set();  
                    System.out.println(test.getLong());  
                    System.out.println(test.getString());  
                };  
            };  
            thread1.start();  
            thread1.join();  
      
            System.out.println(test.getLong());  
            System.out.println(test.getString());  
        }  
    }  
运行结果：

    1  
    main  
    11  
    Thread-0  
    1  
    main  
​         从这段代码的输出结果可以看出，在main线程中和thread1线程中，longLocal保存的副本值和stringLocal保存的副本值都不一样。最后一次在main线程再次打印副本值是为了证明在main线程中和thread1线程中的副本值确实是不同的。

​        总结一下：

​        1）实际的通过ThreadLocal创建的副本是存储在每个线程自己的threadLocals中的；

​        2）为何threadLocals的类型ThreadLocalMap的键值为ThreadLocal对象，因为每个线程中可有多个threadLocal变量，就像上面代码中的longLocal和stringLocal；

​        3）在进行get之前，必须先set，否则会报空指针异常；

​        如果想在get之前不需要调用set就能正常访问的话，必须重写initialValue()方法。

​         因为在上面的代码分析过程中，我们发现如果没有先set的话，即在map中查找不到对应的存储，则会通过调用setInitialValue方法返回i，而在setInitialValue方法中，有一个语句是T  value = initialValue()， 而默认情况下，initialValue方法返回的是null。

![img](http://dl2.iteye.com/upload/attachment/0115/6053/776ec899-ceb7-343d-942d-7b598bdd5771.jpg)
         看下面这个例子：

    package com.bijian.study;  
      
    public class Test02 {  
      
        ThreadLocal<Long> longLocal = new ThreadLocal<Long>();  
        ThreadLocal<String> stringLocal = new ThreadLocal<String>();  
      
        public void set() {  
            longLocal.set(Thread.currentThread().getId());  
            stringLocal.set(Thread.currentThread().getName());  
        }  
      
        public long getLong() {  
            return longLocal.get();  
        }  
      
        public String getString() {  
            return stringLocal.get();  
        }  
      
        public static void main(String[] args) throws InterruptedException {  
            final Test02 test = new Test02();  
      
            System.out.println(test.getLong());  
            System.out.println(test.getString());  
      
            Thread thread1 = new Thread() {  
                public void run() {  
                    test.set();  
                    System.out.println(test.getLong());  
                    System.out.println(test.getString());  
                };  
            };  
            thread1.start();  
            thread1.join();  
      
            System.out.println(test.getLong());  
            System.out.println(test.getString());  
        }  
    }  
运行结果：

    Exception in thread "main" java.lang.NullPointerException  
        at com.bijian.study.Test02.getLong(Test02.java:14)  
        at com.bijian.study.Test02.main(Test02.java:24)  
​        在main线程中，没有先set，直接get的话，运行时会报空指针异常。

​        但是如果改成下面这段代码，即重写了initialValue方法：

    package com.bijian.study;  
      
    public class Test03 {  
      
        ThreadLocal<Long> longLocal = new ThreadLocal<Long>() {  
            protected Long initialValue() {  
                return Thread.currentThread().getId();  
            };  
        };  
          
        ThreadLocal<String> stringLocal = new ThreadLocal<String>() {  
            protected String initialValue() {  
                return Thread.currentThread().getName();  
            };  
        };  
      
        public void set() {  
            longLocal.set(Thread.currentThread().getId());  
            stringLocal.set(Thread.currentThread().getName());  
        }  
      
        public long getLong() {  
            return longLocal.get();  
        }  
      
        public String getString() {  
            return stringLocal.get();  
        }  
      
        public static void main(String[] args) throws InterruptedException {  
            final Test03 test = new Test03();  
      
            //test.set();  
            System.out.println(test.getLong());  
            System.out.println(test.getString());  
      
            Thread thread1 = new Thread() {  
                public void run() {  
                    //test.set();  
                    System.out.println(test.getLong());  
                    System.out.println(test.getString());  
                };  
            };  
            thread1.start();  
            thread1.join();  
      
            System.out.println(test.getLong());  
            System.out.println(test.getString());  
        }  
    } 
运行结果：

    1  
    main  
    8  
    Thread-0  
    1  
    main  
​        就可以直接不用先set而直接调用get了。

 

### **三.ThreadLocal的应用场景**

​        最常见的ThreadLocal使用场景为 用来解决数据库连接、Session管理等。如：

​        数据库连接：

    private static ThreadLocal<Connection> connectionHolder = new ThreadLocal<Connection>() {  
        public Connection initialValue() {  
            return DriverManager.getConnection(DB_URL);  
        }  
    };  
      
    public static Connection getConnection() {  
        return connectionHolder.get();  
    }  
​        Session管理：

    private static final ThreadLocal threadSession = new ThreadLocal();  
      
    public static Session getSession() throws InfrastructureException {  
        Session s = (Session) threadSession.get();  
        try {  
            if (s == null) {  
                s = getSessionFactory().openSession();  
                threadSession.set(s);  
            }  
        } catch (HibernateException ex) {  
            throw new InfrastructureException(ex);  
        }  
        return s;  
    }  
## 19、CountDownLatch 和 CyclicBarrier 的用法，以及相互之间的差别?



摘要： jdk1.5之后，java的concurrent包提供了一些并发工具类，比如CountDownLatch和CyclicBarrier，Semaphore。这里简要的比较一下他们的共同之处与区别，同时介绍一下他们的使用场景。 CountDownLatch：一个线程A或是组线程A等待其它线程执行完毕后，一个线程A或是组线程A才继续执行。CyclicBarrier：一组线程使用await()指定barrier，所有线程都到达各自的barrier后，再同时执行各自barrier下面的代码。Semaphore：是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源...


CountDownLatch、CyclicBarrier和Semaphore区别与共同之处
区别

    CountDownLatch 使一个线程A或是组线程A等待其它线程执行完毕后，一个线程A或是组线程A才继续执行。CyclicBarrier：一组线程使用await()指定barrier，所有线程都到达各自的barrier后，再同时执行各自barrier下面的代码。Semaphore：是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源
    CountDownLatch是减计数方式，计数==0时释放所有等待的线程；CyclicBarrier是加计数方式，计数达到构造方法中参数指定的值时释放所有等待的线程。Semaphore，每次semaphore.acquire()，获取一个资源，每次semaphore.acquire(n)，获取n个资源，当达到semaphore 指定资源数量时就不能再访问线程处于阻塞，必须等其它线程释放资源，semaphore.relase()每次资源一个资源，semaphore.relase(n)每次资源n个资源。
    CountDownLatch当计数到0时，计数无法被重置；CyclicBarrier计数达到指定值时，计数置为0重新开始。
    CountDownLatch每次调用countDown()方法计数减一，调用await()方法只进行阻塞，对计数没任何影响；CyclicBarrier只有一个await()方法，调用await()方法计数加1，若加1后的值不等于构造方法的值，则线程阻塞。
    CountDownLatch、CyclikBarrier、Semaphore 都有一个int类型参数的构造方法。CountDownLatch、CyclikBarrier这个值作为计数用，达到该次数即释放等待的线程，而Semaphore 中所有acquire获取到的资源达到这个数，会使得其它线程阻塞。

共同

    CountDownLatch与CyclikBarrier两者的共同点是都具有await()方法，并且执行此方法会引起线程的阻塞，达到某种条件才能继续执行（这种条件也是两者的不同）。Semaphore，acquire方获取的资源达到最大数量时，线程再次acquire获取资源时，也会使线程处于阻塞状态。CountDownLatch与CyclikBarrier两者的共同点是都具有await()方法，并且执行此方法会引起线程的阻塞，达到某种条件才能继续执行（这种条件也是两者的不同）。Semaphore，acquire方获取的资源达到最大数量时，线程再次acquire获取资源时，也会使线程处于阻塞状态。CountDownLatch、CyclikBarrier、Semaphore 都有一个int类型参数的构造方法。
    CountDownLatch、CyclikBarrier、Semaphore 都有一个int类型参数的构造方法。


CountDownLatch、CyclicBarrier和Semaphore使用场景
CountDownLatch
由于CountDownLatch有个countDown()方法并且countDown()不会引起阻塞，所以CountDownLatch可以应用于主线程等待所有子线程结束后再继续执行的情况。具体使用方式为new一个构造参数为subThread数目的CountDownLatch，启动所有子线程后主线程await(),在每个子线程的最后执行countDown()，这样当所有子线程执行完后计数减为0，主线程释放等待继续执行。比如赛跑，每个运动员看做一个子线程，裁判就是主线程，裁判发令（设置一个值为1的计数器，发令之前所有子线程await等待命令，裁判员发令让计数置为0，所有子线程同时开跑）所有运动员开跑后，需要等待所有人跑完再统计成绩（设置一个值为运动员数目的计数器，所有运动员开跑后裁判await被阻塞，每个运动员跑完的时候countDown()一下，所有运动员跑完计数达到0，裁判释放阻塞开始计分）。

    public class CountDownLatchCase {
        
        public static void main(String args[]){
            //主线程为裁判，子线程为运动员
            final CountDownLatch operatorNum=new CountDownLatch(6);
            for(int i=0;i<6;i++){
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        Long s=System.currentTimeMillis();
                        Random random=new Random();
                        try {
                            Thread.sleep(random.nextInt(10000));
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        Long time=System.currentTimeMillis()-s;
                        System.out.println(String.format("运动员%s跑完,花了%s", Thread.currentThread().getName(),time));
                        //一个运动员跑完了
                        operatorNum.countDown();
                    }
                }).start();
            }
            try {
                operatorNum.await();
                System.out.println(String.format("所有运行员都跑完了，裁判%s可宣布结果啦！", Thread.currentThread().getName()));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            
        }
    }

CyclicBarrier
由于CyclicBarrier计数达到指定后会重新循环使用，所以CyclicBarrier可以用在所有子线程之间互相等待多次的情形。比如在某种需求中，比如一个大型的任务，常常需要分配好多子任务去执行，只有当所有子任务都执行完成时候，才能执行主任务，这时候，就可以选择CyclicBarrier了。 比如团队旅游，一个团队通常分为几组，每组人走的路线可能不同，但都需要到达某一地点等待团队其它成员到达后才能进行下一站。

    /**  
     * 各省数据独立，分库存偖。为了提高计算性能，统计时采用每个省开一个线程先计算单省结果，最后汇总。  
     *   
     * @author guangbo email:weigbo@163.com  
     *   
     */  
    public class Total {   
      
        // private ConcurrentHashMap result = new ConcurrentHashMap();   
      
        public static void main(String[] args) {   
            TotalService totalService = new TotalServiceImpl();   
            CyclicBarrier barrier = new CyclicBarrier(5,   
                    new TotalTask(totalService));   
      
            // 实际系统是查出所有省编码code的列表，然后循环，每个code生成一个线程。   
            new BillTask(new BillServiceImpl(), barrier, "北京").start();   
            new BillTask(new BillServiceImpl(), barrier, "上海").start();   
            new BillTask(new BillServiceImpl(), barrier, "广西").start();   
            new BillTask(new BillServiceImpl(), barrier, "四川").start();   
            new BillTask(new BillServiceImpl(), barrier, "黑龙江").start();   
      
        }   
    }   
      
    /**  
     * 主任务：汇总任务  
     */  
    class TotalTask implements Runnable {   
        private TotalService totalService;   
      
        TotalTask(TotalService totalService) {   
            this.totalService = totalService;   
        }   
      
        public void run() {   
            // 读取内存中各省的数据汇总，过程略。   
            totalService.count();   
            System.out.println("=======================================");   
            System.out.println("开始全国汇总");   
        }   
    }   
      
    /**  
     * 子任务：计费任务  
     */  
    class BillTask extends Thread {   
        // 计费服务   
        private BillService billService;   
        private CyclicBarrier barrier;   
        // 代码，按省代码分类，各省数据库独立。   
        private String code;   
      
        BillTask(BillService billService, CyclicBarrier barrier, String code) {   
            this.billService = billService;   
            this.barrier = barrier;   
            this.code = code;   
        }   
      
        public void run() {   
            System.out.println("开始计算--" + code + "省--数据！");   
            billService.bill(code);   
            // 把bill方法结果存入内存，如ConcurrentHashMap,vector等,代码略   
            System.out.println(code + "省已经计算完成,并通知汇总Service！");   
            try {   
                // 通知barrier已经完成   
                barrier.await();   
            } catch (InterruptedException e) {   
                e.printStackTrace();   
            } catch (BrokenBarrierException e) {   
                e.printStackTrace();   
            }   
        }   
      
    } 




Semaphore
Semaphore可以用于做流量控制，特别公用资源有限的应用场景，比如数据库连接。假如有一个需求，要读取几万个文件的数据，因为都是IO密集型任务，我们可以启动几十个线程并发的读取，但是如果读到内存后，还需要存储到数据库中，而数据库的连接数只有10个，这时我们必须控制只有十个线程同时获取数据库连接保存数据，否则会报错无法获取数据库连接。这个时候，我们就可以使用Semaphore来做流控，代码如下：

    public class SemaphoreCase {
     
        private static final int THREAD_COUNT = 30;
        private static ExecutorService threadPool = Executors.newFixedThreadPool(THREAD_COUNT);
        private static Semaphore s = new Semaphore(10);
        public static void main(String[] args) {
            for (int i = 0; i < THREAD_COUNT; i++) {
                threadPool.execute(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            s.acquire();
                            System.out.println("save data");
                            s.release();
                        } catch (InterruptedException e) {
                        }
                    }
                });
            }
            threadPool.shutdown();
        }
     
    }

注意：

    release函数和acquire并没有要求一定是同一个线程都调用，可以A线程申请资源，B线程释放资源；
    调用release函数之前并没有要求一定要先调用acquire函数。
## 20、LockSupport工具

### 1. LockSupport简介

在之前介绍[AQS的底层实现](https://www.jianshu.com/p/cc308d82cc71)，已经在介绍java中的Lock时，比如[ReentrantLock](https://www.jianshu.com/p/dc5602eafd51),[ReentReadWriteLocks](https://www.jianshu.com/p/4a624281235e),已经在介绍线程间等待/通知机制使用的[Condition](https://www.jianshu.com/p/28387056eeb4)时都会调用LockSupport.park()方法和LockSupport.unpark()方法。而这个在同步组件的实现中被频繁使用的LockSupport到底是何方神圣，现在就来看看。LockSupport位于java.util.concurrent.locks包下，有兴趣的可以直接去看源码，该类的方法并不是很多。LockSupprot是线程的阻塞原语，用来阻塞线程和唤醒线程。每个使用LockSupport的线程都会与一个许可关联，如果该许可可用，并且可在线程中使用，则调用park()将会立即返回，否则可能阻塞。如果许可尚不可用，则可以调用 unpark 使其可用。但是注意许可**不可重入**，也就是说只能调用一次park()方法，否则会一直阻塞。

### 2. LockSupport方法介绍

LockSupport中的方法不多，这里将这些方法做一个总结：

> **阻塞线程**

1. void park()：阻塞当前线程，如果调用unpark方法或者当前线程被中断，从能从park()方法中返回
2. void park(Object blocker)：功能同方法1，入参增加一个Object对象，用来记录导致线程阻塞的阻塞对象，方便进行问题排查；
3. void parkNanos(long nanos)：阻塞当前线程，最长不超过nanos纳秒，增加了超时返回的特性；
4. void parkNanos(Object blocker, long nanos)：功能同方法3，入参增加一个Object对象，用来记录导致线程阻塞的阻塞对象，方便进行问题排查；
5. void parkUntil(long deadline)：阻塞当前线程，知道deadline；
6. void parkUntil(Object blocker, long deadline)：功能同方法5，入参增加一个Object对象，用来记录导致线程阻塞的阻塞对象，方便进行问题排查；

> **唤醒线程**

void unpark(Thread thread):唤醒处于阻塞状态的指定线程

实际上LockSupport阻塞和唤醒线程的功能是依赖于sun.misc.Unsafe，这是一个很底层的类，有兴趣的可以去查阅资料，比如park()方法的功能实现则是靠unsafe.park()方法。另外在阻塞线程这一系列方法中还有一个很有意思的现象就是，每个方法都会新增一个带有Object的阻塞对象的重载方法。那么增加了一个Object对象的入参会有什么不同的地方了？示例代码很简单就不说了，直接看dump线程的信息。

**调用park()方法dump线程**：

```
"main" #1 prio=5 os_prio=0 tid=0x02cdcc00 nid=0x2b48 waiting on condition [0x00d6f000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:304)
        at learn.LockSupportDemo.main(LockSupportDemo.java:7)
```

**调用park(Object blocker)方法dump线程**

```
"main" #1 prio=5 os_prio=0 tid=0x0069cc00 nid=0x6c0 waiting on condition [0x00dcf000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x048c2d18> (a java.lang.String)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at learn.LockSupportDemo.main(LockSupportDemo.java:7)
```

通过分别调用这两个方法然后dump线程信息可以看出，带Object的park方法相较于无参的park方法会增加 parking to wait for  <0x048c2d18> (a java.lang.String）的信息，这种信息就类似于记录“案发现场”，有助于工程人员能够迅速发现问题解决问题。有个有意思的事情是，我们都知道如果使用synchronzed阻塞了线程dump线程时都会有阻塞对象的描述，在java 5推出LockSupport时遗漏了这一点，在java 6时进行了补充。还有一点需要需要的是：**synchronzed致使线程阻塞，线程会进入到BLOCKED状态，而调用LockSupprt方法阻塞线程会致使线程进入到WAITING状态。**

### 3. 一个例子

用一个很简单的例子说说这些方法怎么用。

```
public class LockSupportDemo {
    public static void main(String[] args) {
        Thread thread = new Thread(() -> {
            LockSupport.park();
            System.out.println(Thread.currentThread().getName() + "被唤醒");
        });
        thread.start();
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        LockSupport.unpark(thread);
    }
}
```

thread线程调用LockSupport.park()致使thread阻塞，当mian线程睡眠3秒结束后通过LockSupport.unpark(thread)方法唤醒thread线程,thread线程被唤醒执行后续操作。另外，还有一点值得关注的是，**LockSupport.unpark(thread)可以指定线程对象唤醒指定的线程**。

## 21、Condition接口及其实现原理

使用场景

为了更好的理解Lock和Condition的使用场景，下面我们先来实现这样一个功能：有多个生产者，多个消费者，一个产品容器，我们假设容器最多可以放3个产品，如果满了，生产者需要等待产品被消费，如果没有产品了，消费者需要等待。我们的目标是一共生产10个产品，最终消费10个产品，如何在多线程环境下完成这一挑战呢？下面是我简单实现的一个demo，仅供参考。

```
package com.lock.condition.test;

import java.util.LinkedList;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class LockConditionTest {
    // 生产 和 消费 的最大总数
    public static int totalCount = 10;
    // 已经生产的产品数
    public static volatile int hasProduceCount = 0;
    // 已经消费的产品数
    public static volatile int hasConsumeCount = 0;
    // 容器最大容量
    public static int containerSize = 3;
    // 使用公平策略的可重入锁，便于观察演示结果
    public static ReentrantLock lock = new ReentrantLock(true);
    public static Condition notEmpty = lock.newCondition();
    public static Condition notFull = lock.newCondition();
    // 容器
    public static LinkedList<Integer> container = new LinkedList<Integer>();
    // 用于标识产品
    public static AtomicInteger idGenerator = new AtomicInteger();



public static void main(String[] args) {
    Thread p1 = new Thread(new Producer(), "p-1");
    Thread p2 = new Thread(new Producer(), "p-2");
    Thread p3 = new Thread(new Producer(), "p-3");

​    Thread c1 = new Thread(new Consumer(), "c-1");
​    Thread c2 = new Thread(new Consumer(), "c-2");
​    Thread c3 = new Thread(new Consumer(), "c-3");

​    c1.start();
​    c2.start();
​    c3.start();
​    p1.start();
​    p2.start();
​    p3.start();
​    try{
​        c1.join();
​        c2.join();
​        c3.join();
​        p1.join();
​        p2.join();
​        p3.join();
​    }catch(Exception e){

​    }
​    System.out.println(" done. ");
}
static class Producer implements Runnable{
​    @Override
​    public void run() {
​        while(true){
​            lock.lock();
​            try{
​                // 容器满了，需要等待非满条件
​                while(container.size() >= containerSize){
​                    notFull.await();
​                }

​                // 到这里表明容器未满，但需要再次判断是否已经完成了任务
​                if(hasProduceCount >= totalCount){
​                    System.out.println(Thread.currentThread().getName()+" producer exit");
​                    return ;
​                }

​                int product = idGenerator.incrementAndGet();
​                // 把生产出来的产品放入容器
​                container.addLast(product);
​                System.out.println(Thread.currentThread().getName() + " product " + product);
​                hasProduceCount++;

​                // 通知消费线程可以去消费了
​                notEmpty.signal();
​            } catch (InterruptedException e) {
​            }finally{
​                lock.unlock();
​            }
​        }
​    }
}
static class Consumer implements Runnable{
​    @Override
​    public void run() {
​        while(true){
​            lock.lock();
​            try{
​                if(hasConsumeCount >= totalCount){
​                    System.out.println(Thread.currentThread().getName()+" consumer exit");
​                    return ;
​                }

​                // 一直等待有产品了，再继续往下消费
​                while(container.isEmpty()){
​                    notEmpty.await(2, TimeUnit.SECONDS);
​                    if(hasConsumeCount >= totalCount){
​                        System.out.println(Thread.currentThread().getName()+" consumer exit");
​                        return ;
​                    }
​                }

​                Integer product = container.removeFirst();
​                System.out.println(Thread.currentThread().getName() + " consume " + product);
​                hasConsumeCount++;

​                // 通知生产线程可以继续生产产品了
​                notFull.signal();
​            } catch (InterruptedException e) {
​            }finally{
​                lock.unlock();
​            }
​        }
​    }
}
}
```





一次执行的结果如下：


    `p-1 product 1`
    `p-3 product 2`
    `p-2 product 3`
    `c-3 consume 1`
    `c-2 consume 2`
    `c-1 consume 3`
    `p-1 product 4`
    `p-3 product 5`
    `p-2 product 6`
    `c-3 consume 4`
    `c-2 consume 5`
    `c-1 consume 6`
    `p-1 product 7`
    `p-3 product 8`
    `p-2 product 9`
    `c-3 consume 7`
    `c-2 consume 8`
    `c-1 consume 9`
    `p-1 product 10`
    `p-3 producer exit`
    `p-2 producer exit`
    `c-3 consume 10`
    `c-2 consumer exit`
    `c-1 consumer exit`
    `p-1 producer exit`
    `c-3 consumer exit`
     done. 
从结果可以发现已经达到我们的目的了。

### 深入理解Condition的实现原理

上面的示例只是为了展示 Lock结合Condition可以实现的一种经典场景，在有了感性的认识之后，我们将一步一步来观察Lock和Condition是如何协作完成这一任务的，这也是本篇的核心内容。

为了更好的理解和演示这一个过程，我们使用到的锁是使用公平策略模式的，我们会使用上面例子运作的流程。我们会使用到3个生产线程，3个消费线程，分别表示 p1、p2、p3和c1、c2、c3。

Condition的内部实现是使用节点链来实现的，每个条件实例对应一个节点链，我们有notEmpty 和 notFull 两个条件实例，所以会有两个等待节点链。

一切准备就绪 ，开始我们的探索之旅。

1、线程c3执行，然后发现没有产品可以消费，执行 notEmpty.await，进入等待队列中等候。

![](https://img-blog.csdn.net/20160911115601043)

2、线程c2和线程c1执行，然后发现没有产品可以消费，执行 notEmpty.await，进入等待队列中等候。

![这里写图片描述](https://img-blog.csdn.net/20160911115636605)

3、 线程 p1 启动，得到了锁，p1开始生产产品，这时候p3抢在p2之前，执行了lock操作，结果p2和p3都处于等待状态，入同步队列等待。

![这里写图片描述](https://img-blog.csdn.net/20160911115720121)

注意，本例中我们使用的是公平策略模式下的排它锁，由于p3抢先执行取锁操作，所以虽然p2和p3都被阻塞了，但是p3会优先被唤醒 。

4、这会，p1生产完毕，通知 not empty等待队列，可以唤醒一个等待线程节点了，然后释放了锁，释放锁会导致p3被唤醒，然后p1进入下一个循环，进入同步队列。

![这里写图片描述](https://img-blog.csdn.net/20160911110443597)

事情开始变得有趣了，p1执行一次生产后，执行了 notEmpty.signal，其效果就是把 not empty等待列表中的头节点，即c3节点移到同步等待列队中，重新参与抢占锁。

5、p3生产完了产品后，继续notEmpty.signal，同时释放锁，释放锁后会唤醒p2线程，然后p3在下一轮尝试获取锁的时候，再次入队。

![这里写图片描述](https://img-blog.csdn.net/20160911110722270)

6、接着，p2继续生产，生产后执行 notEmpty.signal，同时释放锁，释放锁后唤醒c3线程，然后p2在下一轮尝试取锁的时候，入列。

![这里写图片描述](https://img-blog.csdn.net/20160911110800897)

7、c3进行消费，你可以看到，现在 not empty等待列队中已经没有等待节点了，由于我们使用的是公平策略排它锁，这就会导致同步队列中的节点一个接着一个执行，而目前同步队列中的节点排列为一生产，一消费，这不难可以知道，接下来代码已经不会进入 wait条件了，所以一个一个轮流执行就是，比如c3，执行完了，继续notFull.signal(); 然后释放锁，入队，这里要明白，notFull.signal();这句代码其实没有作用了，因为 not full等待队列中没有任何等待线程节点。 c3执行后，状态如下图所示：

![这里写图片描述](https://img-blog.csdn.net/20160911110837989)


8、后面的事情我想大家都可以想得出来是怎样一步一步交替执行的了。

**总结**

本篇基于一个实例来演示结合Lock和Condition如何实现生产-消费模式，而且只讨论一种可能执行的流程，是想更简单的表述AQS底层是如何实现的。基于上面这个演示过程，针对其它的执行流程，其原来也是一样的。Condition内部使用一个节点链来保存所有 wait状态的线程，当对应条件被signal的时候，就会把等待节点转移到同步队列中，继续竞争锁。原理其实并不复杂，有兴趣的朋友可以翻阅源码。



## 22、Fork/Join框架的理解



### 前言

Java 1.7 引入了一种新的并发框架—— Fork/Join Framework。

本文的主要目的是介绍 ForkJoinPool 的适用场景，实现原理，以及示例代码。

*TLDR;* 如果觉得文章太长的话，以下就是**结论**：

- `ForkJoinPool` 不是为了替代 `ExecutorService`，而是它的补充，在某些应用场景下性能比 `ExecutorService` 更好。（见 *Java Tip: When to use ForkJoinPool vs ExecutorService* ）
- **ForkJoinPool 主要用于实现“分而治之”的算法，特别是分治之后递归调用的函数**，例如 quick sort 等。
- `ForkJoinPool` 最适合的是计算密集型的任务，如果存在 I/O，线程间同步，`sleep()` 等会造成线程长时间阻塞的情况时，最好配合使用 `ManagedBlocker`。

### 使用

首先介绍的是大家最关心的 Fork/Join Framework 的使用方法，如果对使用方法已经很熟悉的话，可以跳过这一节，直接阅读[原理](http://blog.dyngr.com/blog/2016/09/15/java-forkjoinpool-internals/#原理)。

用一个特别简单的求整数数组所有元素之和来作为我们现在需要解决的问题吧。

### 问题

> 计算1至1000的正整数之和。

### 解决方法

#### For-loop

最简单的，显然是不使用任何并行编程的手段，只用最直白的 *for-loop* 来实现。下面就是具体的实现代码。

不过为了便于横向对比，也为了让代码更加 Java Style，首先我们先定义一个 interface。

```
public interface Calculator {
    long sumUp(long[] numbers);
}
```

这个 interface 非常简单，只有一个函数 `sumUp`，就是返回数组内所有元素的和。

再写一个 `main` 方法。

```
public class Main {
    public static void main(String[] args) {
        long[] numbers = LongStream.rangeClosed(1, 1000).toArray();
        Calculator calculator = new MyCalculator();
        System.out.println(calculator.sumUp(numbers)); // 打印结果500500
    }
}
```

接下来就是我们的 Plain Old For-loop Calculator，简称 *POFLC* 的实现了。（这其实是个段子，和主题完全无关，感兴趣的请见文末的[彩蛋](http://blog.dyngr.com/blog/2016/09/15/java-forkjoinpool-internals/#彩蛋)）

```
public class ForLoopCalculator implements Calculator {
    public long sumUp(long[] numbers) {
        long total = 0;
        for (long i : numbers) {
            total += i;
        }
        return total;
    }
}
```

这段代码毫无出奇之处，也就不多解释了，直接跳入下一节——并行计算。

#### ExecutorService

在 Java 1.5 引入 `ExecutorService` 之后，基本上已经不推荐直接创建 `Thread` 对象，而是统一使用 `ExecutorService`。毕竟从接口的易用程度上来说 `ExecutorService` 就远胜于原始的 `Thread`，更不用提 `java.util.concurrent` 提供的数种线程池，Future 类，Lock 类等各种便利工具。

使用 `ExecutorService` 的实现

```
public class ExecutorServiceCalculator implements Calculator {
    private int parallism;
    private ExecutorService pool;

    public ExecutorServiceCalculator() {
        parallism = Runtime.getRuntime().availableProcessors(); // CPU的核心数
        pool = Executors.newFixedThreadPool(parallism);
    }

    private static class SumTask implements Callable<Long> {
        private long[] numbers;
        private int from;
        private int to;

        public SumTask(long[] numbers, int from, int to) {
            this.numbers = numbers;
            this.from = from;
            this.to = to;
        }

        @Override
        public Long call() throws Exception {
            long total = 0;
            for (int i = from; i <= to; i++) {
                total += numbers[i];
            }
            return total;
        }
    }

    @Override
    public long sumUp(long[] numbers) {
        List<Future<Long>> results = new ArrayList<>();

        // 把任务分解为 n 份，交给 n 个线程处理
        int part = numbers.length / parallism;
        for (int i = 0; i < parallism; i++) {
            int from = i * part;
            int to = (i == parallism - 1) ? numbers.length - 1 : (i + 1) * part - 1;
            results.add(pool.submit(new SumTask(numbers, from, to)));
        }

        // 把每个线程的结果相加，得到最终结果
        long total = 0L;
        for (Future<Long> f : results) {
            try {
                total += f.get();
            } catch (Exception ignore) {}
        }

        return total;
    }
}
```

如果对 `ExecutorService` 不太熟悉的话，推荐阅读[《七天七并发模型》](https://book.douban.com/subject/26337939/)的第二章，对 Java 的多线程编程基础讲解得比较清晰。当然著名的[《Java并发编程实战》](https://book.douban.com/subject/10484692/)也是不可多得的好书。

#### ForkJoinPool

前面花了点时间讲解了 `ForkJoinPool` 之前的实现方法，主要为了在代码的编写难度上进行一下对比。现在就列出本篇文章的重点——`ForkJoinPool` 的实现方法。

```
public class ForkJoinCalculator implements Calculator {
    private ForkJoinPool pool;

    private static class SumTask extends RecursiveTask<Long> {
        private long[] numbers;
        private int from;
        private int to;

        public SumTask(long[] numbers, int from, int to) {
            this.numbers = numbers;
            this.from = from;
            this.to = to;
        }

        @Override
        protected Long compute() {
            // 当需要计算的数字小于6时，直接计算结果
            if (to - from < 6) {
                long total = 0;
                for (int i = from; i <= to; i++) {
                    total += numbers[i];
                }
                return total;
            // 否则，把任务一分为二，递归计算
            } else {
                int middle = (from + to) / 2;
                SumTask taskLeft = new SumTask(numbers, from, middle);
                SumTask taskRight = new SumTask(numbers, middle+1, to);
                taskLeft.fork();
                taskRight.fork();
                return taskLeft.join() + taskRight.join();
            }
        }
    }

    public ForkJoinCalculator() {
        // 也可以使用公用的 ForkJoinPool：
        // pool = ForkJoinPool.commonPool()
        pool = new ForkJoinPool();
    }

    @Override
    public long sumUp(long[] numbers) {
        return pool.invoke(new SumTask(numbers, 0, numbers.length-1));
    }
}
```

可以看出，使用了 `ForkJoinPool` 的实现逻辑全部集中在了 `compute()` 这个函数里，仅用了14行就实现了完整的计算过程。特别是，在这段代码里没有显式地“把任务分配给线程”，只是分解了任务，而把具体的任务到线程的映射交给了 `ForkJoinPool` 来完成。

### 原理

如果你除了 `ForkJoinPool` 的用法以外，对 `ForkJoinPoll` 的原理也感兴趣的话，那么请接着阅读这一节。在这一节中，我会结合 `ForkJoinPool` 的作者 Doug Lea 的论文——[《A Java Fork/Join Framework》](http://gee.cs.oswego.edu/dl/papers/fj.pdf)，尽可能通俗地解释 Fork/Join Framework 的原理。

**我一直以为，要理解一样东西的原理，最好就是自己尝试着去实现一遍。**根据上面的示例代码，可以看出 `fork()` 和 `join()` 是 Fork/Join Framework “魔法”的关键。我们可以根据函数名假设一下 `fork()` 和 `join()` 的作用：

- `fork()`：开启一个新线程（或是重用线程池内的空闲线程），将任务交给该线程处理。
- `join()`：等待该任务的处理线程处理完毕，获得返回值。

以上模型似乎可以（？）解释 ForkJoinPool 能够多线程执行的事实，但有一个很明显的问题

> **当任务分解得越来越细时，所需要的线程数就会越来越多，而且大部分线程处于等待状态。**

但是如果我们在上面的示例代码加入以下代码

```
System.out.println(pool.getPoolSize());
```

这会显示当前线程池的大小，在我的机器上这个值是4，也就是说只有4个工作线程。甚至即使我们在初始化 pool 时指定所使用的线程数为1时，上述程序也没有任何问题——除了变成了一个串行程序以外。

```
public ForkJoinCalculator() {
    pool = new ForkJoinPool(1);
}
```

这个矛盾可以导出，**我们的假设是错误的，并不是每个 fork() 都会促成一个新线程被创建，而每个 join() 也不是一定会造成线程被阻塞。**Fork/Join Framework 的实现算法并不是那么“显然”，而是一个更加复杂的算法——这个算法的名字就叫做 *work stealing* 算法。

work stealing 算法在 Doung Lea 的[论文](http://gee.cs.oswego.edu/dl/papers/fj.pdf)中有详细的描述，以下是我在结合 Java 1.8 代码的阅读以后——现有代码的实现有一部分相比于论文中的描述发生了变化——得到的相对通俗的解释：

基本思想

![img](https://img-blog.csdn.net/20180808141149611?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyMzMxMDcz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

- `ForkJoinPool` 的每个工作线程都维护着一个**工作队列**（`WorkQueue`），这是一个双端队列（Deque），里面存放的对象是**任务**（`ForkJoinTask`）。
- 每个工作线程在运行中产生新的任务（通常是因为调用了 `fork()`）时，会放入工作队列的队尾，并且工作线程在处理自己的工作队列时，使用的是 *LIFO* 方式，也就是说每次从队尾取出任务来执行。
- 每个工作线程在处理自己的工作队列同时，会尝试**窃取**一个任务（或是来自于刚刚提交到 pool 的任务，或是来自于其他工作线程的工作队列），窃取的任务位于其他线程的工作队列的队首，也就是说工作线程在窃取其他工作线程的任务时，使用的是 *FIFO* 方式。
- 在遇到 `join()` 时，如果需要 join 的任务尚未完成，则会先处理其他任务，并等待其完成。
- 在既没有自己的任务，也没有可以窃取的任务时，进入休眠。

下面来介绍一下关键的两个函数：`fork()` 和 `join()` 的实现细节，相比来说 `fork()` 比 `join()` 简单很多，所以先来介绍 `fork()`。

fork

`fork()` 做的工作只有一件事，既是**把任务推入当前工作线程的工作队列里**。可以参看以下的源代码：

```
public final ForkJoinTask<V> fork() {
    Thread t;
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
        ((ForkJoinWorkerThread)t).workQueue.push(this);
    else
        ForkJoinPool.common.externalPush(this);
    return this;
}
```

join

`join()` 的工作则复杂得多，也是 `join()` 可以使得线程免于被阻塞的原因——不像同名的 `Thread.join()`。

1. 检查调用 `join()` 的线程是否是 ForkJoinThread 线程。如果不是（例如 main 线程），则阻塞当前线程，等待任务完成。如果是，则不阻塞。
2. 查看任务的完成状态，如果已经完成，直接返回结果。
3. 如果任务尚未完成，但处于自己的工作队列内，则完成它。
4. 如果任务已经被其他的工作线程偷走，则窃取这个小偷的工作队列内的任务（以 *FIFO* 方式），执行，以期帮助它早日完成欲 join 的任务。
5. 如果偷走任务的小偷也已经把自己的任务全部做完，正在等待需要 join 的任务时，则找到小偷的小偷，帮助它完成它的任务。
6. 递归地执行第5步。

将上述流程画成序列图的话就是这个样子：

![img](https://img-blog.csdn.net/20180808141229897?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyMzMxMDcz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

以上就是 `fork()` 和 `join()` 的原理，这可以解释 ForkJoinPool 在递归过程中的执行逻辑，但还有一个问题

> **最初的任务是 push 到哪个线程的工作队列里的？**

这就涉及到 `submit()` 函数的实现方法了

submit

其实除了前面介绍过的每个工作线程自己拥有的工作队列以外，`ForkJoinPool` 自身也拥有工作队列，这些工作队列的作用是用来接收由外部线程（非 `ForkJoinThread` 线程）提交过来的任务，而这些工作队列被称为 *submitting queue* 。

`submit()` 和 `fork()` 其实没有本质区别，只是提交对象变成了  submitting queue 而已（还有一些同步，初始化的操作）。submitting queue 和其他 work queue  一样，是工作线程”窃取“的对象，因此当其中的任务被一个工作线程成功窃取时，就意味着提交的任务真正开始进入执行阶段。

### 总结

在了解了 Fork/Join Framework 的工作原理之后，相信很多使用上的注意事项就可以从原理中找到原因。例如：**为什么在 ForkJoinTask 里最好不要存在 I/O 等会阻塞线程的行为？**，这个我姑且留作思考题吧 :)

还有一些延伸阅读的内容，在此仅提及一下：

1. `ForkJoinPool` 有一个 *Async Mode* ，效果是**工作线程在处理本地任务时也使用 FIFO 顺序**。这种模式下的 `ForkJoinPool` 更接近于是一个消息队列，而不是用来处理递归式的任务。
2. 在需要阻塞工作线程时，可以使用 `ManagedBlocker`。
3. Java 1.8 新增加的 `CompletableFuture` 类可以实现类似于 Javascript 的 promise-chain，内部就是使用 `ForkJoinPool` 来实现的。

## 23、分段锁的原理,锁力度减小的思考



前言：在分析ConcurrentHashMap的源码的时候，了解到这个并发容器类的加锁机制是基于粒度更小的分段锁，分段锁也是提升多并发程序性能的重要手段之一。

在并发程序中，串行操作是会降低可伸缩性，并且上下文切换也会减低性能。在锁上发生竞争时将通水导致这两种问题，使用独占锁时保护受限资源的时候，基本上是采用串行方式—-每次只能有一个线程能访问它。所以对于可伸缩性来说最大的威胁就是独占锁。

我们一般有三种方式降低锁的竞争程度： 
1、减少锁的持有时间 
2、降低锁的请求频率 
3、使用带有协调机制的独占锁，这些机制允许更高的并发性。

在某些情况下我们可以将锁分解技术进一步扩展为一组独立对象上的锁进行分解，这成为分段锁。其实说的简单一点就是：容器里有多把锁，每一把锁用于锁容器其中一部分数据，那么当多线程访问容器里不同数据段的数据时，线程间就不会存在锁竞争，从而可以有效的提高并发访问效率，这就是ConcurrentHashMap所使用的锁分段技术，首先将数据分成一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。

比如：在ConcurrentHashMap中使用了一个包含16个锁的数组，每个锁保护所有散列桶的1/16，其中第N个散列桶由第（N  mod  16）个锁来保护。假设使用合理的散列算法使关键字能够均匀的分部，那么这大约能使对锁的请求减少到越来的1/16。也正是这项技术使得ConcurrentHashMap支持多达16个并发的写入线程。

当然，任何技术必有其劣势，与独占锁相比，维护多个锁来实现独占访问将更加困难而且开销更加大。

下面给出一个基于散列的Map的实现，使用分段锁技术。

```
import java.util.Map;

/**
 * Created by louyuting on 17/1/10.
 */
public class StripedMap {
//同步策略: buckets[n]由 locks[n%N_LOCKS] 来保护

  private static final int N_LOCKS = 16;//分段锁的个数
  private final Node[] buckets;
  private final Object[] locks;

  /**
   * 结点
   * @param <K>
   * @param <V>
   */
  private static class Node<K,V> implements Map.Entry<K,V>{
final K key;//key
V value;//value
Node<K,V> next;//指向下一个结点的指针
int hash;//hash值

//构造器，传入Entry的四个属性
Node(int h, K k, V v, Node<K,V> n) {
  value = v;
  next = n;//该Entry的后继
  key = k;
  hash = h;
}

public final K getKey() {
return key;
}

public final V getValue() {
return value;
}

public final V setValue(V newValue) {
  V oldValue = value;
  value = newValue;
  return oldValue;
}

  }

/**
   * 构造器: 初始化散列桶和分段锁数组
   * @param numBuckets
   */
  public StripedMap(int numBuckets) {
buckets = new Node[numBuckets];
locks = new Object[N_LOCKS];

for(int i=0; i<N_LOCKS; i++){
  locks[i] = new Object();
}

  }

/**
   * 返回散列之后在散列桶之中的定位
   * @param key
   * @return
   */
  private final int hash(Object key){
return Math.abs(key.hashCode() % N_LOCKS);
  }


/**
   * 分段锁实现的get
   * @param key
   * @return
   */
  public Object get(Object key){
int hash = hash(key);//计算hash值

//获取分段锁中的某一把锁
synchronized (locks[hash% N_LOCKS]){
for(Node m=buckets[hash]; m!=null; m=m.next){
if(m.key.equals(key)){
return m.value;
}
  }
}

return null;
  }

/**
   * 清除整个map
   */
  public void clear() {
//分段获取散列桶中每个桶地锁，然后清除对应的桶的锁
for(int i=0; i<buckets.length; i++){
  synchronized (locks[i%N_LOCKS]){
buckets[i] = null;
  }
}
  }
}
```

上面的实现中：使用了N_LOCKS个锁对象数组，并且每个锁保护容器的一个子集，对于大多数的方法只需要回去key值的hash散列之后对应的数据区域的一把锁就行了。但是对于某些方法却要获得全部的锁，比如clear()方法，但是获得全部的锁不必是同时获得，可以使分段获得，具体的查看源码。

这就是分段锁的思想。

## 24、八种阻塞队列以及各个阻塞队列的特性



一. 前言

　　在新增的Concurrent包中，BlockingQueue很好的解决了多线程中，如何高效安全“传输”数据的问题。通过这些高效并且线程安全的队列类，为我们快速搭建高质量的多线程程序带来极大的便利。本文详细介绍了BlockingQueue家庭中的所有成员，包括他们各自的功能以及常见使用场景。

二. 认识BlockingQueue

　　阻塞队列，顾名思义，首先它是一个队列，而一个队列在数据结构中所起的作用大致如下图所示：
![img](https://pic002.cnblogs.com/images/2010/161940/2010112414472791.jpg)
　　从上图我们可以很清楚看到，通过一个共享的队列，可以使得数据由队列的一端输入，从另外一端输出；

　　常用的队列主要有以下两种：（当然通过不同的实现方式，还可以延伸出很多不同类型的队列，DelayQueue就是其中的一种）

　　　　先进先出（FIFO）：先插入的队列的元素也最先出队列，类似于排队的功能。从某种程度上来说这种队列也体现了一种公平性。

　　　　后进先出（LIFO）：后插入队列的元素最先出队列，这种队列优先处理最近发生的事件。　　

​      多线程环境中，通过队列可以很容易实现数据共享，比如经典的“生产者”和“消费者”模型中，通过队列可以很便利地实现两者之间的数据共享。假设我们有若干生产者线程，另外又有若干个消费者线程。如果生产者线程需要把准备好的数据共享给消费者线程，利用队列的方式来传递数据，就可以很方便地解决他们之间的数据共享问题。但如果生产者和消费者在某个时间段内，万一发生数据处理速度不匹配的情况呢？理想情况下，如果生产者产出数据的速度大于消费者消费的速度，并且当生产出来的数据累积到一定程度的时候，那么生产者必须暂停等待一下（阻塞生产者线程），以便等待消费者线程把累积的数据处理完毕，反之亦然。然而，在concurrent包发布以前，在多线程环境下，我们每个程序员都必须去自己控制这些细节，尤其还要兼顾效率和线程安全，而这会给我们的程序带来不小的复杂度。好在此时，强大的concurrent包横空出世了，而他也给我们带来了强大的BlockingQueue。（在多线程领域：所谓阻塞，在某些情况下会挂起线程（即阻塞），一旦条件满足，被挂起的线程又会自动被唤醒），下面两幅图演示了BlockingQueue的两个常见阻塞场景：*![img](https://pic002.cnblogs.com/images/2010/161940/2010112414442194.jpg)　　　　　　　**如上图所示：当队列中没有数据的情况下，消费者端的所有线程都会被自动阻塞（挂起），直到有数据放入队列。**![img](https://pic002.cnblogs.com/images/2010/161940/2010112414451925.jpg)　　　**如上图所示：当队列中填满数据的情况下，生产者端的所有线程都会被自动阻塞（挂起），直到队列中有空的位置，线程被自动唤醒。***

　　这也是我们在多线程环境下，为什么需要BlockingQueue的原因。作为BlockingQueue的使用者，我们再也不需要关心什么时候需要阻塞线程，什么时候需要唤醒线程，因为这一切BlockingQueue都给你一手包办了。既然BlockingQueue如此神通广大，让我们一起来见识下它的常用方法：

三. **BlockingQueue的核心方法**：

　　1.放入数据

　　　　（1）offer(anObject):表示如果可能的话,将anObject加到BlockingQueue里,即如果BlockingQueue可以容纳,则返回true,否则返回false.（本方法不阻塞当前执行方法

 的线程）；　　　　　　 
     　　（2）offer(E o, long timeout, TimeUnit unit)：可以设定等待的时间，如果在指定的时间内，还不能往队列中加入BlockingQueue，则返回失败。

　　　　（3）put(anObject):把anObject加到BlockingQueue里,如果BlockQueue没有空间,则调用此方法的线程被阻断直到BlockingQueue里面有空间再继续.

　　2. 获取数据

　　　　（1）poll(time):取走BlockingQueue里排在首位的对象,若不能立即取出,则可以等time参数规定的时间,取不到时返回null;

　　　　（2）poll(long timeout, TimeUnit unit)：从BlockingQueue取出一个队首的对象，如果在指定时间内，队列一旦有数据可取，则立即返回队列中的数据。否则知道时间

超时还没有数据可取，返回失败。

　　　　（3）take():取走BlockingQueue里排在首位的对象,若BlockingQueue为空,阻断进入等待状态直到BlockingQueue有新的数据被加入; 

　　　　（4）drainTo():一次性从BlockingQueue获取所有可用的数据对象（还可以指定获取数据的个数），通过该方法，可以提升获取数据效率；不需要多次分批加锁或释放锁。

四. **常见BlockingQueue**

　　在了解了BlockingQueue的基本功能后，让我们来看看BlockingQueue家庭大致有哪些成员？

![img](https://images0.cnblogs.com/blog2015/697611/201504/242030449842574.png)

　　1. **ArrayBlockingQueue**

　　基于数组的阻塞队列实现，在ArrayBlockingQueue内部，维护了一个定长数组，以便缓存队列中的数据对象，这是一个常用的阻塞队列，除了一个定长数组外，ArrayBlockingQueue内部还保存着两个整形变量，分别标识着队列的头部和尾部在数组中的位置。

　　ArrayBlockingQueue在生产者放入数据和消费者获取数据，都是共用同一个锁对象，由此也意味着两者无法真正并行运行，这点尤其不同于LinkedBlockingQueue；按照实现原理来分析，ArrayBlockingQueue完全可以采用分离锁，从而实现生产者和消费者操作的完全并行运行。Doug   Lea之所以没这样去做，也许是因为ArrayBlockingQueue的数据写入和获取操作已经足够轻巧，以至于引入独立的锁机制，除了给代码带来额外的复杂性外，其在性能上完全占不到任何便宜。   ArrayBlockingQueue和LinkedBlockingQueue间还有一个明显的不同之处在于，前者在插入或删除元素时不会产生或销毁任何额外的对象实例，而后者则会生成一个额外的Node对象。这在长时间内需要高效并发地处理大批量数据的系统中，其对于GC的影响还是存在一定的区别。而在创建ArrayBlockingQueue时，我们还可以控制对象的内部锁是否采用公平锁，默认采用非公平锁。

　　2.**LinkedBlockingQueue**

　　基于链表的阻塞队列，同ArrayListBlockingQueue类似，其内部也维持着一个数据缓冲队列（该队列由一个链表构成），当生产者往队列中放入一个数据时，队列会从生产者手中获取数据，并缓存在队列内部，而生产者立即返回；只有当队列缓冲区达到最大值缓存容量时（LinkedBlockingQueue可以通过构造函数指定该值），才会阻塞生产者队列，直到消费者从队列中消费掉一份数据，生产者线程会被唤醒，反之对于消费者这端的处理也基于同样的原理。而LinkedBlockingQueue之所以能够高效的处理并发数据，还因为其对于生产者端和消费者端分别采用了独立的锁来控制数据同步，这也意味着在高并发的情况下生产者和消费者可以并行地操作队列中的数据，以此来提高整个队列的并发性能。

　　作为开发者，我们需要注意的是，如果构造一个LinkedBlockingQueue对象，而没有指定其容量大小，LinkedBlockingQueue会默认一个类似无限大小的容量（Integer.MAX_VALUE），这样的话，如果生产者的速度一旦大于消费者的速度，也许还没有等到队列满阻塞产生，系统内存就有可能已被消耗殆尽了。

　　ArrayBlockingQueue和LinkedBlockingQueue是两个最普通也是最常用的阻塞队列，一般情况下，在处理多线程间的生产者消费者问题，使用这两个类足以。

　　下面的代码演示了如何使用BlockingQueue：

　　(1) 测试类



```
 1 import java.util.concurrent.BlockingQueue;
 2 import java.util.concurrent.ExecutorService;
 3 import java.util.concurrent.Executors;
 4 import java.util.concurrent.LinkedBlockingQueue; 
 5 
 6 public class BlockingQueueTest {
 7  
 8     public static void main(String[] args) throws InterruptedException {
 9         // 声明一个容量为10的缓存队列
10         BlockingQueue<String> queue = new LinkedBlockingQueue<String>(10);
11  
12         //new了三个生产者和一个消费者
13         Producer producer1 = new Producer(queue);
14         Producer producer2 = new Producer(queue);
15         Producer producer3 = new Producer(queue);
16         Consumer consumer = new Consumer(queue);
17  
18         // 借助Executors
19         ExecutorService service = Executors.newCachedThreadPool();
20         // 启动线程
21         service.execute(producer1);
22         service.execute(producer2);
23         service.execute(producer3);
24         service.execute(consumer);
25  
26         // 执行10s
27         Thread.sleep(10 * 1000);
28         producer1.stop();
29         producer2.stop();
30         producer3.stop();
31  
32         Thread.sleep(2000);
33         // 退出Executor
34         service.shutdown();
35     }
36 }
```



　　（2）生产者类



```
 1 import java.util.Random;
 2 import java.util.concurrent.BlockingQueue;
 3 import java.util.concurrent.TimeUnit;
 4 import java.util.concurrent.atomic.AtomicInteger;
 5  
 6 /**
 7  * 生产者线程
 8  * 
 9  * @author jackyuj
10  */
11 public class Producer implements Runnable {
12     
13     private volatile boolean  isRunning = true;//是否在运行标志
14     private BlockingQueue queue;//阻塞队列
15     private static AtomicInteger count = new AtomicInteger();//自动更新的值
16     private static final int DEFAULT_RANGE_FOR_SLEEP = 1000;
17  
18     //构造函数
19     public Producer(BlockingQueue queue) {
20         this.queue = queue;
21     }
22  
23     public void run() {
24         String data = null;
25         Random r = new Random();
26  
27         System.out.println("启动生产者线程！");
28         try {
29             while (isRunning) {
30                 System.out.println("正在生产数据...");
31                 Thread.sleep(r.nextInt(DEFAULT_RANGE_FOR_SLEEP));//取0~DEFAULT_RANGE_FOR_SLEEP值的一个随机数
32  
33                 data = "data:" + count.incrementAndGet();//以原子方式将count当前值加1
34                 System.out.println("将数据：" + data + "放入队列...");
35                 if (!queue.offer(data, 2, TimeUnit.SECONDS)) {//设定的等待时间为2s，如果超过2s还没加进去返回true
36                     System.out.println("放入数据失败：" + data);
37                 }
38             }
39         } catch (InterruptedException e) {
40             e.printStackTrace();
41             Thread.currentThread().interrupt();
42         } finally {
43             System.out.println("退出生产者线程！");
44         }
45     }
46  
47     public void stop() {
48         isRunning = false;
49     }
50 }
```



　　（3）消费者类



```
 1 import java.util.Random;
 2 import java.util.concurrent.BlockingQueue;
 3 import java.util.concurrent.TimeUnit;
 4  
 5 /**
 6  * 消费者线程
 7  * 
 8  * @author jackyuj
 9  */
10 public class Consumer implements Runnable {
11     
12     private BlockingQueue<String> queue;
13     private static final int DEFAULT_RANGE_FOR_SLEEP = 1000;
14  
15     //构造函数
16     public Consumer(BlockingQueue<String> queue) {
17         this.queue = queue;
18     }
19  
20     public void run() {
21         System.out.println("启动消费者线程！");
22         Random r = new Random();
23         boolean isRunning = true;
24         try {
25             while (isRunning) {
26                 System.out.println("正从队列获取数据...");
27                 String data = queue.poll(2, TimeUnit.SECONDS);//有数据时直接从队列的队首取走，无数据时阻塞，在2s内有数据，取走，超过2s还没数据，返回失败
28                 if (null != data) {
29                     System.out.println("拿到数据：" + data);
30                     System.out.println("正在消费数据：" + data);
31                     Thread.sleep(r.nextInt(DEFAULT_RANGE_FOR_SLEEP));
32                 } else {
33                     // 超过2s还没数据，认为所有生产线程都已经退出，自动退出消费线程。
34                     isRunning = false;
35                 }
36             }
37         } catch (InterruptedException e) {
38             e.printStackTrace();
39             Thread.currentThread().interrupt();
40         } finally {
41             System.out.println("退出消费者线程！");
42         }
43     }
44  
45     
46 }
```

![复制代码](https://common.cnblogs.com/images/copycode.gif)

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　**3. DelayQueue**

　　DelayQueue中的元素只有当其指定的延迟时间到了，才能够从队列中获取到该元素。DelayQueue是一个没有大小限制的队列，因此往队列中插入数据的操作（生产者）永远不会被阻塞，而只有获取数据的操作（消费者）才会被阻塞。

　　使用场景：

　　DelayQueue使用场景较少，但都相当巧妙，常见的例子比如使用一个DelayQueue来管理一个超时未响应的连接队列。

　　**4. PriorityBlockingQueue**

　　 基于优先级的阻塞队列（优先级的判断通过构造函数传入的Compator对象来决定），但需要注意的是PriorityBlockingQueue并不会阻塞数据生产者，而只会在没有可消费的数据时，阻塞数据的消费者。因此使用的时候要特别注意，生产者生产数据的速度绝对不能快于消费者消费数据的速度，否则时间一长，会最终耗尽所有的可用堆内存空间。在实现PriorityBlockingQueue时，内部控制线程同步的锁采用的是公平锁。

　　**5. SynchronousQueue**

　　 一种无缓冲的等待队列，类似于无中介的直接交易，有点像原始社会中的生产者和消费者，生产者拿着产品去集市销售给产品的最终消费者，而消费者必须亲自去集市找到所要商品的直接生产者，如果一方没有找到合适的目标，那么对不起，大家都在集市等待。相对于有缓冲的BlockingQueue来说，少了一个中间经销商的环节（缓冲区），如果有经销商，生产者直接把产品批发给经销商，而无需在意经销商最终会将这些产品卖给那些消费者，由于经销商可以库存一部分商品，因此相对于直接交易模式，总体来说采用中间经销商的模式会吞吐量高一些（可以批量买卖）；但另一方面，又因为经销商的引入，使得产品从生产者到消费者中间增加了额外的交易环节，单个产品的及时响应性能可能会降低。

　　声明一个SynchronousQueue有两种不同的方式，它们之间有着不太一样的行为。公平模式和非公平模式的区别:

　　如果采用公平模式：SynchronousQueue会采用公平锁，并配合一个FIFO队列来阻塞多余的生产者和消费者，从而体系整体的公平策略；

　　但如果是非公平模式（SynchronousQueue默认）：SynchronousQueue采用非公平锁，同时配合一个LIFO队列来管理多余的生产者和消费者，而后一种模式，如果生产者和消费者的处理速度有差距，则很容易出现饥渴的情况，即可能有某些生产者或者是消费者的数据永远都得不到处理。

五. 小结

　　BlockingQueue不光实现了一个完整队列所具有的基本功能，同时在多线程环境下，他还自动管理了多线间的自动等待于唤醒功能，从而使得程序员可以忽略这些细节，关注更高级的功能。



