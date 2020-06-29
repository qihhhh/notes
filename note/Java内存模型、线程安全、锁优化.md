# 一、Java内存模型

> 目的：保证在Java并发内存访问操作时的一致性

## 1. 主内存与工作内存

<img src="C:\Users\60462\AppData\Roaming\Typora\typora-user-images\image-20200525145049328.png" alt="image-20200525145049328" style="zoom:67%;" />

* 所有的变量都存储在主内存中，每个线程还有自己的工作内存，工作内存存储在高速缓存或者寄存器中，保存了该线程使用的变量的主内存副本拷贝。

* 线程只能直接操作工作内存中的变量。
* 不同线程之间的变量值传递需要通过主内存来完成。

## 2. 线程间的交互操作

> Java虚拟机实现时必须保证下面提及的每一种操作都是**原子的**、不可再分的:

* lock: 作用于主内存的变量，把标量标识为一条线程独占状态
* unlock：作用于主内存的变量，把一个处于锁定状态的变量释放出来

- read：把一个变量的值从主内存传输到工作内存中
- load：在 read 之后执行，把 read 得到的值放入工作内存的变量副本中
- use：把工作内存中一个变量的值传递给执行引擎
- assign：把一个从执行引擎接收到的值赋给工作内存的变量
- store：把工作内存的一个变量的值传送到主内存中
- write：在 store 之后执行，把 store 得到的值放入主内存的变量中

<img src="C:\Users\60462\AppData\Roaming\Typora\typora-user-images\image-20200525150229068.png" alt="image-20200525150229068" style="zoom:150%;" />

注意点：如果要把一个变量从主内存拷贝到工作内存，那就要按顺序执行read和load操作，如果要把变量从工作内存同步回主内存，就要按顺序执行store和write操作。Java内存模型只要求上述两个操作必须按顺序执行，但不要求是连续执行。

## 3. 关于Volatile变量

//首先明确关于线程安全采用同步机制解决时有两种方案：1）锁机制（Synchronized/lock）2）Volatile关键字

> 当一个变量被定义成volatile之后，它将具备两项特性：
>
> 1. 可见性：当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。
> 2. 禁止指令重排序优化：

### 3.1 关于可见性的说明

基于volatile变量的**运算**在并发下不一定是线程安全的！

**解释**：对Volatile修饰变量执行一条java代码，这一条java代码可能产生不止一条的机器指令。只有在这些机器指令全部执行后才能保证可见性，而其他线程可能在这些机器指令执行的过程中对Volatile修饰的变量进行操作从而造成线程不安全的发生。

**实例**：

![image-20200525160236853](C:\Users\60462\AppData\Roaming\Typora\typora-user-images\image-20200525160236853.png)

其对应的反编译代码：

![image-20200525160352430](C:\Users\60462\AppData\Roaming\Typora\typora-user-images\image-20200525160352430.png)

**补充**：可以使用JUC包中的整数原子类（该类方法调用Unsafe包的中CAS方法）解决。

### 3.2 原理分析

实例：

![image-20200525160608790](C:\Users\60462\AppData\Roaming\Typora\typora-user-images\image-20200525160608790.png)

new语句对应的指令：

![image-20200525160650820](C:\Users\60462\AppData\Roaming\Typora\typora-user-images\image-20200525160650820.png)

> 观察汇编指令，一条赋值操作语句的多条汇编指令的最后一条指令为："**lOCK  ADD...**."

关键在于Lock指令的含义：

1. 它的作用是将本处理器的缓存写入了内存
2. 该写入动作也会引起别的处理器或者别的内核无效化（Invalidate）其缓存

这也就意味着：

1. 对其他线程立即可见
2. 形成内存屏障，lock add...指令把修改同步到内存时，意味着所有之前的操作都已经执行完成，这样便形成了“指令重排序无法越过内存屏障”的效果，从而禁止了指令重排序

## 4. 补充：对于64位变量的特殊规则

* lock、unlock、read、load、assign、use、store、write这八种操作都具有原子性
* 但允许虚拟机将**没有被volatile修饰的64位数据**的读写操作划分为两次32位的操作来进行（一些32位机器会这样做），即允许虚拟机实现自行选择是否要保证64位数据类型的load、store、read和write这四个操作的原子性。



