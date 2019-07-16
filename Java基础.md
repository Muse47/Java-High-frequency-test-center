# Java基础

[TOC]



## 1、List 和 Set 的区别

**List:有序，重复**

**有序**：指按照一定的顺序输出。   
**重复**：指可以向list中添加相同的值
**检索元素效率低下**，删除和插入效率高，插入和删除不会引起元素位置改变。 

**Set:无序，唯一**

**无序**：指输出是没有顺序。  
**唯一**：指不添加可以向set中添加相同的元素，如果你添加相同的元素，最后输出的结果也是唯一的。

**和数组类似**，List可以动态增长，查找元素效率高，插入删除元素效率低，因为会引起其他元素位置改变。

## 2、HashSet 是如何保证不重复的

打开HashSet源码，发现其内部维护了一个HashMap：

    public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
    {
    static final long serialVersionUID = -5024744406713321676L;
    
    private transient HashMap<E,Object> map;
    
    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();
    
    /**
     * Constructs a new, empty set; the backing <tt>HashMap</tt> instance has
     * default initial capacity (16) and load factor (0.75).
     */
    public HashSet() {
    map = new HashMap<>();
    }
    ...
    }

HashSet的构造方法其实就是在内部实例化了一个HashMap对象。其中还会看到一个static final的PRESENT变量，这个稍候再说，其实没什么实际用处。

想知道为什么HashSet不能存放重复对象，那么第一步当然是看它的add方法怎么进行的判重，代码如下：


    public boolean add(E e) {
    return map.put(e, PRESENT)==null;
    }

好吧，就把元素存放在了map里面。但是值得注意的是元素值作为的是map的key，map的value则是前面提到的PRESENT变量，这个变量只作为放入map时的一个占位符而存在，所以没什么实际用处。

其实，这时候答案已经出来了：**HashMap的key是不能重复的，而这里HashSet的元素又是作为了map的key，当然也不能重复了。**

HashSet怎么做到保证元素不重复的原因找到了，文章也就结束了。。。等等，顺便看一下HashMap里面又是怎么保证key不重复的吧，代码如下：


    public V put(K key, V value) {
    if (table == EMPTY_TABLE) {
    inflateTable(threshold);
    }
    if (key == null)
    return putForNullKey(value);
    int hash = hash(key);
    int i = indexFor(hash, table.length);
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
    Object k;
    if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
    V oldValue = e.value;
    e.value = value;
    e.recordAccess(this);
    return oldValue;
    }
    }
    
    modCount++;
    addEntry(hash, key, value, i);
    return null;
    }

其中最关键的一句：


    if (e.hash == hash && ((k = e.key) == key || key.equals(k)))

调用了对象的hashCode和equals方法进行的判断，所以又得出一个结论：若要将对象存放到HashSet中并保证对象不重复，应根据实际情况将对象的hashCode方法和equals方法进行重写

## 3、HashMap 是线程安全的吗，为什么不是线程安全的（最好画图说明多线程环境下不安全）?4、HashMap 的扩容过程

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



## 5、HashMap 1.7 与 1.8 的 区别，说明 1.8 做了哪些优化，如何优化的？

不同点：

（1）JDK1.7用的是头插法，而JDK1.8及之后使用的都是尾插法，那么他们为什么要这样做呢？因为JDK1.7是用单链表进行的纵向延伸，当采用头插法时会容易出现逆序且环形链表死循环问题。但是在JDK1.8之后是因为加入了红黑树使用尾插法，能够避免出现逆序且链表死循环的问题。

（2）扩容后数据存储位置的计算方式也不一样：1. 在JDK1.7的时候是直接用hash值和需要扩容的二进制数进行&（这里就是为什么扩容的时候为啥一定必须是2的多少次幂的原因所在，因为如果只有2的n次幂的情况时最后一位二进制数才一定是1，这样能最大程度减少hash碰撞）（hash值 & length-1）

2、而在JDK1.8的时候直接用了JDK1.7的时候计算的规律，也就是扩容前的原始位置+扩容的大小值=JDK1.8的计算方式，而不再是JDK1.7的那种异或的方法。但是这种方式就相当于只需要判断Hash值的新增参与运算的位是0还是1就直接迅速计算出了扩容后的储存方式。

## 6、final finally finalize

一、final ：

1、修饰符（关键字） 如果一个类被声明为final,意味着它不能再派生新的子类，不能作为父类被继承。因此一个类不能及被声明为abstract，又被声明为final的。

