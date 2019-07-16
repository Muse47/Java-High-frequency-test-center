# JVM

[TOC]



## 1、详细jvm内存模型

### 一、概念

#### 1、JVM内存模型

首先老规矩，祭上一张自己画的内存模型图
![内存模型](https://img-blog.csdn.net/20180308225715383?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMjIxNTIyNjE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
画的比较简陋，简单介绍一下，整个JVM占用的内存可分为两个大区，分别是线程共享区和线程私有区，线程共享区和JVM同生共死，所有线程均可访问此区域；而线程私有区顾名思义每个线程各自占有，与各自线程同生共死。这两个大区内部根据JVM规范定义又分为以下几个区：

##### 方法区（Method Area）

方法区主要是放一下类似类定义、常量、编译后的代码、静态变量等，在JDK1.7中，HotSpot VM的实现就是将其放在永久代中，这样的好处就是可以直接使用堆中的GC算法来进行管理，但坏处就是经常会出现内存溢出，即PermGen Space异常，所以在JDK1.8中，HotSpot VM取消了永久代，用元空间取而代之，元空间直接使用本地内存，理论上电脑有多少内存它就可以使用多少内存，所以不会再出现PermGen Space异常。

##### 堆（Heap）

几乎所有对象、数组等都是在此分配内存的，在JVM内存中占的比例也是极大的，也是GC垃圾回收的主要阵地，平时我们说的什么新生代、老年代、永久代也是指的这片区域，至于为什么要进行分代后面会解释。

##### 虚拟机栈（Java Stack）

当JVM在执行方法时，会在此区域中创建一个栈帧来存放方法的各种信息，比如返回值，局部变量表和各种对象引用等，方法开始执行前就先创建栈帧入栈，执行完后就出栈。

##### 本地方法栈（Native Method Stack）

和虚拟机栈类似，不过区别是专门提供给Native方法用的。

##### 程序计数器（Program Counter Register）

占用很小的一片区域，我们知道JVM执行代码是一行一行执行字节码，所以需要一个计数器来记录当前执行的行数。

### 2、堆内存

堆内存是JVM内存中占用较大的一块区域，对象都在此地分配内存。在堆中，又分为新生代及老年代，新生代中又分三个区域，分别是Eden，Survivor To,Survivor From。堆内存是JVM调优的重点区域，也是这篇博客重点讨论的内容。

### 3、提出问题

可能看到这里我们都会产生这样一个经典问题

    为何堆内存要进行分代？

##### 最简单的回收方式（标记-回收算法）

假设堆内存不进行分代，那么垃圾回收应该如何进行呢？我们可以大胆想象一下，在一大片内存空间中我们分配了若干个对象（假设图中一个黑色方块代表一个字节，分配的对象均占用两个字节）
![堆满](https://img-blog.csdn.net/20180309003935847?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMjIxNTIyNjE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
此时堆内存满了，是时候来一波垃圾回收操作了，通过某种分析算法，我们分析到某几个对象是需要进行回收（下述此类对象称之为回收对象），我们让无用对象就地清除，回收后的结果如下：
![堆一](https://img-blog.csdn.net/20180309005246236?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMjIxNTIyNjE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
我们可以看到，回收后的内存支离破碎的，虽然现在还有八个字节的内存空间，但只要有三个字节或以上的对象需要申请内存，那么这片支离破碎依旧无法为其分配内存，因为没有连续的空间。

##### 第一次演化（复制算法）

既然我们没法回收出连续的空间，那我们可以从一开始就把内存分两个大区，平时只用其中一个区，如下图
![回收二](https://img-blog.csdn.net/20180309010455704?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMjIxNTIyNjE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
当左边内存区满时，就开始一波回收操作，找到那些无需回收的对象（下述称此类对象为存活对象），将它们工工整整地复制到右边的区域中，接着将左边的区域来次大清理，清理后的结果如下：
![回收三](https://img-blog.csdn.net/20180309010846251?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMjIxNTIyNjE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
这样当需要再分配内存给对象时，就使用右边的区域，而右边的区域此时也有8个字节的连续空间供分配，当右边满了，再如法炮制，将存活对象复制到左边再将右边回收。貌似这样就解决了问题了，但是总感觉有什么地方不对，是的没错，每次只使用一半的内存，未免也太浪费了！

##### 第二次演化（标记-压缩算法）

我们依旧不分区，将整个内存用满， 开始回收垃圾时，我们将存活对象全部移动到左边，然后对边界外的内存进行清理，如下图
![回收四](https://img-blog.csdn.net/20180309012703247?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMjIxNTIyNjE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
这算是一种折中的做法，起码比第一次演化中的做法更加充分使用内存，也比最开始的做法的空闲内存更加连续。
在这里我们可以再往下思考，在第一次演化中，如果每次回收对象特别多，而存活对象特别少，那么只需要通过少数的复制操作和一次清除就可以实现回收，此时效率会特别高。而在第二次演化中，如果每次回收对象较少，而存活对象较多，则可以采取此策略进行回收确保最终剩余的空间是连续的空间。
到这里其实并不足够完善，毕竟上述几种演化都有缺点和优点，有没有办法可以取长补短呢？
在开发中，其实我们可以发现，大多数对象都是在方法体中new出来，new完使用后就不再使用了，此时该对象即可进行回收。所以这一类的对象有个特点就是朝生夕死。假如在方法执行完，该对象的引用还被持有着，证明该对象是比较重要的对象，越到后面要回收则越来越困难。这个情况不就刚好符合上述两种演化的情况，当对象刚出生时，我们可以将其使用演化一的方式进行回收，当使用演化一的方式回收不了的对象，则证明该对象为比较重要的对象，我们就可以采用演化二的方式进行回收。这样我们可以对我们的内存进行分区

##### 第一次分区

![分区一](https://img-blog.csdn.net/20180309015757868?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMjIxNTIyNjE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
当new对象时，内存均从上图右上方的区域申请，当右上方的内存区域满时，则进行一次复制算法进行垃圾回收。从上面的思考我们知道，绝大多数新对象都有朝生夕死的特点，所以在这次的垃圾回收中，存活的对象寥寥无几，然后存活的对象全部塞到右下方区域。在下一次垃圾回收到来时，根据上述分析，之前存活的对象绝大多数还会继续存活，我们将经历过一次垃圾回收的对象年龄+1，可见大多数的对象都熬不过两岁，一般在一岁时就被回收了。而当对象经历了多次垃圾回收仍然存活，此时它很难被回收了，我们可以将其移到左边的区域，另外右边上下俩区域都满了时，则通过垃圾回收将存活对象的那一边区域也移动到左边区域中。当左边区域满时，可通过标记-压缩算法进行垃圾回收。在这种分区方式中，左侧区域称之为老年代，而右侧区域则为新生代。新生代使用复制算法进行一次垃圾回收，称之为Minor GC,而复制完后如果老年代区域不够，也会触发老年代使用标记-压缩算法进行垃圾回收，称之为Major GC,一般Major GC会伴随着Minor GC，所以也称为Full GC。
在上述分区中，新生代仍然只有一半的区域可以用，之前使用一半区域的原因是考虑到有可能所有对象都是存活对象，这样才足够完全复制，但现在有老年代的存在，再考虑到此区域每一次回收时仅有少数对象需要复制，分区方式是否还有优化的空间呢？

##### 第二次分区

![第二次分区](https://img-blog.csdn.net/20180309212206932?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMjIxNTIyNjE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)



这个分区是在第一次分区的基础上，将新生代分为三部分，分别是伊甸园、幸存区S0，幸存区S1，伊甸园内存占比为8:1:1，S0与S1大小相同。对象的一生如下：
①所有对象都在伊甸园出生，当伊甸园占满时，开始进行一次Minor GC,此次GC会将已存活的对象复制到S0区中
②伊甸园区又被占满，此时又进行一次Minor GC，伊甸园存活的对象又复制到S0区。
③在若干次GC后，幸存区S0也满了，此时Minor GC会对伊甸园和幸存区S0的
做一次垃圾回收，将两个区存活的对象复制到幸存区S1中，再把伊甸园和S0清空，最后把S1的内存与S0交换，此时S1又腾空了，S0剩下一些老对象。
④又经历若干次GC，幸存区S0已经放满了经历过N次GC都回收不了的老对象，此时会将老对象复制到老年代中，腾空幸存区。
⑤并非当幸存区被老对象占满才复制到老年代中，当老对象年龄达到15岁，即经历过15次GC都还活着的，也会复制到老年代中，另外伊甸园中如果诞生了一个比幸存区还大的对象，那么该对象回收不了时，也会直接送入到老年代中。
⑥又经历过若干次GC后，老年代也满了，那么此时它会进行一次Major GC。

##### 动图演示

上述过程使用文字描述可能比较抽象，下面用动图简单来演示一次Minor GC。
![动图](https://img-blog.csdn.net/20180309234356298?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMjIxNTIyNjE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 二、垃圾回收涉及的算法

在垃圾回收中，涉及的算法主要有以下五个

> 引用计数算法
> 可达性分析
> 标记-回收算法
> 标记-压缩算法
> 复制算法

前两个算法用于判断对象是否需要回收，其原理简单讲，引用计数算法就是计算对象被谁引用，一旦有其它对象引用此对象，引用次数加一，而GC时引用次数大于零的对象则判断为存活对象，但此算法无法解决循环引用问题，如A引用B，B引用A，此时A与B均无法回收，所以现在JVM不采用此算法；而可达性算法则从GCRoot出发，若A引用B，B引用C，则通过A可以到达C，此时ABC三个对象均不进行回收。后面的三个算法为回收策略，其思路在第一章有提及，在这里就不加赘述，下面总结一下三个算法优缺点：

| 算法          | 优点               | 缺点                             |
| ------------- | ------------------ | -------------------------------- |
| 标记-回收算法 | 暂无               | 标记和清除效率低、可用空间不连续 |
| 标记-压缩算法 | 实现简单，运行高效 | 内存空间利用不充分               |
| 复制算法      | 内存空间利用率高   | 性能较低                         |

### 三、JVM常用参数

    Xss：每个线程的栈大小
    Xms：堆空间的初始值
    Xmx：堆空间最大值、默认为物理内存的1/4，一般Xms与Xmx最好一样
    Xmn：年轻代的大小
    XX:NewRatio ：新生代和年老代的比例
    XX:SurvivorRatio ：伊甸园区和幸存区的占用比例
    XX:PermSize：设定内存的永久保存区域（1.8已废除）
    XX:MetaspaceSize：1.8使用此参数替代上述参数
    XX:MaxPermSize：设定最大内存的永久保存区域（1.8已废除）
    XX:MaxMetaspaceSize：1.8使用此参数替代上述参数

### 四、附录-演示代码

此代码为第一章中动图所使用的测试代码

    public class OutOfMemoryErrorTest {
        public static void main(String[] args) throws Throwable {
            Random r = new Random();
            List<TestObject[]> testObjectList = new ArrayList<TestObject[]>();
            while (true) {
                try {
                    TestObject[] testObjects = new TestObject[2048];
                    // 模拟30%左右的对象为有用对象
                    if (r.nextInt(10) > 4) {
                        testObjectList.add(testObjects);
                    }
                    Thread.sleep(1);
                } catch (Throwable t) {
                    throw t;
                }
            }
        }
    }
    class TestObject {
    
        private String name;
    
        private int age;
    
        public String getName() {
            return name;
        }
    
        public void setName(String name) {
            this.name = name;
        }
    
        public int getAge() {
            return age;
        }
    
        public void setAge(int age) {
            this.age = age;
        }
    }


## 2、讲讲什么情况下回出现内存溢出，内存泄漏？



**内存溢出 out of memory**，是指程序在申请内存时，没有足够的内存空间供其使用，出现out of memory；比如申请了一个integer,但给它存了long才能存下的数，那就是内存溢出。

**内存泄露 memory leak**，是指程序在申请内存后，无法释放已申请的内存空间，一次内存泄露危害可以忽略，但内存泄露堆积后果很严重，无论多少内存,迟早会被占光。

memory leak会最终会导致out of memory！

内存溢出就是你要求分配的内存超出了系统能给你的，系统不能满足需求，于是产生溢出。 

​     内存泄漏是指你向系统申请分配内存进行使用(new)，可是使用完了以后却不归还(delete)，结果你申请到的那块内存你自己也不能再访问（也许你把它的地址给弄丢了），而系统也不能再次将它分配给需要的程序。一个盘子用尽各种方法只能装4个果子，你装了5个，结果掉倒地上不能吃了。这就是溢出！比方说栈，栈满时再做进栈必定产生空间溢出，叫上溢，栈空时再做退栈也产生空间溢出，称为下溢。就是分配的内存不足以放下数据项序列,称为内存溢出. 

   以发生的方式来分类，内存泄漏可以分为4类： 

1. 常发性内存泄漏。发生内存泄漏的代码会被多次执行到，每次被执行的时候都会导致一块内存泄漏。 

2. 偶发性内存泄漏。发生内存泄漏的代码只有在某些特定环境或操作过程下才会发生。常发性和偶发性是相对的。对于特定的环境，偶发性的也许就变成了常发性的。所以测试环境和测试方法对检测内存泄漏至关重要。 

3. 一次性内存泄漏。发生内存泄漏的代码只会被执行一次，或者由于算法上的缺陷，导致总会有一块仅且一块内存发生泄漏。比如，在类的构造函数中分配内存，在析构函数中却没有释放该内存，所以内存泄漏只会发生一次。 

4. 隐式内存泄漏。程序在运行过程中不停的分配内存，但是直到结束的时候才释放内存。严格的说这里并没有发生内存泄漏，因为最终程序释放了所有申请的内存。但是对于一个服务器程序，需要运行几天，几周甚至几个月，不及时释放内存也可能导致最终耗尽系统的所有内存。所以，我们称这类内存泄漏为隐式内存泄漏。 

从用户使用程序的角度来看，内存泄漏本身不会产生什么危害，作为一般的用户，根本感觉不到内存泄漏的存在。真正有危害的是内存泄漏的堆积，这会最终消耗尽系统所有的内存。从这个角度来说，一次性内存泄漏并没有什么危害，因为它不会堆积，而隐式内存泄漏危害性则非常大，因为较之于常发性和偶发性内存泄漏它更难被检测到 

内存溢出的原因以及解决方法

**引起内存溢出的原因有很多种，小编列举一下常见的有以下几种：**

1.内存中加载的数据量过于庞大，如一次从数据库取出过多数据；
2.集合类中有对对象的引用，使用完后未清空，使得JVM不能回收；
3.代码中存在死循环或循环产生过多重复的对象实体；
4.使用的第三方软件中的BUG；
5.启动参数内存值设定的过小

**内存溢出的解决方案：**

第一步，修改JVM启动参数，直接增加内存。(-Xms，-Xmx参数一定不要忘记加。)

第二步，检查错误日志，查看“OutOfMemory”错误前是否有其它异常或错误。

第三步，对代码进行走查和分析，找出可能发生内存溢出的位置。

重点排查以下几点：
1.检查对数据库查询中，是否有一次获得全部数据的查询。一般来说，如果一次取十万条记录到内存，就可能引起内存溢出。这个问题比较隐蔽，在上线前，数据库中数据较少，不容易出问题，上线后，数据库中数据多了，一次查询就有可能引起内存溢出。因此对于数据库查询尽量采用分页的方式查询。

2.检查代码中是否有死循环或递归调用。

3.检查是否有大循环重复产生新对象实体。

4.检查对数据库查询中，是否有一次获得全部数据的查询。一般来说，如果一次取十万条记录到内存，就可能引起内存溢出。这个问题比较隐蔽，在上线前，数据库中数据较少，不容易出问题，上线后，数据库中数据多了，一次查询就有可能引起内存溢出。因此对于数据库查询尽量采用分页的方式查询。

5.检查List、MAP等集合对象是否有使用完后，未清除的问题。List、MAP等集合对象会始终存有对对象的引用，使得这些对象不能被GC回收。

第四步，使用内存查看工具动态查看内存使用情况



> 更多内存溢出解决思路https://www.cnblogs.com/200911/p/3965108.html





## 3、说说Java线程栈

### 一、线程栈模型

线程栈模型是理解线程调度原理以及线程执行过程的基础。线程栈是指某时刻时内存中线程调度的栈信息，当前调用的方法总是位于栈顶，线程栈的内容是随着线程的运行状态变化而变化的，研究线程栈必须选择一个运行的时刻(指代码运行到什么地方)


上图中的栈A是主线程main的运行栈信息，当执行new JavaThreadDemo().threadMethod();方法时，threadMethod方法位于主线程栈中的栈顶，在threadMethod方法中运行的start()方法新建立了一个线程，此时，新建立的线程也将拥有自己的线程栈B，可以看到在不同运行时刻栈的信息在发生变化，栈A的变化可以从上图中看出来。此时栈A和栈B并行运行，main线程和新建立的线程并行运行。由此可以看出方法调用和线程启动的区别。方法调用只是在原来的线程栈中调用方法的代码，而线程启动会新建立一个独属于自己的线程栈来运行自己的这个线程。

### 二、线程的生命周期

线程的生命周期包括五种状态：新建、可运行状态、运行状态、阻塞、死亡。


新建：创建了一个新的线程对象，但是还没有调用线程中start()方法，此时线程处于新建状态。

可运行状态：调用线程的start()方法，线程进入可运行状态，此时线程等待着JVM的调度程序来执行，就是说我已经准备好了，可以上战场了，只等待一个机会就可以变为运行状态了。

运行状态：线程调度程序从众多的可运行状态线程中选择一个线程来运行，好比主帅选择一个大将来上阵较量。这也是线程进入运行状态的唯一方式，必须由JVM来调度。

阻塞状态：线程的等待、睡眠和阻塞统称为阻塞状态，此时线程依然是活得，处于待命状态，知道某个条件出现时，即可返回可运行状态。

死亡状态：当线程的run()方法执行完毕后，线程即结束。此时线程已经不在存在，它所占用的所有资源都会被回收。

### 三、线程阻塞

线程的阻塞有多种，常见的包括三项(IO阻塞不讨论)

1、睡眠

2、等待

3、获取线程锁而阻塞

睡眠状态：线程的sleep()方法可以使一个正在执行的线程进入睡眠状态。线程睡眠时，不会返回可运行状态，当睡眠时间到时才返回可运行状态。

    @Override
    	public void run() {
    		for(int i = 0; i < 5; i++) {
    			try {
    				Thread.sleep(50);//模拟耗时操作
    				System.out.println(name + ":" + i);
    				
    			} catch (InterruptedException e) {
    				e.printStackTrace();
    			}
    		}
    		
    	}

调用线程的静态方法Thread.sleep()可以使线程睡眠，参数为毫秒。

1、线程处于睡眠状态时，JVM调度程序会暂停此线程的执行并来执行其他的处于可运行状态的线程。

2、sleep()方法是Thread的静态方法，只能控制当前线程的睡眠。

3、线程睡眠时间到之后会返回到可运行状态，而不是运行状态。

线程的优先级和线程让步yield()

线程通过调用Thread.yield()方法来实现线程的让步。yield()方法的作用为暂停当前正在执行的线程对象，让出处理器资源，让处理器来执行其他线程。

Java中的线程存在优先级，优先级的范围为1——10,之间。JVM的线程调度程序是基于优先级的抢先调度机制。高优先级的线程比低优先级的线程有更高的几率得到执行，只是几率高一点，并不是说低优先级的线程必须等待高优先级的线程执行完成后才开始执行，只是得到的资源分配的比例不一样而已。（线程优先级在实际应用中并不能按你预期的来执行）

当线程池中的线程都具有相同的优先级时，JVM的调度程序随机选择处于可运行状态的线程来执行，此时可能有两种执行过程：(1)选择一个线程运行，直到此线程阻塞或者运行到死亡。(2)时间分片机制，为线程池中的每个线程都提供均匀的运行机会。

设置线程的优先级可以通过调用线程的setPriority(int)方法来设置线程的优先级。默认的线程优先级为5，JVM不会随便更改一个线程的优先级，但是不同系统中的线程优先级的范围可能不同，当JVM不能识别10个不同的优先级时，JVM会将这些优先级进行两个或者多个的合并，此时合并前的不同优先级的线程最后会被合并成一个优先级的线程。

Thread类中定义了三个常量来定义优先级的范围：

static int MAX_PRIORITY 线程具有的最大优先级

static int MIN_PRIORITY  线程具有的最小优先级

static int NORM_PRIORITY 线程具有的默认优先级


使线程让步的yield()方法

Thread.yield()方法可以暂停当前正在执行的线程，使处理器来有机会执行其他线程，即可以让当前线程做出让步，也让其他线程有机会被执行。线程执行yield()方法后，线程就从运行状态回到了可运行状态，使具有相同优先级的线程获得机会来执行。使用yield()方法可以做到在相同优先级的线程之间适当的轮换执行。但是，实际中无法保证yield()方法达到让步，因为让步的线程从运行状态回到了可运行状态，那么此线程还可以被JVM的调度机制来选中来执行。


join方法

Thread的非静态方法join()让一个线程B加入到另一个线程A的尾部，直到线程A执行完毕后，线程B才能继续执行。这相当于打断线程B的执行来先执行了线程A，类似与在main()函数中调用了另一个函数，直到此函数执行完毕后，main()函数才继续执行。

join的方法中有一个带时间参数的版本，t.join(5000)让线程等待5000毫秒，如果超过这个时间，则停止等待，变为可运行状态。


让线程暂时离开运行状态的三中方法：

1、调用线程的sleep()方法，使线程睡眠一段时间

2、调用线程的yield()方法，使线程暂时回到可运行状态，来使其他线程有机会执行。

3、调用线程的join()方法，使当前线程停止执行，知道当前线程中加入的线程执行完毕后，当前线程才可以执行。





## 4、JVM 年轻代到年老代的晋升过程的判断条件是什么呢？

### 几种 GC 方式

    Minor GC：年轻代，频率高，速度快
    Major GC：老年代
    Full GC：整个堆（年轻代，老年代）

### 对象的生命历程

一般新生对象要进入Eden 区，Eden 区被填满时，要进行GC，GC之后还存活的对象将被复制到两个Survivor 区域中的一个。假定该Survivor 为From 区，From 区被填满之后，这个区域也要进行GC，GC 之后存活的的对象将会复制到To 区，From区清空。To区也被填满时，之前从From 区复制过来的那部分对象仍在活动则进入老年代。（From，To 必有一区会是空的）

### 对象年龄与转移时机

    通过年龄计数器判断一个对象是否需要转移。对象每经过一个GC 仍然活着，年龄计数器加1。当年龄超过设定的值，则将其通过担保机制转移到老年代。
    也可以动态判定，当Survivor 中 年龄相同的对象超半数，则年龄大于该年龄的对象转移到老年代，无需等待到达设置的最大年龄值。
    大对象直接进入老年代。

### GC 安全检查

    在Minor GC 之前，检查老年代的可用空间是否大于年轻代的对象总和，若大于则是一次安全的Minor GC，不大于[ 允许担保失败]，则比较历次晋升到老年代的对象的平均大小是否大于老年代的最大可用空间，若大于则进行一次冒险的Minor GC。
    有可能老年代不满足空间需求（不大于，并且不允许担保失败），则进行一次Full GC。
    Minor GC 存在的原因是，年轻代只使用要给Survivor 保存仍存活的对象。
    在一次安全Minor GC 中，仍然存活的对象不能在另一个Survivor 完全容纳，则会通过担保机制进入老年代。

### 老年代的对象

    大对象（字符串与数组），即超过了设定值的对象
    长期存活的对象




## 5、JVM 出现 fullGC 很频繁，怎么去线上排查问题？

这个题目考查的是工程经验，没有确切答案



堆内存划分为 Eden、Survivor 和 Tenured/Old 空间，如下图所示：

![](https://img-blog.csdn.net/20150731163641941?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

从年轻代空间（包括 Eden 和 Survivor 区域）回收内存被称为 Minor GC，对老年代GC称为Major GC,而Full GC是对整个堆来说的，在最近几个版本的JDK里默认包括了对永生带即方法区的回收（JDK8中无永生带了），出现Full GC的时候经常伴随至少一次的Minor GC,但非绝对的。Major GC的速度一般会比Minor GC慢10倍以上。下边看看有那种情况触发JVM进行Full GC及应对策略。

### 1、System.gc()方法的调用

此方法的调用是建议JVM进行Full GC,虽然只是建议而非一定,但很多情况下它会触发 Full GC,从而增加Full GC的频率,也即增加了间歇性停顿的次数。强烈影响系建议能不使用此方法就别使用，让虚拟机自己去管理它的内存，可通过通过-XX:+ DisableExplicitGC来禁止RMI调用System.gc。

### 2、老年代代空间不足

老年代空间只有在新生代对象转入及创建为大对象、大数组时才会出现不足的现象，当执行Full GC后空间仍然不足，则抛出如下错误：
java.lang.OutOfMemoryError: Java heap space 
为避免以上两种状况引起的Full GC，调优时应尽量做到让对象在Minor GC阶段被回收、让对象在新生代多存活一段时间及不要创建过大的对象及数组。

### 3、永生区空间不足


JVM规范中运行时数据区域中的方法区，在HotSpot虚拟机中又被习惯称为永生代或者永生区，Permanet Generation中存放的为一些class的信息、常量、静态变量等数据，当系统中要加载的类、反射的类和调用的方法较多时，Permanet Generation可能会被占满，在未配置为采用CMS GC的情况下也会执行Full GC。如果经过Full GC仍然回收不了，那么JVM会抛出如下错误信息：
java.lang.OutOfMemoryError: PermGen space 
为避免Perm Gen占满造成Full GC现象，可采用的方法为增大Perm Gen空间或转为使用CMS GC。

### 4、CMS GC时出现promotion failed和concurrent mode failure


对于采用CMS进行老年代GC的程序而言，尤其要注意GC日志中是否有promotion failed和concurrent mode failure两种状况，当这两种状况出现时可能

会触发Full GC。
promotion failed是在进行Minor GC时，survivor space放不下、对象只能放入老年代，而此时老年代也放不下造成的；concurrent mode failure是在

执行CMS GC的过程中同时有对象要放入老年代，而此时老年代空间不足造成的（有时候“空间不足”是CMS GC时当前的浮动垃圾过多导致暂时性的空间不足触发Full GC）。
对措施为：增大survivor space、老年代空间或调低触发并发GC的比率，但在JDK 5.0+、6.0+的版本中有可能会由于JDK的bug29导致CMS在remark完毕

后很久才触发sweeping动作。对于这种状况，可通过设置-XX: CMSMaxAbortablePrecleanTime=5（单位为ms）来避免。

### 5、统计得到的Minor GC晋升到旧生代的平均大小大于老年代的剩余空间


这是一个较为复杂的触发情况，Hotspot为了避免由于新生代对象晋升到旧生代导致旧生代空间不足的现象，在进行Minor GC时，做了一个判断，如果之

前统计所得到的Minor GC晋升到旧生代的平均大小大于旧生代的剩余空间，那么就直接触发Full GC。
例如程序第一次触发Minor GC后，有6MB的对象晋升到旧生代，那么当下一次Minor GC发生时，首先检查旧生代的剩余空间是否大于6MB，如果小于6MB，

则执行Full GC。
当新生代采用PS GC时，方式稍有不同，PS GC是在Minor GC后也会检查，例如上面的例子中第一次Minor GC后，PS GC会检查此时旧生代的剩余空间是否

大于6MB，如小于，则触发对旧生代的回收。
除了以上4种状况外，对于使用RMI来进行RPC或管理的Sun JDK应用而言，默认情况下会一小时执行一次Full GC。可通过在启动时通过- java -

Dsun.rmi.dgc.client.gcInterval=3600000来设置Full GC执行的间隔时间或通过-XX:+ DisableExplicitGC来禁止RMI调用System.gc。

### 6、堆中分配很大的对象

所谓大对象，是指需要大量连续内存空间的java对象，例如很长的数组，此种对象会直接进入老年代，而老年代虽然有很大的剩余空间，但是无法找到足够大的连续空间来分配给当前对象，此种情况就会触发JVM进行Full GC。

为了解决这个问题，CMS垃圾收集器提供了一个可配置的参数，即-XX:+UseCMSCompactAtFullCollection开关参数，用于在“享受”完Full GC服务之后额外免费赠送一个碎片整理的过程，内存整理的过程无法并发的，空间碎片问题没有了，但提顿时间不得不变长了，JVM设计者们还提供了另外一个参数 -XX:CMSFullGCsBeforeCompaction,这个参数用于设置在执行多少次不压缩的Full GC后,跟着来一次带压缩的。



## 6、类加载为什么要使用双亲委派模式，有没有什么场景是打破了这个模式？



### 一、前言

笔者曾经阅读过周志明的《深入理解Java虚拟机》这本书，阅读完后自以为对jvm有了一定的了解，然而当真正碰到问题的时候，才发现自己读的有多粗糙，也体会到只有实践才能加深理解，正应对了那句话——“Talk  is cheap, show me the  code”。前段时间，笔者同事提出了一个关于类加载器破坏双亲委派的问题，以我们常见到的数据库驱动Driver为例，为什么要实现破坏双亲委派，下面一起来重温一下。





### 二、双亲委派

想要知道为什么要破坏双亲委派，就要先从什么是双亲委派说起，在此之前，我们先要了解一些概念：

- 对于任意一个类，都需要由**加载它的类加载器**和这个**类本身**来一同确立其在Java虚拟机中的**唯一性**。

什么意思呢？我们知道，判断一个类是否相同，通常用equals()方法，isInstance()方法和isAssignableFrom()方法。来判断，对于同一个类，如果没有采用相同的类加载器来加载，在调用的时候，会产生意想不到的结果：

```
public class DifferentClassLoaderTest {

    public static void main(String[] args) throws Exception {
        ClassLoader classLoader = new ClassLoader() {
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                String fileName = name.substring(name.lastIndexOf(".") + 1) + ".class";
                InputStream stream = getClass().getResourceAsStream(fileName);
                if (stream == null) {
                    return super.loadClass(name);
                }
                try {
                    byte[] b = new byte[stream.available()];
                    // 将流写入字节数组b中
                    stream.read(b);
                    return defineClass(name, b, 0, b.length);
                } catch (IOException e) {
                    e.printStackTrace();
                }

                return super.loadClass(name);
            }
        };
        Object obj = classLoader.loadClass("jvm.DifferentClassLoaderTest").newInstance();
        System.out.println(obj.getClass());
        System.out.println(obj instanceof DifferentClassLoaderTest);

    }
}
```

输出结果：

```
class jvm.DifferentClassLoaderTest
false
```

如果在通过classLoader实例化的使用，直接转化成DifferentClassLoaderTest对象：

```
DifferentClassLoaderTest obj = (DifferentClassLoaderTest) classLoader.loadClass("jvm.DifferentClassLoaderTest").newInstance();
```

就会直接报`java.lang.ClassCastException:`，因为两者不属于同一类加载器加载，所以不能转化！



#### 2.1、为什么需要双亲委派

基于上述的问题：如果不是同一个类加载器加载，即时是相同的class文件，也会出现判断不想同的情况，从而引发一些意想不到的情况，为了保证相同的class文件，在使用的时候，是相同的对象，jvm设计的时候，采用了双亲委派的方式来加载类。

> 双亲委派：如果一个类加载器收到了加载某个类的请求,则该类加载器并不会去加载该类,而是把这个请求委派给父类加载器,每一个层次的类加载器都是如此,因此所有的类加载请求最终都会传送到顶端的启动类加载器;只有当父类加载器在其搜索范围内无法找到所需的类,并将该结果反馈给子类加载器,子类加载器会尝试去自己加载。

这里有几个流程要注意一下：

1. 子类先委托父类加载
2. 父类加载器有自己的**加载范围**，范围内没有找到，则不加载，并返回给子类
3. 子类在收到父类无法加载的时候，才会自己去加载

jvm提供了三种系统加载器：

1. 启动类加载器（Bootstrap ClassLoader）：C++实现，在java里无法获取，**负责加载/lib**下的类。
2. 扩展类加载器（Extension ClassLoader）： Java实现，可以在java里获取，**负责加载/lib/ext**下的类。
3. 系统类加载器/应用程序类加载器（Application ClassLoader）：是与我们接触对多的类加载器，我们写的代码默认就是由它来加载，ClassLoader.getSystemClassLoader返回的就是它。

附上三者的关系：

![双亲委派图](https://images2018.cnblogs.com/blog/1256203/201807/1256203-20180714171531925-1737231049.png)



补充：

> 为什么需要双亲委派模型呢？假设没有双亲委派模型，试想一个场景：
>
> 黑客自定义一个java.lang.String类，该String类具有系统的String类一样的功能，只是在某个函数稍作修改。比如equals函数，这个函数经常使用，如果在这这个函数中，黑客加入一些“病毒代码”。并且通过自定义类加载器加入到JVM中。此时，如果没有双亲委派模型，那么JVM就可能误以为黑客自定义的java.lang.String类是系统的String类，导致“病毒代码”被执行。
>
> 而有了双亲委派模型，黑客自定义的java.lang.String类永远都不会被加载进内存。因为首先是最顶端的类加载器加载系统的java.lang.String类，最终自定义的类加载器无法加载java.lang.String类。

#### 2.2、双亲委派的实现

双亲委派的实现其实并不复杂，其实就是一个递归，我们一起来看一下`ClassLoader`里的代码：

```
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
    {
        // 同步上锁
        synchronized (getClassLoadingLock(name)) {
            // 先查看这个类是不是已经加载过
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    // 递归，双亲委派的实现，先获取父类加载器，不为空则交给父类加载器
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    // 前面提到，bootstrap classloader的类加载器为null，通过find方法来获得
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                }

                if (c == null) {
                    // 如果还是没有获得该类，调用findClass找到类
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // jvm统计
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            // 连接类
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```





### 三、破坏双亲委派

#### 3.1、为什么需要破坏双亲委派？

因为在某些情况下父类加载器需要委托子类加载器去加载class文件。受到加载范围的限制，父类加载器无法加载到需要的文件，以Driver接口为例，由于Driver接口定义在jdk当中的，而其实现由各个数据库的服务商来提供，比如mysql的就写了`MySQL Connector`，那么问题就来了，DriverManager（也由jdk提供）要加载各个实现了Driver接口的实现类，然后进行管理，但是DriverManager由启动类加载器加载，只能记载JAVA_HOME的lib下文件，而其实现是由服务商提供的，由系统类加载器加载，这个时候就需要启动类加载器来委托子类来加载Driver实现，从而破坏了双亲委派，这里仅仅是举了破坏双亲委派的其中一个情况。

#### 3.2、破坏双亲委派的实现

我们结合Driver来看一下在spi（Service Provider Inteface）中如何实现破坏双亲委派。

先从DriverManager开始看，平时我们通过DriverManager来获取数据库的Connection：

```
String url = "jdbc:mysql://localhost:3306/testdb";
Connection conn = java.sql.DriverManager.getConnection(url, "root", "root"); 
```

在调用DriverManager的时候，会先初始化类，调用其中的静态块：

```
static {
    loadInitialDrivers();
    println("JDBC DriverManager initialized");
}

private static void loadInitialDrivers() {
    ...
        // 加载Driver的实现类
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {

                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                Iterator<Driver> driversIterator = loadedDrivers.iterator();
                try{
                    while(driversIterator.hasNext()) {
                        driversIterator.next();
                    }
                } catch(Throwable t) {
                }
                return null;
            }
        });
    ...
}
```

为了节约空间，笔者省略了一部分的代码，重点来看一下`ServiceLoader.load(Driver.class)`：

```
public static <S> ServiceLoader<S> load(Class<S> service) {
    // 获取当前线程中的上下文类加载器
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}
```

可以看到，load方法调用获取了当前线程中的上下文类加载器，那么上下文类加载器放的是什么加载器呢？

```
public Launcher() {
    ...
    try {
        this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);
    } catch (IOException var9) {
        throw new InternalError("Could not create application class loader", var9);
    }
    Thread.currentThread().setContextClassLoader(this.loader);
    ...
}
```

在`sun.misc.Launcher`中，我们找到了答案，在Launcher初始化的时候，会获取AppClassLoader，然后将其设置为上下文类加载器，而这个AppClassLoader，就是之前上文提到的系统类加载器Application ClassLoader，所以**上下文类加载器默认情况下就是系统加载器**。

继续来看下`ServiceLoader.load(service, cl)`：

```
public static <S> ServiceLoader<S> load(Class<S> service,
                                        ClassLoader loader){
    return new ServiceLoader<>(service, loader);
}

private ServiceLoader(Class<S> svc, ClassLoader cl) {
    service = Objects.requireNonNull(svc, "Service interface cannot be null");
    // ClassLoader.getSystemClassLoader()返回的也是系统类加载器
    loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
    acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
    reload();
}

public void reload() {
    providers.clear();
    lookupIterator = new LazyIterator(service, loader);
}
```

上面这段就不解释了，比较简单，然后就是看LazyIterator迭代器：

```
private class LazyIterator implements Iterator<S>{
    // ServiceLoader的iterator()方法最后调用的是这个迭代器里的next
    public S next() {
        if (acc == null) {
            return nextService();
        } else {
            PrivilegedAction<S> action = new PrivilegedAction<S>() {
                public S run() { return nextService(); }
            };
            return AccessController.doPrivileged(action, acc);
        }
    }
    
    private S nextService() {
        if (!hasNextService())
            throw new NoSuchElementException();
        String cn = nextName;
        nextName = null;
        Class<?> c = null;
        // 根据名字来加载类
        try {
            c = Class.forName(cn, false, loader);
        } catch (ClassNotFoundException x) {
            fail(service,
                 "Provider " + cn + " not found");
        }
        if (!service.isAssignableFrom(c)) {
            fail(service,
                 "Provider " + cn  + " not a subtype");
        }
        try {
            S p = service.cast(c.newInstance());
            providers.put(cn, p);
            return p;
        } catch (Throwable x) {
            fail(service,
                 "Provider " + cn + " could not be instantiated",
                 x);
        }
        throw new Error();          // This cannot happen
    }
    
    public boolean hasNext() {
        if (acc == null) {
            return hasNextService();
        } else {
            PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {
                public Boolean run() { return hasNextService(); }
            };
            return AccessController.doPrivileged(action, acc);
        }
    }
    
    
    private boolean hasNextService() {
        if (nextName != null) {
            return true;
        }
        if (configs == null) {
            try {
                // 在classpath下查找META-INF/services/java.sql.Driver名字的文件夹
                // private static final String PREFIX = "META-INF/services/";
                String fullName = PREFIX + service.getName();
                if (loader == null)
                    configs = ClassLoader.getSystemResources(fullName);
                else
                    configs = loader.getResources(fullName);
            } catch (IOException x) {
                fail(service, "Error locating configuration files", x);
            }
        }
        while ((pending == null) || !pending.hasNext()) {
            if (!configs.hasMoreElements()) {
                return false;
            }
            pending = parse(service, configs.nextElement());
        }
        nextName = pending.next();
        return true;
    }

}
```

好了，这里基本就差不多完成整个流程了，一起走一遍：

![spi加载过程](https://images2018.cnblogs.com/blog/1256203/201807/1256203-20180714171604349-2005119436.png)



## 7、类的实例化顺序

## 普通类的初始化（不存在继承，内部类的时候）

为了更详细的验证类的初始化顺序，首先我创建了一个被另一个类使用的类**B.java**

```
public class B {
    private int varOneInB = initInt("varOneInB"); // 6 14
    private static int staticVarOneInB = initInt("staticVarOneB");  // 4  
    private int varTwoInB = initInt("varTwoInB"); // 7 15
    private static int staticvarTwoInB = initInt("staticvarTwoInB"); // 5
    
    /**
     * 构造方法
     */
    public B() {
        System.out.println("B  constructor"); // 8 16
    }
    
    /**
     * 用于对int类型的变量赋值
     * @param varName
     * @return
     */
    private static int initInt(String varName) {
        System.out.println(varName + " init");
        return 2017;
    }
}
```

然后我创建了一个**A**类来验证初始化顺序，并且在该类中同时使用的static变量和static块等。

```
public class A {
    private int varOneInA = initInt("varOneInA"); // 11
    private static int staticVarOneInA = initInt("staticVarOneInA"); // 1 
    {
          int varTwoInA = initInt("varTwoInA"); // 12
    }
    static {
          int staticvarTwoInA = initInt("staticvarTwoInA");  // 2
    }
    private B b = new B(); // 13
    private static B staticB = new B(); // 3
    
    /**
     * 构造方法
     */
    public A() {
        System.out.println("A  constructor"); // 17
    }
    
    /**
     * 用于对int型变量赋值
     * @param varName
     * @return
     */
    private static int initInt(String varName) {
        System.out.println(varName + " init");
        return 2017;
    }
    
    public void run() {
        System.out.println("run be called");// 23
    }
    
    public static void main (String[] args) {
        System.out.println("start running");// 9
        A a = new A();// 10
        a.run();// 18
    }
}
```

运行后结果为：

```
staticVarOneInA init
staticvarTwoInA init
staticVarOneB init
staticvarTwoInB init
varOneInB init
varTwoInB init
B  constructor
start running
varOneInA init
varTwoInA init
varOneInB init
varTwoInB init
B  constructor
A  constructor
run be called
```

对《Think in java》这本书里面的关于初始化顺序的总结进行归纳如下：

> **注意：**即使变量定义散布于方法定义之间，它们仍旧会在任何方法（包括构造器）被调用之前得到初始化。

1. 即使没有显示地使用static关键字，构造器实际上也是静态方法。因此，当首次创建类的对象时（构造器可以看出静态方法），或者类的静态方法/静态域被首次访问时，Java解释器必须查找类路径。
2. 然后载入class，有关静态初始化的所有动作都会执行（所以静态初始化只在Class对象首次被加载的时候进行一次）。
3. 当使用new创建对象的时候，首先将在堆上为对象分配足够的存储空间。
4. 这块存储空间会被清零，这就自动将Dog对象中的所有基本类型数据都设置成了默认值（对数字来说就是0，对布尔类型和字符类型也相同），而引用就则被设置成了null。
5. 执行出现于字段定义出的初始化动作。
6. 执行构造器。

有了上面的知识点，再来看上面的结果。我用数字1 2 3做了标记，括号后的阿拉伯数字表示上面代码对应的地方。

- 在类A中执行main方法，由于main()是静态方法，必须加载A类，然后起静态域staticVarOneInA(1)，staticvarTwoInA(2)，staticB(3)被初始化。
- 在staticB被初始化的时候，导致B类被加载，因为是第一次加载，对静态域进行初始化，因此staticVarOneInB(4)，staticvarTwoInB(5)被初始化。
- 之后顺序初始化varOneInB(6)，varTwoInB(7)，执行构造器B(8)。
- 在A静态域初始化后，回到main()方法，打印出了“start running”(9),在new A()(10)的时候，分配a对象的空间，开始顺序初始化varOneInA(11)，varTwoInA(12)和b(13)，初始化b的时候因为不是第一次加载，所以staticVarOneInB，staticvarTwoInB不再被初始化，只是初始化了varOneInB(14)，varTwoInB(15)，然后执行构造B(16)。
- 初始化完成后，调用A的构造器(17)；最后通过a.run()调用run(18)方法，打印出“run be called”。

## 具有继承的类的初始化

下面，我创建了一个Father类和一个继承Father类的Son类，来探究在有继承的时候类的初始化和加载，情况基本和上面类似，我就不再写太多的注释了。Father类如下：

```
public class Father {
    private int varInFather = initInt("varInFather");
    private static int staticVarInFather = initInt("staticVarInFather");
    
    public Father(String name) {
        System.out.println("Father constructor" + " name:" + name);
    }
    
    private static int initInt(String varName) {
        System.out.println(varName + " init");
        return 2017;
    }
}
```

Son.java如下：

```
public class Son extends Father{
    private int varInSon = initInt("varInSon");
    private static int staticVarInSon = initInt("staticVarInSon");
    
    public Son(String name) {
        super(name);
        System.out.println("Son constructor" + " name:" + name);
    }
    
    private static int initInt(String varName) {
        System.out.println(varName + " init");
        return 2017;
    }
    
    public static void main(String[] args) {
        System.out.println("start running");
        Son son = new Son("Bob");
    }
}
```

输出结果如下；

```
staticVarInFather init
staticVarInSon init
start running
varInFather init
Father constructor name:Bob
varInSon init
Son constructor name:Bob
```

同样的，我将《Think in java》中的关于继承的类加载和初始化归纳如下：

> **注意：**即使变量定义散布于方法定义之间，它们仍旧会在任何方法（包括构造器）被调用之前得到初始化。

1. （同上）即使没有显示地使用static关键字，构造器实际上也是静态方法。因此，当首次创建类的对象时（构造器可以看出静态方法），或者类的静态方法/静态域被首次访问时，Java解释器必须查找类路径，在对它进行加载的过程中，编译器注意到它有一个基类（有extends得知），于是它继续加载，不管你时候打算产生一个该基类的对象。如果该基类还有其自身的基类，那么第二个基类就会被加载，如此类推。
2. 接下来，根基类的static初始化会被执行，然后是下一个导出类，如此类推。
3. 必要的类加载完成后，对象就可以被创建。同样的，首先对象中所有的基本类型都会被设为默认值，对象引用被设为null——通过将对象内存设为二进制零值而一举生成。
4. 然后基类的构造器会被调用。基类构造器和导出类的构造器一样，以相同的顺序来经历相同的过程。
5. 在基类构造器完成之后，实例变量按其次序被初始化。
6. 最后，构造器的其余部分被执行。

在有了上述归纳后，我们来分析上面程序的结果。

- 在类Son中执行main方法，由于main()是静态方法，必须加载Son类，在加载的时候发现其有父类Father，进而加载Father类。
- Father类被加载的时候，其静态变量staticVarInFather被初始化，之后Son类中的静态变量staticVarInSon被初始化。
- 回到main方法，打印出“start running”
- 在执行new Son("Bob")的时候，对基类也就是Father中的varInFather进行初始化，之后Father的构造器被调用。
- 之后导出类变量varInSon被初始化，调用导出类Son的构造器。

## 具有继承的静态内部类

关于这个的讲解，我引用一道2015携程Java工程师笔试题。来自csdb博客[fuck两点水](https://link.jianshu.com?t=http://my.csdn.net/Two_Water)的[ 2015携程JAVA工程师笔试题(基础却又没多少人做对的面向对象面试题)](https://link.jianshu.com?t=http://blog.csdn.net/two_water/article/details/53891952)。题目如下：

```
public class Base
{
    private String baseName = "base";
    public Base()
    {
        callName();
    }

    public void callName()
    {
        System. out. println(baseName);
    }

    static class Sub extends Base
    {
        private String baseName = "sub";
        public void callName()
        {
            System. out. println (baseName) ;
        }
    }
    public static void main(String[] args)
    {
        Base b = new Sub();
    }
}
```

当时看到这道题的时候，关于类的加载，初始化基本已经忘记，所以直接做错。该题的正确答案是：

```
null
```

为什么是null?首先我们从上面的内容可以了解到，类的初始化顺序是：

> 父类静态块 ->子类静态块 ->父类初始化语句 ->父类构造函器 ->子类初始化语句 子类构造器。

其实在掌握了我上面说的东西后，这道题的的答案为什么为null,已经是“柳暗花明又一村了”；所以我这里直接把[fuck两点水](https://link.jianshu.com?t=http://my.csdn.net/Two_Water)博客上的内容摘抄过来

1. Base b = new Sub();在 main方法中声明父类变量b对子类的引用，JAVA类加载器将Base,Sub类加载到JVM;也就是完成了 Base 类和 Sub 类的初始化
2. JVM 为 Base,Sub 的的成员开辟内存空间且值均为null;在初始化Sub对象前，首先JAVA虚拟机就在堆区开辟内存并将子类 Sub 中的 baseName 和父类 Base 中的 baseName（已被隐藏）均赋为 null，就是子类继承父类的时候，同名的属性不会覆盖父类，只是会将父类的同名属性隐藏
3. 调用父类的无参构造调用 Sub 的构造函数，因为子类没有重写构造函数，默认调用无参的构造函数，调用了 super() 。
4. callName 在子类中被重写，因此调用子类的 callName();调用了父类的构造函数，父类的构造函数中调用了 callName 方法，此时父类中的 baseName 的值为 base，可是子类重写了 callName 方法，且 调用父类 Base 中的 callName 是在子类 Sub 中调用的，因此当前的 this 指向的是子类，也就是说是实现子类的 callName 方法
5. 调用子类的callName，打印baseName

实际上在new Sub()时，实际执行过程为：

```
public Sub(){
    super();
    baseName = "sub"; 
}
```

可见，在 baseName = “sub” 执行前，子类的 callName() 已经执行，所以子类的 baseName 为默认值状态 null 。
   上面的题，大家可以试着把子类中的baseName使用static进行修饰，看看会得到什么结果，加深自己的理解。
   关于类的加载和初始化的备忘录就到此结束了。

## 8、JVM垃圾回收机制，何时触发MinorGC等操作

GC，即就是Java垃圾回收机制。目前主流的JVM（HotSpot）采用的是分代收集算法。与C++不同的是，Java采用的是类似于树形结构的可达性分析法来判断对象是否还存在引用。即：从gcroot开始，把所有可以搜索得到的对象标记为存活对象。

### GC机制

要准确理解Java的垃圾回收机制，就要从：“什么时候”，“对什么东西”，“做了什么”三个方面来具体分析。

第一：“什么时候”即就是GC触发的条件。GC触发的条件有两种。（1）程序调用System.gc时可以触发；（2）系统自身来决定GC触发的时机。

系统判断GC触发的依据：根据Eden区和From Space区的内存大小来决定。当内存大小不足时，则会启动GC线程并停止应用线程。

第二：“对什么东西”笼统的认为是Java对象并没有错。但是准确来讲，GC操作的对象分为：通过可达性分析法无法搜索到的对象和可以搜索到的对象。对于搜索不到的方法进行标记。

第三：“做了什么”最浅显的理解为释放对象。但是从GC的底层机制可以看出，对于可以搜索到的对象进行复制操作，对于搜索不到的对象，调用finalize()方法进行释放。

具体过程：当GC线程启动时，会通过可达性分析法把Eden区和From Space区的存活对象复制到To Space区，然后把Eden Space和From Space区的对象释放掉。当GC轮训扫描To Space区一定次数后，把依然存活的对象复制到老年代，然后释放To Space区的对象。

对于用可达性分析法搜索不到的对象，GC并不一定会回收该对象。要完全回收一个对象，至少需要经过两次标记的过程。

第一次标记：对于一个没有其他引用的对象，筛选该对象是否有必要执行finalize()方法，如果没有执行必要，则意味可直接回收。（筛选依据：是否复写或执行过finalize()方法；因为finalize方法只能被执行一次）。

第二次标记：如果被筛选判定位有必要执行，则会放入FQueue队列，并自动创建一个低优先级的finalize线程来执行释放操作。如果在一个对象释放前被其他对象引用，则该对象会被移除FQueue队列。

### GC过程中用到的回收算法：

通过上面的GC过程不难看出，Java堆中的年轻代和老年代采用了不同的回收算法。年轻代采用了复制法；而老年代采用了标记-整理法

具体各种回收算法的详解参考：http://www.cnblogs.com/dolphin0520/p/3783345.html

### JVM内存空间图解

 ![](https://img-blog.csdn.net/20160917225930028?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

程序计数器：线程私有。是一块较小的内存，是当前线程所执行的字节码的行号指示器。是Java虚拟机规范中唯一没有规定OOM（OutOfMemoryError）的区域。

Java栈：线程私有。生命周期和线程相同。是Java方法执行的内存模型。执行每个方法都会创建一个栈帧，用于存储局部变量和操作数（对象引用）。局部变量所需要的内存空间大小在编译期间完成分配。所以栈帧的大小不会改变。存在两种异常情况：若线程请求深度大于栈的深度，抛StackOverflowError。若栈在动态扩展时无法请求足够内存，抛OOM。

Java堆：所有线程共享。虚拟机启动时创建。存放对象实力和数组。所占内存最大。分为新生代（Young区），老年代（Old区）。新生代分Eden区，Survivor区。Survivor区又分为From space区和To Space区。Eden区和Survivor区的内存比为8:1。 当扩展内存大于可用内存，抛OOM。

方法区：所有线程共享。用于存储已被虚拟机加载的类信息、常量、静态变量等数据。又称为非堆（Non – Heap）。方法区又称“永久代”。GC很少在这个区域进行，但不代表不会回收。这个区域回收目标主要是针对常量池的回收和对类型的卸载。当内存申请大于实际可用内存，抛OOM。

本地方法栈：线程私有。与Java栈类似，但是不是为Java方法（字节码）服务，而是为本地非Java方法服务。也会抛StackOverflowError和OOM。

 

###  Minor GC ，Full GC 触发条件

Minor GC触发条件：当Eden区满时，触发Minor GC。

Full GC触发条件：

（1）调用System.gc时，系统建议执行Full GC，但是不必然执行

（2）老年代空间不足

（3）方法去空间不足

（4）通过Minor GC后进入老年代的平均大小大于老年代的可用内存

（5）由Eden区、From Space区向To Space区复制时，对象大小大于To Space可用内存，则把该对象转存到老年代，且老年代的可用内存小于该对象大小



## 9、JVM 中一次完整的 GC 流程（从 ygc 到 fgc）是怎样的



## 10、各种回收器，各自优缺点，重点CMS、G1

## 11、各种回收算法

## 12、OOM错误，stackoverflow错误，permgen space错误