## 5. 内存模型的三大特性

### 1. 原子性

实现方式：

1. 小范围：JAVA内存模型的8个交互操作，对基本数据类型操作具备原子性
2. 大范围：Synchronized关键字（实际上使用了lock/unlock操作）

### 2. 可见性

> 可见性就是指当一个线程修改了共享变量的值时，其他线程能够立即得知这个修改。

实现方式：

1. Volatile关键字：保证了新值能立即同步到主内存，以及每次使用前立即从主内存刷新
2. Synchronized关键字：对一个变量执行unlock操作之前，必须先把此变量同步回主内存中（执行store、write操作）
3. final关键字

### 3. 有序性

实现机制：

1. 线程内有序性：Java天然有序性（Within-Thread As-If-Serial Semantics）

2. 线程间有序性：

    1）Volatile关键字：通过添加内存屏障的方式来禁止指令重排，即重排序时不能把后面的指令放到内存屏障之前。

   2）Synchronized关键字：synchronized则是由“一个变量在同一个时刻只允许一条线程对其进行lock操作”这条规则获得的，这个规则决定了持有同一个锁的两个同步块只能串行地进入。

## 6. 先行发生原则

定义：先行发生是Java内存模型中定义的两项操作之间的偏序关系，比如说操作A先行发生于操作B，其实就是说**在发生操作B之前，操作A产生的影响能被操作B观察到**

作用： 判断数据是否存在竞争，线程是否安全的非常有用的手段

* **java内存模型的天然先行发生关系：**

1. 程序次序规则（Program Order Rule）：在一个线程内，按照控制流顺序，书写在前面的操作先行发生于书写在后面的操作。
2. 管程锁定规则（Monitor Lock Rule）：一个unlock操作先行发生于后面对同一个锁的lock操作。这里必须强调的是“同一个锁”，而“后面”是指时间上的先后。

3. volatile变量规则（Volatile Variable Rule）：对一个volatile变量的写操作先行发生于后面对这个变量的读操作，这里的“后面”同样是指时间上的先后。
4. 线程启动规则（Thread Start Rule）：Thread对象的start()方法先行发生于此线程的每一个动作。
5. 线程终止规则（Thread Termination Rule）：线程中的所有操作都先行发生于对此线程的终止检测，我们可以通过Thread::join()方法是否结束、Thread::isAlive()的返回值等手段检测线程是否已经终止执行。
6. 线程中断规则（Thread Interruption Rule）：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过Thread::interrupted()方法检测到是否有中断发生。
7. 对象终结规则（Finalizer Rule）：一个对象的初始化完成（构造函数执行结束）先行发生于它的finalize()方法的开始。
8. 传递性（Transitivity）：如果操作A先行发生于操作B，操作B先行发生于操作C，那就可以得出操作A先行发生于操作C的结论。

# 二、线程安全

## 线程安全的实现方法

### 1. 不可变

不可变（Immutable）的对象一定是线程安全的，不需要再采取任何的线程安全保障措施。只要一个不可变的对象被正确地构建出来，永远也不会看到它在多个线程之中处于不一致的状态。多线程环境下，应当尽量使对象成为不可变，来满足线程安全。

不可变的类型有：

* final 关键字修饰的基本数据类型
* String
* 枚举类型
* Number 部分子类，如 Long 和 Double 等数值包装类型，BigInteger 和 BigDecimal 等大数据类型。但同为 Number 的原子类 AtomicInteger 和 AtomicLong 则是可变的。
* 集合类型使用Collections.unmodifiableXXX()方法获取一个不可变集合

```java
//集合类不可变示例：
public class ImmutableExample {
    public static void main(String[] args) {
        Map<String, Integer> map = new HashMap<>();
        Map<String, Integer> unmodifiableMap = Collections.unmodifiableMap(map);//!
        unmodifiableMap.put("a", 1);
    }
}
```

```
Exception in thread "main" java.lang.UnsupportedOperationException
    at java.util.Collections$UnmodifiableMap.put(Collections.java:1457)
    at ImmutableExample.main(ImmutableExample.java:9)
```