2、将变量或方法声明为final,可以保证他们使用中不被改变。被声明为final的变量必须在声明时给定初值，而以后的引用中只能读取，不可修改，被声明为final的方法也同样只能使用，不能重载。

二、finally:

在异常处理时提供finally块来执行清楚操作。如果抛出一个异常，那么相匹配的catch语句就会执行，然后控制就会进入finally块，如果有的话。

三、finalize：

是方法名。java技术允许使用finalize()方法在垃圾收集器将对象从内存中清除之前做必要的清理工作。这个方法是在垃圾收集器在确定了，被清理对象没有被引用的情况下调用的。

finalize是在Object类中定义的，因此，所有的类都继承了它。子类可以覆盖finalize()方法，来整理系统资源或者执行其他清理工作。

## 7、强引用 、软引用、 弱引用、虚引用

从 JDK1.2 版本开始，Java 把对象的引用分为四种级别，从而使程序能更加灵活的控制对象的生命周期。这四种级别由高到低依次为：强引用、软引用、弱引用和虚引用。

### 1、强引用（Strong Reference）

强引用就是我们经常使用的引用，其写法如下：

    Object o = new Object();


只要还有强引用指向一个对象，垃圾收集器就不会回收这个对象；显式地设置 o 为 null，或者超出对象的生命周期，此时就可以回收这个对象。具体回收时机还是要看垃圾收集策略。

在不用对象的时将引用赋值为 null，能够帮助垃圾回收器回收对象。比如 ArrayList 的 clear() 方法实现：

     public void clear() {
     modCount++;
    
     // clear to let GC do its work
     for (int i = 0; i < size; i++)
     elementData[i] = null;
     size = 0;
     }

### 2、软引用（Soft Reference）

如果一个对象只具有软引用，在内存足够时，垃圾回收器不会回收它；如果内存不足，就会回收这个对象的内存。

使用场景：

图片缓存。图片缓存框架中，“内存缓存”中的图片是以这种引用保存，使得 JVM 在发生 OOM 之前，可以回收这部分缓存。此外，还可以用在网页缓存上。

    Browser prev = new Browser();   // 获取页面进行浏览
    SoftReference sr = new SoftReference(prev); // 浏览完毕后置为软引用
    if(sr.get()!=null) { 
    rev = (Browser) sr.get();   // 还没有被回收器回收，直接获取
    } else {
    prev = new Browser();   // 由于内存吃紧，所以对软引用的对象回收了
    sr = new SoftReference(prev);   // 重新构建
    }


​    

### 3、弱引用（Weak Reference）

简单来说，就是将对象留在内存的能力不是那么强的引用。当垃圾回收器扫描到只具有弱引用的对象，不管当前内存空间是否足够，都会回收内存。

使用场景：

在下面的代码中，如果类 B 不是虚引用类 A 的话，执行 main 方法会出现内存泄漏的问题， 因为类 B 依然依赖于 A。

    public class Main {
    public static void main(String[] args) {
    
    A a = new A();
    B b = new B(a);
    a = null;
    System.gc();
    System.out.println(b.getA());  // null
    
    }
    
    }
    
    class A {}
    
    class B {
    
    WeakReference<A> weakReference;
    
    public B(A a) {
    weakReference = new WeakReference<>(a);
    }
    
    public A getA() {
    return weakReference.get();
    }
    }

   




在静态内部类中，经常会使用虚引用。例如：一个类发送网络请求，承担 callback 的静态内部类，则常以虚引用的方式来保存外部类的引用，当外部类需要被 JVM 回收时，不会因为网络请求没有及时回应，引起内存泄漏。

### 4、虚引用（Phantom Reference）

虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收器回收。

虚引用与软引用和弱引用的一个区别在于：虚引用必须和引用队列（ReferenceQueue）联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。
    
    Object obj = new Object();
    ReferenceQueue refQueue = new ReferenceQueue();
    PhantomReference<Object> phantomReference = new PhantomReference<Object>(obj,refQueue);


使用场景：

可以用来跟踪对象呗垃圾回收的活动。一般可以通过虚引用达到回收一些非java内的一些资源比如堆外内存的行为。例如：在 DirectByteBuffer 中，会创建一个 PhantomReference 的子类Cleaner的虚引用实例用来引用该 DirectByteBuffer 实例，Cleaner 创建时会添加一个 Runnable 实例，当被引用的 DirectByteBuffer 对象不可达被垃圾回收时，将会执行 Cleaner 实例内部的 Runnable 实例的 run 方法，用来回收堆外资源。

## 8、Java反射

### 1、java反射机制的作用

 1)在运行时判断任意一个对象所属的类

 2)在运行时构造任意一个类的对象

 3)在运行时判断任意一个类所具有的成员变量和方法

 4)在运行时调用任意一个对象的方法
> 
>    反射就是动态加载对象，并对对象进行剖析。在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法，这种动态获取信息以及动态调用对象方法的功能成为Java反射机制。

优点：可以动态的创建对象和编译，最大限度发挥了java的灵活性。

缺点：对性能有影响。使用反射基本上一种解释操作，告诉JVM我们要做什么并且满足我们的要求，这类操作总是慢于直接执行java代码。

### 2. 如何使用java的反射?

#### a. 通过一个全限类名创建一个对象

1)、Class.forName("全限类名"); 例如：com.mysql.jdbc.Driver Driver类已经被加载到 jvm中，并且完成了类的初始化工作就行了

2）、类名.class; 获取Class<？> clz 对象  
   3）、对象.getClass();   

#### b. 获取构造器对象，通过构造器new出一个对象

​    1）. Clazz.getConstructor([String.class]);
​    2）. Con.newInstance([参数]);

#### c. 通过class对象创建一个实例对象（就相当与new类名（）无参构造器)

     1）. Clazz.newInstance();

#### d. 通过class对象获得一个属性对象

1）、Field c=clz.getFields()：获得某个类的所有的公共（public）的字段，包括父类中的字段。

2)、Field c=clz.getDeclaredFields()：获得某个类的所有声明的字段，即包括public、private和proteced，但是不包括父类的申明字段 
####e.通过class对象获得一个方法对象

   1）. Clazz.getMethod("方法名",class…..parameaType);（只能获取公共的）

   2）. Clazz.getDeclareMethod("方法名");（获取任意修饰的方法，不能执行私有）

   3) M.setAccessible(true);（让私有的方法可以执行）

#### f. 让方法执行

   1）. Method.invoke(obj实例对象,obj可变参数);-----（是有返回值的）

## 9、Arrays.sort 实现原理和 Collection 实现原理

事实上Collections.sort方法底层就是调用的array.sort方法，

    public static void sort(Object[] a) {
    if (LegacyMergeSort.userRequested)
    legacyMergeSort(a);
    else
    ComparableTimSort.sort(a, 0, a.length, null, 0, 0);
    }
    //void java.util.ComparableTimSort.sort()
    static void sort(Object[] a, int lo, int hi, Object[] work, int workBase, int workLen) {
    assert a != null && lo >= 0 && lo <= hi && hi <= a.length;
    
    int nRemaining  = hi - lo;
    if (nRemaining < 2)
    return;  // Arrays of size 0 and 1 are always sorted
    
    // If array is small, do a "mini-TimSort" with no merges
    if (nRemaining < MIN_MERGE) {
    int initRunLen = countRunAndMakeAscending(a, lo, hi);
    binarySort(a, lo, hi, lo + initRunLen);
    return;
    }

legacyMergeSort(a)：归并排序

ComparableTimSort.sort()：Timsort排序

备注：
Timsort排序是结合了合并排序（merge sort）和插入排序（insertion sort）而得出的排序算法。（参考：Timsort原理介绍）
Timsort的核心过程

> TimSort 算法为了减少对升序部分的回溯和对降序部分的性能倒退，将输入按其升序和降序特点进行了分区。排序的输入的单位不是一个个单独的数字，而是一个个的块-分区。其中每一个分区叫一个run。针对这些 run 序列，每次拿一个 run 出来按规则进行合并。每次合并会将两个 run合并成一个 run。合并的结果保存到栈中。合并直到消耗掉所有的 run，这时将栈上剩余的 run合并到只剩一个 run 为止。这时这个仅剩的 run 便是排好序的结果。

综上述过程，Timsort算法的过程包括

（0）如何数组长度小于某个值，直接用二分插入排序算法

（1）找到各个run，并入栈

（2）按规则合并run

### 总结

　　不论是Collections.sort方法或者是Arrays.sort方法，底层实现都是TimSort实现的，这是jdk1.7新增的，以前是归并排序。TimSort算法就是找到已经排好序数据的子序列，然后对剩余部分排序，然后合并起来

## 10、LinkedHashMap的应用


**LinkedHashMap是HashMap的子类，但是内部还有一个双向链表维护键值对的顺序，每个键值对既位于哈希表中，也位于双向链表中。LinkedHashMap支持两种顺序插入顺序 、 访问顺序**

> 插入顺序：先添加的在前面，后添加的在后面。修改操作不影响顺序
> 访问顺序：所谓访问指的是get/put操作，对一个键执行get/put操作后，其对应的键值对会移动到链表末尾，所以最末尾的是最近访问的，最开始的是最久没有被访问的，这就是访问顺序。

### 1.1 指定按照访问顺序排序

LinkedHashMap有5个构造方法，其中4个都是按插入顺序，只有一个是可以指定按访问顺序：

    public LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder)