### 2. 互斥同步（阻塞同步）

同步机制：同步是指在多个线程并发访问共享数据时，保证共享数据在同一个时刻只被一条（或者是一些，当使用信号量的时候）线程使用。

* java实现互斥同步的手段

1. Synchronized关键字
2. J.U.C Lock

* 为什么称为阻塞同步：

Java虚拟机实现中，Java的线程是映射到操作系统的原生内核线程之上的，如果要阻塞或唤醒一条线程，则需要操作系统来帮忙完成，这就不可避免地陷入用户态到核心态的转换中，进行这种状态转换需要耗费很多的处理器时间

### 3. 非阻塞同步

> 基于冲突检测的乐观并发策略，通俗地说就是不管风险，先进行操作，如果没有其他线程争用共享数据，那操作就直接成功了；如果共享的数据的确被争用，**产生了冲突**，那再进行其他的补偿措施，最常用的补偿措施是**不断地重试**，直到出现没有竞争的共享数据为止。这种乐观并发策略的实现不再需要把线程阻塞挂起，因此这种同步操作被称为非阻塞同步（Non-Blocking Synchronization）。

#### 3.1 CAS

乐观锁需要操作和冲突检测这两个步骤具备原子性，这里就不能再使用互斥同步来保证了，只能靠硬件来完成。硬件支持的原子性操作最典型的是：比较并交换（Compare-and-Swap，CAS）.

> CAS: CAS 指令需要有 3 个操作数，分别是内存地址 V、旧的预期值 A 和新值 B。当执行操作时，只有当 V 的值等于 A，才将 V 的值更新为 B。

在JDK 5之后，Java类库中才开始使用CAS操作，该操作由sun.misc.Unsafe类里面的compareAndSwapInt()和compareAndSwapLong()等几个方法包装提供。

* 实例：J.U.C包中的整数原子类（如AtomicInteger）调用了Unsafe类中的CAS操作

```java
private AtomicInteger cnt = new AtomicInteger();

public void add() {
    cnt.incrementAndGet();//该函数调用了unsafe的CAS操作
}
```

```java
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
```

以下代码是 getAndAddInt() 源码，var1 指示对象内存地址，var2 指示该字段相对对象内存地址的偏移，var4 指示操作需要加的数值，这里为 1。通过 getIntVolatile(var1, var2) 得到旧的预期值，通过调用 compareAndSwapInt() 来进行 CAS 比较，如果该字段内存地址中的值等于 var5，那么就更新内存地址为 var1+var2 的变量为 var5+var4。

可以看到 getAndAddInt() 在一个循环中进行，发生冲突的做法是不断的进行重试。

```java
//Unsafe类
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```

#### 3.2 ABA问题

如果一个变量初次读取的时候是 A 值，它的值被改成了 B，后来又被改回为 A，那 CAS 操作就会误认为它从来没有被改变过。

J.U.C 包提供了一个带有标记的原子引用类 AtomicStampedReference 来解决这个问题，它可以通过控制变量值的版本来保证 CAS 的正确性。大部分情况下 ABA 问题不会影响程序并发的正确性，如果需要解决 ABA 问题，改用传统的互斥同步可能会比原子类更高效。

### 4. 无同步方案

要保证线程安全，并不是一定就要进行同步。如果一个方法本来就不涉及共享数据，那它自然就无须任何同步措施去保证正确性。

#### 4.1 栈封闭

多个线程访问同一个方法的局部变量时，不会出现线程安全问题，因为局部变量存储在虚拟机栈中，属于线程私有的。

```java
public class StackClosedExample {
    public void add100() {
        int cnt = 0;
        for (int i = 0; i < 100; i++) {
            cnt++;
        }
        System.out.println(cnt);
    }
}

```

```java
public static void main(String[] args) {
    StackClosedExample example = new StackClosedExample();
    ExecutorService executorService = Executors.newCachedThreadPool();
    executorService.execute(() -> example.add100());
    executorService.execute(() -> example.add100());
    executorService.shutdown();
}

```

```java
100
100
```

#### 4.2 线程本地存储（Thread Local Storage