其中参数accessOrder就是用来指定是否按访问顺序，如果为true，就是访问顺序。

### 1.2 栗子

默认情况下，LinkedHashMap是按照插入顺序的，我们举个栗子:

    Map<String, Integer> seqMap = new LinkedHashMap<>();
    seqMap.put("c",100);
    seqMap.put("d",200);
    seqMap.put("a",500);
    seqMap.put("d",300);
    for(Entry<String,Integer> entry:seqMap.entrySet()){
    	System.out.println(entry.getKey()+" "+entry.getValue());
    }



键是按照:“c”, “d”,"a"的顺序插入的，修改d不会修改顺序，输出为：

    c 100
    d 300
    a 500



按访问顺序的栗子：

    Map<String, Integer> accessMap = new LinkedHashMap<>(16,0.75f,true);
    accessMap.put("c",100);
    accessMap.put("d",200);
    accessMap.put("a",500);
    accessMap.get("c");
    accessMap.put("d",300);
    for(Map.Entry<String,Integer> entry:accessMap.entrySet()){
    System.out.println(entry.getKey()+" "+entry.getValue());
    }



输出为：

    a 500
    c 100
    d 300

### 1.3 使用按访问有序实现缓存

    import java.util.LinkedHashMap;
    import java.util.Map;
    
    public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    
    private int maxEntries;
    
    public LRUCache(int maxEntries) {
    super(16, 0.75f, true);
    this.maxEntries = maxEntries;
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
    return size() > maxEntries;
    }
    
    }



在LinkedHashMap添加元素后，会调用removeEldestEntry防范，传递的参数时最久没有被访问的键值对，如果方法返回true，这个最久的键值对就会被删除。LinkedHashMap中的实现总返回false，该子类重写后即可实现对容量的控制

使用该缓存：
    
    LRUCache<String,Object> cache = new LRUCache<>(3);
    cache.put("a","abstract");
    cache.put("b","basic");
    cache.put("c","call");
    cache.get("a");
    cache.put("d","滴滴滴");
    System.out.println(cache); // 输出为：{c=call, a=abstract, d=滴滴滴}

## 11、cloneable接口实现原理



### java的克隆分为深克隆和浅克隆：

了解克隆clone我们必须要了解

1.首先，要了解什么是克隆，怎么实现克隆。
2.其次，你要大概知道什么是地址传递，什么是值传递。
3.最后，你要知道你为什么使用这个clone方法

### Cloneable接口的作用：

> Object类中的clone()是protected的方法，在没有重写Object的clone()方法且没有实现Cloneable接口的实例上调用clone方法，会报CloneNotSupportedException异常。
> 
> 实现Cloneable接口仅仅是用来指示Object类中的clone()方法可以用来合法的进行克隆，即实现了Cloneable接口在调用Object的clone方法时不会再报CloneNotSupportedException异常。

### clone方法：

> 类在重写clone方法的时候，要把clone方法的属性设置为public。

#### 为什么需要克隆？

> 在实际编程过程中，会需要创建与已经存在的对象A的值一样的对象B，但是是与A完全独立的一个对象，即对两个对象做修改互不影响，这时需要用克隆来创建对象B。通过new一个对象，然后各个属性赋值，也能实现该需求，但是clone方法是native方法，native方法的效率一般远高于非native方法。

#### 怎么实现克隆？

> 
> 在要克隆的类要实现Cloneable接口和重写Object的clone()方法。

#### 浅克隆