如果一段代码中所需要的数据必须与其他代码共享，那就看看这些共享数据的代码是否能保证在同一个线程中执行。如果能保证，我们就可以**把共享数据的可见范围限制在同一个线程之内**，这样，无须同步也能保证线程之间不出现数据争用的问题。

可以使用 java.lang.ThreadLocal 类来实现线程本地存储功能。

对于以下代码，thread1 中设置 threadLocal 为 1，而 thread2 设置 threadLocal 为 2。过了一段时间之后，thread1 读取 threadLocal 依然是 1，不受 thread2 的影响。

```java
public class ThreadLocalExample {
    public static void main(String[] args) {
        ThreadLocal threadLocal = new ThreadLocal();
        Thread thread1 = new Thread(() -> {
            threadLocal.set(1);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(threadLocal.get());
            threadLocal.remove();
        });
        Thread thread2 = new Thread(() -> {
            threadLocal.set(2);
            threadLocal.remove();
        });
        thread1.start();
        thread2.start();
    }
}Copy to clipboardErrorCopied
1Copy to clipboardErrorCopied
```

为了理解 ThreadLocal，先看以下代码：

```java
public class ThreadLocalExample1 {
    public static void main(String[] args) {
        ThreadLocal threadLocal1 = new ThreadLocal();
        ThreadLocal threadLocal2 = new ThreadLocal();
        Thread thread1 = new Thread(() -> {
            threadLocal1.set(1);
            threadLocal2.set(1);
        });
        Thread thread2 = new Thread(() -> {
            threadLocal1.set(2);
            threadLocal2.set(2);
        });
        thread1.start();
        thread2.start();
    }
}Copy to clipboardErrorCopied
```

它所对应的底层结构图为：

![img](https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/6782674c-1bfe-4879-af39-e9d722a95d39.png)



每个 Thread 都有一个 ThreadLocal.ThreadLocalMap 对象。

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

当调用一个 ThreadLocal 的 set(T value) 方法时，先得到当前线程的 ThreadLocalMap 对象，然后将 ThreadLocal->value 键值对插入到该 Map 中。

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

get() 方法类似。

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

ThreadLocal 从理论上讲并不是用来解决多线程并发问题的，因为根本不存在多线程竞争。

在一些场景 (尤其是使用线程池) 下，由于 ThreadLocal.ThreadLocalMap 的底层数据结构导致 ThreadLocal 有内存泄漏的情况，应该尽可能在每次使用 ThreadLocal 后手动调用 remove()，以避免出现 ThreadLocal 经典的内存泄漏甚至是造成自身业务混乱的风险。

# 三. 锁优化

## 1. 自旋锁

> 互斥同步进入阻塞状态的开销都很大，应该尽量避免。在许多应用中，共享数据的锁定状态只会持续很短的一段时间。自旋锁的思想是让一个线程在请求一个共享数据的锁时**执行忙循环（自旋）**一段时间，如果在这段时间内能获得锁，就可以**避免进入阻塞状态**。

自旋锁虽然能避免进入阻塞状态从而减少开销，但是它需要进行忙循环操作占用 CPU 时间，它只**适用于共享数据的锁定状态很短的场景**。

在 JDK 1.6 中引入了自适应的自旋锁。自适应意味着自旋的次数不再固定了，而是由前一次在同一个锁上的自旋次数及锁的拥有者的状态来决定。

## 2. 锁消除

> 锁消除是指对于被检测出不可能存在竞争的共享数据的锁进行消除。

锁消除主要是通过逃逸分析来支持，如果堆上的共享数据不可能逃逸出去被其它线程访问到，那么就可以把它们当成私有数据对待，也就可以将它们的锁进行消除。

对于一些看起来没有加锁的代码，其实隐式的加了很多锁。例如下面的字符串拼接代码就隐式加了锁：

```java
public String concatString(String s1, String s2, String s3) {
    return s1 + s2 + s3;
}
```

String 是一个不可变的类，编译器会对 String 的拼接自动优化。在 JDK 1.5 之前，会转化为 StringBuffer 对象的连续 append() 操作：

```java
public String concatString(String s1, String s2, String s3) {
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    sb.append(s3);
    return sb.toString();
}
```