> 对于Object类中的clone()方法产生的效果是：现在内存中开辟一块和原始对象一样的内存空间，然后原样拷贝原始对象中的内容。对基本数据类型来说，这样的操作不会有问题，但是对于非基本类型的变量，保存的仅仅是对象的引用，导致clone后的非基本类型变量和原始对象中相应的变量指向的是同一个对象，对非基本类型的变量的操作会相互影响。

#### 深克隆

> 
> 深克隆除了克隆自身对象，还对其非基本数据类型的成员变量克隆一遍。

深克隆的步骤：

1、首先克隆的类要实现Cloneable接口和重写Object的clone()方法。

2、在不引入第三方jar包的情况下，可以使用两种方法：1）先对对象进行序列化，紧接着马上反序列化 2）先调用super.clone()方法克隆出一个新对象，然后手动给克隆出来的对象的非基本数据类型的成员变量赋值。

> 在数据结构比较复杂的情况下，序列化和反序列化可能实现起来简单，方法2）实现比较复杂。经测试2）会比1）的执行效率高。

### 结论：

1、克隆一个对象不会调用对象的构造方法。

2、clone()方法有三条规则：1）x.clone() != x; 2)x.clone().getClass() == x.getClass(); 3)一般情况下x.clone().equals(x); 3）不是必须要满足的。

3、对对象基本数据类型的修改不会互相影响，浅克隆对对象非基本数据类型的修改会相互影响，所以需要实现深克隆。

## 12、异常分类以及处理机制

常见的异常：

异常的分级分类：

![](https://images2017.cnblogs.com/blog/1056713/201801/1056713-20180129210257453-672411972.png)
> 
>  NET调用过程中产生的异常，对于终端用户来说这些异常不应该出现，应该在系统测试阶段就解决。
>   数据库访问异常：给只读的属性赋值 ，属性的值与类型不匹配， 在实体的属性值设为NULL之后，仍然去访问它的属性性 等。
>  应用系统内部异常：是应用系统自己定义的异常，对于终端用户来说这些异常不应该出现，主要用来支持内部系统的决策，比如是否可恢复等，通常是将.NET异常转化而来。
>  应用系统业务逻辑异常：是应用系统自己定义，用来向终端用户报告业务逻辑错误或将系统内部异常转化为终端用户可以理解的错误消息，终端用户应该只能看到此类异常。

异常的处理策略

>  对于异常来说一个明确的处理策略就是只处理或封装你明确知道的异常，通过测试来暴露所有可能出错的异常，其中有的异常会被作为bug修改，有的异常会因为无法避免而暴露出来成为我们明确知道的异常，这时我们需要决策这类异常该如何处理（忽略，尝试，报告用户）

使用


应用程序异常的处理方式：

使用try-catch-finally块捕获异常，基本格式如下：
复制代码

    try
    {
    //获取并使用资源，可能出现异常
    }
    catch(Exception e)
    {
      //是否抛出异常 
    }
    finally
    {
    //无论什么情况(即使在catch块中return)下，都会执行该块的代码(如：关闭文件)
    //另外需要说明的是，在finally块中使用任何break、continue、return退出都是非法的。
    }



## 13、wait和sleep的区别

Sleep()

> Sleep是线程类Thread 的方法，它是使当前线程暂时睡眠，可以放在任何位置。Sleep使用的时候，线程并不会放弃对象的使用权，所以在同步方法或同步块中使用sleep，一个线程访问时，其他的线程无法访问。sleep只是暂时休眠一定时间，时间到了之后，自动恢复运行，不需另外的线程唤醒.

Wait()

> wait是使当前线程暂时放弃对象的使用权进行等待，必须放在同步方法或同步块里。wait会释放当前线程,放弃对象的使用权，让其他的线程也可以访问。线程执行wait方法时，需要其他线程调用Monitor.Pulse()或者Monitor.PulseAll() 通过monitor监视器进行唤醒或者是通知等待的队列。

 

所以sleep()和wait()方法的最大区别是：

sleep()睡眠时，保持对象锁，仍然占有该锁；其他线程无法访问
　

而wait()睡眠时，释放对象锁。其他线程可以访问

## 14、数组在内存中如何分配

当一个对象使用new关键字创建的时候，会在堆上分配内存空间，然后才返回到对象的引用。这对数组来说也是一样的，因为数组也是一个对象，简单的值类型的数组，每个数组成员是一个引用（指针），引用到栈上的空间。 （详细百度） 