每个 append() 方法中都有一个同步块，锁就是sb对象。虚拟机观察变量 sb，很快就会发现它的动态作用域被限制在 concatString() 方法内部。也就是说，sb 的所有引用永远不会逃逸到 concatString() 方法之外，其他线程无法访问到它，因此可以进行消除。

## 3. 锁粗化

如果一系列的连续操作都对**同一个对象反复加锁和解锁**，这样频繁的互斥同步操作（用户态到核心态的转换）会导致不必要的性能损耗。

上一节的示例代码中连续的 append() 方法就属于这类情况。如果虚拟机探测到由这样的一串零碎的操作都对同一个对象sb加锁，将会把加锁的范围扩展（粗化）到整个操作序列的外部。对于上一节的示例代码就是扩展到第一个 append() 操作之前直至最后一个 append() 操作之后，这样只需要加锁一次就可以了。

## 4. 轻量级锁

> JDK 1.6 引入了偏向锁和轻量级锁，从而让锁拥有了四个状态：无锁状态（unlocked）、偏向锁状态（biasble）、轻量级锁状态（lightweight locked）和重量级锁状态（inflated）。

以下是 HotSpot 虚拟机对象头的内存布局，这些数据被称为 **Mark Word**。一共有五个状态：

![image-20200526133826236](C:\Users\60462\AppData\Roaming\Typora\typora-user-images\image-20200526133826236.png)

下图左侧是一个线程的虚拟机栈，其中有一部分称为 Lock Record 的区域，这是在轻量级锁运行过程创建的，用于存放锁对象的 Mark Word。而右侧就是一个锁对象，包含了 Mark Word 和其它信息。

![image-20200526133934016](C:\Users\60462\AppData\Roaming\Typora\typora-user-images\image-20200526133934016.png)

* 综述：轻量级锁是相对于传统的重量级锁而言，它使用 CAS 操作来避免重量级锁使用互斥量的开销。**对于绝大部分的锁，在整个同步周期内都是不存在竞争的，因此也就不需要都使用互斥量进行同步，可以先采用 CAS 操作进行同步，如果 CAS 失败了再改用互斥量进行同步**。
* 轻量级锁工作流程：

1. 在代码即将进入同步块的时候，如果此同步对象没有被锁定（锁标志位为“01”状态），虚拟机首先将**在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝**

2. **使用 CAS 操作将对象的 Mark Word （CAS的地址V）更新为 Lock Record 指针**。如果 CAS 操作成功了，那么线程就获取了该对象上的锁，并且**对象的 Mark Word 的锁标记变为 00**，表示该对象处于轻量级锁状态。



![image-20200526134448827](C:\Users\60462\AppData\Roaming\Typora\typora-user-images\image-20200526134448827.png)

如果这个更新操作失败了，那就意味着至少存在一条线程与当前线程竞争获取该对象的锁。虚拟机首先会检查对象的Mark Word是否指向当前线程的栈帧，如果是，说明当前线程已经拥有了这个对象的锁，那直接进入同步块继续执行就可以了，否则就说明这个锁对象已经被其他线程抢占了。如果出现两条以上的线程争用同一个锁的情况，那轻量级锁就不再有效，必须要膨胀为重量级锁，锁标志的状态值变为“10”。

## 5. 偏向锁

> 目的是消除数据在无竞争情况下的同步原语，进一步提高程序的运行性能

偏向锁的思想是偏向于让第一个获取锁对象的线程，这个线程在之后获取该锁就不再需要进行同步操作，甚至连 CAS 操作也不再需要。

当锁对象第一次被线程获得的时候，进入偏向状态，标记为 1 01。同时使用 CAS 操作将线程 ID 记录到 Mark Word 中，如果 CAS 操作成功，这个线程以后每次进入这个锁相关的同步块就不需要再进行任何同步操作。

当有另外一个线程去尝试获取这个锁对象时，偏向状态就宣告结束，此时撤销偏向（Revoke Bias）后恢复到未锁定状态或者轻量级锁状态。

![image-20200526140411763](C:\Users\60462\AppData\Roaming\Typora\typora-user-images\image-20200526140411763.png)