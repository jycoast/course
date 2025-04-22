# 可见性、有序性、原子性

## 可见性

在单核时代，所有的线程都是在一颗 CPU 上执行，CPU 缓存与内存的数据一致性容易解决。因为所有线程都是操作同一个 CPU 的缓存，一个线程对缓存的写，对另外一个线程来说一定是可见的。例如在下面的图中，线程 A 和线程 B 都是操作同一个 CPU 里面的缓存，所以线程 A 更新了变量 V 的值，那么线程 B 之后再访问变量 V，得到的一定是 V 的最新值（线程 A 写过的值）。

<img src="https://blog-1304855543.cos.ap-guangzhou.myqcloud.com/blog/image-20250422113415754.png" alt="image-20250422113415754" style="zoom: 67%;" />

一个线程对共享变量的修改，另外一个线程能够立刻看到，我们称为**可见性**。

多核时代，每颗 CPU 都有自己的缓存，这时 CPU 缓存与内存的数据一致性就没那么容易解决了，当多个线程在不同的 CPU 上执行时，这些线程操作的是不同的 CPU 缓存。比如下图中，线程 A 操作的是 CPU-1 上的缓存，而线程 B 操作的是 CPU-2 上的缓存，很明显，这个时候线程 A 对变量 V 的操作对于线程 B 而言就不具备可见性了。这个就属于硬件程序员给软件程序员挖的“坑”。

<img src="https://blog-1304855543.cos.ap-guangzhou.myqcloud.com/blog/image-20250422113457989.png" alt="image-20250422113457989" style="zoom:67%;" />

可见性的例子：

```java
public class VisibilityTest {

    private boolean flag = true;

    private int count = 0;

    public void refresh() {
        flag = false;
        System.out.println(Thread.currentThread().getName() + "修改flag为false");
    }

    public void load() {
        System.out.println(Thread.currentThread().getName() + "开始执行....");
        while (flag) {
            count++;
        }
        System.out.println(Thread.currentThread().getName() + "跳出循环：count=" + count);
    }

    public static void main(String[] args) throws InterruptedException {
        VisibilityTest test = new VisibilityTest();

        Thread threadA = new Thread(() -> test.load(), "threadA");
        threadA.start();

        Thread.sleep(1000);

        Thread threadB = new Thread(() -> test.refresh(), "threadB");
        threadB.start();
    }
}
```

## 原子性

由于 IO 太慢，早期的操作系统就发明了多进程，即便在单核的 CPU 上我们也可以一边听着歌，一边写 Bug，这个就是多进程的功劳。

操作系统允许某个进程执行一小段时间，例如 50 毫秒，过了 50 毫秒操作系统就会重新选择一个进程来执行（我们称为“任务切换”），这个 50 毫秒称为“**时间片**”。

<img src="https://blog-1304855543.cos.ap-guangzhou.myqcloud.com/blog/image-20250422113553926.png" alt="image-20250422113553926" style="zoom:67%;" />

在一个时间片内，如果一个进程进行一个 IO 操作，例如读个文件，这个时候该进程可以把自己标记为“休眠状态”并出让 CPU 的使用权，待文件读进内存，操作系统会把这个休眠的进程唤醒，唤醒后的进程就有机会重新获得 CPU 的使用权了。

这里的进程在等待 IO 时之所以会释放 CPU 使用权，是为了让 CPU 在这段等待时间里可以做别的事情，这样一来 CPU 的使用率就上来了；此外，如果这时有另外一个进程也读文件，读文件的操作就会排队，磁盘驱动在完成一个进程的读操作后，发现有排队的任务，就会立即启动下一个读操作，这样 IO 的使用率也上来了。

Java 并发程序都是基于多线程的，自然也会涉及到任务切换，任务切换的时机大多数是在时间片结束的时候，我们现在基本都使用高级语言编程，高级语言里一条语句往往需要多条 CPU 指令完成，例如上面代码中的count += 1，至少需要三条 CPU 指令。

指令 1：首先，需要把变量 count 从内存加载到 CPU 的寄存器；

指令 2：之后，在寄存器中执行 +1 操作；

指令 3：最后，将结果写入内存（缓存机制导致可能写入的是 CPU 缓存而不是内存）。

操作系统做任务切换，可以发生在任何一条 **CPU 指令**执行完，是的，是 CPU 指令，而不是高级语言里的一条语句。对于上面的三条指令来说，我们假设 count=0，如果线程 A 在指令 1 执行完后做线程切换，线程 A 和线程 B 按照下图的序列执行，那么我们会发现两个线程都执行了 count+=1 的操作，但是得到的结果不是我们期望的 2，而是 1。

<img src="https://blog-1304855543.cos.ap-guangzhou.myqcloud.com/blog/image-20250422113709088.png" alt="image-20250422113709088" style="zoom:67%;" />

我们潜意识里面觉得 count+=1 这个操作是一个不可分割的整体，就像一个原子一样，线程的切换可以发生在 count+=1 之前，也可以发生在 count+=1 之后，但就是不会发生在中间。**我们把一个或者多个操作在 CPU 执行的过程中不被中断的特性称为原子性**。CPU 能保证的原子操作是 CPU 指令级别的，而不是高级语言的操作符，这是违背我们直觉的地方。因此，很多时候我们需要在高级语言层面保证操作的原子性。

## 有序性

有序性指的是程序按照代码的先后顺序执行。编译器为了优化性能，有时候会改变程序中语句的先后顺序，例如程序中：“a=6；b=7；”编译器优化后可能变成“b=7；a=6；”，在这个例子中，编译器调整了语句的顺序，但是不影响程序的最终结果。不过有时候编译器及解释器的优化可能导致意想不到的 Bug。

```java
public class ReOrderTest {


    private static int x = 0, y = 0;
    private static int a = 0, b = 0;

    public static void main(String[] args) throws InterruptedException {

        int i = 0;

        while (true) {
            i++;
            x = 0;
            y = 0;
            a = 0;
            b = 0;

            Thread threadA = new Thread(() -> {
                shortWait(20000);
                a = 1;
                UnsafeFactory.getUnsafe().storeFence(); // 内存屏障
                x = b;
            });

            Thread threadB = new Thread(() -> {
                b = 1;
                y = a;
            });


            threadA.start();
            threadB.start();

            threadA.join();
            threadB.join();

            System.out.println("第" + i + "次（" + x + "," + y + ")");

            if (x == 0 && y == 0) {
                break;
            }
        }
    }

    public static void shortWait(long interval) {
        long start = System.nanoTime();
        long end;
        do {
            end = System.nanoTime();
        } while (start + interval >= end);
    }
}
```

### 为什么会出现指令重排序？

计算机在执行程序时，为了提高性能，编译器和处理器常常会对指令做重排。

**为什么指令重排序可以提高性能？**

简单地说，每一个指令都会包含多个步骤，每个步骤可能使用不同的硬件。因此，**流水线技术**产生了，它的原理是指令1还没有执行完，就可以开始执行指令2，而不用等到指令1执行结束之后再执行指令2，这样就大大提高了效率。

但是，流水线技术最害怕**中断**，恢复中断的代价是比较大的，所以我们要想尽办法不让流水线中断。指令重排就是减少中断的一种技术。

```java
a = b + c;
d = e - f ;
```

先加载b、c（**注意，即有可能先加载b，也有可能先加载c**），但是在执行add(b,c)的时候，需要等待b、c装载结束才能继续执行，也就是增加了停顿，那么后面的指令也会依次有停顿,这降低了计算机的执行效率。

为了减少这个停顿，我们可以先加载e和f,然后再去加载add(b,c),这样做对程序（串行）是没有影响的,但却减少了停顿。既然add(b,c)需要停顿，那还不如去做一些有意义的事情。

综上所述，**指令重排对于提高CPU处理性能十分必要。虽然由此带来了乱序的问题，但是这点牺牲是值得的。**

指令重排一般分为以下三种：

- **编译器优化重排**

  编译器在**不改变单线程程序语义**的前提下，可以重新安排语句的执行顺序。

- **指令并行重排**

  现代处理器采用了指令级并行技术来将多条指令重叠执行。如果**不存在数据依赖性**(即后一个执行的语句无需依赖前面执行的语句的结果)，处理器可以改变语句对应的机器指令的执行顺序。

- **内存系统重排**

  由于处理器使用缓存和读写缓存冲区，这使得加载(load)和存储(store)操作看上去可能是在乱序执行，因为三级缓存的存在，导致内存与缓存的数据同步存在时间差。

**指令重排可以保证串行语义一致，但是没有义务保证多线程间的语义也一致**。所以在多线程下，指令重排序可能会导致一些问题。

# 什么是JMM？

## JMM的由来

一般来说，编程语言也可以直接复用操作系统层面的内存模型。不过，不同的操作系统内存模型不同。如果直接复用操作系统层面的内存模型，就可能会导致同样一套代码换了一个操作系统就无法执行了。但Java 语言是跨平台的，它需要自己提供一套内存模型以屏蔽系统差异。

这只是 JMM 存在的其中一个原因。实际上，对于 Java 来说，你可以把 JMM 看作是 Java 定义的并发编程相关的一组规范（[JSR133](http://ifeve.com/wp-content/uploads/2014/03/JSR133%E4%B8%AD%E6%96%87%E7%89%88.pdf)），除了抽象了线程和主内存之间的关系之外，其还规定了从 Java 源代码到 CPU 可执行指令的这个转化过程要遵守哪些和并发相关的原则和规范，其主要目的是为了简化多线程编程，增强程序可移植性的。

Java线程之间的通信由Java内存模型（简称JMM）控制，从抽象的角度来说，JMM定义了线程和主内存之间的抽象关系。JMM的抽象示意图如图所示：

<img src="https://blog-1304855543.cos.ap-guangzhou.myqcloud.com/blog/image-20250421233422584.png" alt="image-20250421233422584" style="zoom:50%;" />

**什么是主内存？什么是本地内存？**

- **主内存**：所有线程创建的实例对象都存放在主内存中，不管该实例对象是成员变量，还是局部变量，类信息、常量、静态变量都是放在主内存中。为了获取更好的运行速度，虚拟机及硬件系统可能会让工作内存优先存储于寄存器和高速缓存中。
- **本地内存**：每个线程都有一个私有的本地内存，本地内存存储了该线程以读 / 写共享变量的副本。每个线程只能操作自己本地内存中的变量，无法直接访问其他线程的本地内存。如果线程间需要通信，必须通过主内存来进行。本地内存是 JMM 抽象出来的一个概念，并不真实存在，它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器优化。

从图中可以看出： 

1. 所有的共享变量都存在主内存中。 
2. 每个线程都保存了一份该线程使用到的共享变量的副本。 

如果线程A与线程B之间要通信的话，必须经历下面2个步骤：

1. 线程A将本地内存A中更新过的共享变量刷新到主内存中去。 
2. 线程B到主内存中去读取线程A之前已经更新过的共享变量。

**所以，线程A无法直接访问线程B的工作内存，线程间通信必须经过主内存。**

所以线程B并不是直接去主内存中读取共享变量的值，而是先在本地内存B中找到这个共享变量，发现这个共享变量已经被更新了，然后本地内存B去主内存中读取这个共享变量的新值，并拷贝到本地内存B中，最后线程B再读取本地内存B中的新值。

那么怎么知道这个共享变量的被其他线程更新了呢？这就是JMM的功劳了，也是JMM存在的必要性之一。**JMM通过控制主内存与每个线程的本地内存之间的交互，来提供内存可见性保证**。

Java 内存模型规范了 JVM 如何提供按需禁用缓存和编译优化的方法。具体来说，这些方法包括 **volatile**、**synchronized** 和 **final** 三个关键字，以及六项 **Happens-Before 规则**。

## 关于JMM

**为什么这么设计？**

1. 减少CPU等待时间
2. 空间局部性原理、时间局部性

**别的编程语言也有内存模型吗？**

只要支持并发的现代编程语言都有类似的概念。

# 按需禁用缓存和编译优化

## happen before

Happens-Before 的含义：**前面一个操作的结果对后续操作是可见的**。

Happens-Before 的作用：用于程序员判断程序是否有线程安全问题，另外Happens-Before 约束了编译器的优化行为，虽允许编译器优化，但是要求编译器优化后一定遵守 Happens-Before 规则。

Happens-Before 的语义是一种因果关系。在现实世界里，如果 A 事件是导致 B 事件的起因，那么 A 事件一定是先于（Happens-Before）B 事件发生的，这个就是 Happens-Before 语义的现实理解。

在 Java 语言里面，Happens-Before 的语义本质上是一种可见性，A Happens-Before B 意味着 A 事件对 B 事件来说是可见的，无论 A 事件和 B 事件是否发生在同一个线程里。例如 A 事件发生在线程 1 上，B 事件发生在线程 2 上，Happens-Before 规则保证线程 2 上也能看到 A 事件的发生。

happens-before的内容：

- 程序顺序规则：一个线程中的每一个操作，happens-before于该线程中的任意后续操作。
- volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。
- 传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。
- 管程中锁的规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。
- start规则：如果线程A执行操作ThreadB.start()启动线程B，那么A线程的ThreadB.start（）操作happens-before于线程B中的任意操作
- join规则：如果线程A执行操作ThreadB.join（）并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。

### 1. 程序的顺序性规则

这条规则是指在一个线程中，按照程序顺序，前面的操作 Happens-Before 于后续的任意操作。

这还是比较容易理解的，在如下代码中：

```java
class VolatileExample {
  int x = 0;
  volatile boolean v = false;
  public void writer() {
    x = 42;
    v = true;
  }
  public void reader() {
    if (v == true) {
      // 这里x会是多少呢？
    }
  }
}
```

按照程序的顺序，第 6 行代码 “x = 42;” Happens-Before 于第 7 行代码 “v = true;”，这就是规则 1 的内容，也比较符合单线程里面的思维：程序前面对某个变量的修改一定是对后续操作可见的。

```java
class VolatileExample {
  int x = 0;
  volatile boolean v = false;
  public void writer() {
    x = 42;
    v = true;
  }
  public void reader() {
    if (v == true) {
      // 这里x会是多少呢？
    }
  }
}
```

### 2. volatile 变量规则

这条规则是指对一个 volatile 变量的写操作， Happens-Before 于后续对这个 volatile 变量的读操作。

这个就有点费解了，对一个 volatile 变量的写操作相对于后续对这个 volatile 变量的读操作可见，这怎么看都是禁用缓存的意思啊，貌似和 1.5 版本以前的语义没有变化啊？如果单看这个规则，的确是这样，但是如果我们关联一下规则 3，就有点不一样的感觉了。

### 3. 传递性

这条规则是指如果 A Happens-Before B，且 B Happens-Before C，那么 A Happens-Before C。

我们将规则 3 的传递性应用到我们的例子中，会发生什么呢？可以看下面这幅图：

<img src="https://blog-1304855543.cos.ap-guangzhou.myqcloud.com/blog/image-20250422120234992.png" alt="image-20250422120234992" style="zoom:67%;" />

从图中，我们可以看到：

“x=42” Happens-Before 写变量 “v=true” ，这是规则 1 的内容；

写变量“v=true” Happens-Before 读变量 “v=true”，这是规则 2 的内容 。

再根据这个传递性规则，我们得到结果：“x=42” Happens-Before 读变量“v=true”。这意味着什么呢？

如果线程 B 读到了“v=true”，那么线程 A 设置的“x=42”对线程 B 是可见的。也就是说，线程 B 能看到 “x == 42” 。

代码如下：

```java
public class Transitivity {

    int x = 0;
    volatile boolean v = false;

    public static void main(String[] args) throws InterruptedException {

        Transitivity transitivity = new Transitivity();
        Thread threadA = new Thread(() -> transitivity.writer());
        Thread threadB = new Thread(() -> transitivity.reader());

        threadA.start();
        threadA.join();

        threadB.start();
        threadB.join();
    }


    public void writer() {
        x = 42;
        v = true;
    }

    public void reader() {
        if (v == true) {
            // 这里x会是42
            System.out.println(x);
        }
    }
}
```

### 4. 管程中锁的规则

这条规则是指对一个锁的解锁 Happens-Before 于后续对这个锁的加锁。

要理解这个规则，就首先要了解“管程指的是什么”。**管程**是一种通用的同步原语，在 Java 中指的就是 synchronized，synchronized 是 Java 里对管程的实现。

管程中的锁在 Java 里是隐式实现的，例如下面的代码，在进入同步块之前，会自动加锁，而在代码块执行完会自动释放锁，加锁以及释放锁都是编译器帮我们实现的。

```java
synchronized (this) { //此处自动加锁
  // x是共享变量,初始值=10
  if (this.x < 12) {
    this.x = 12; 
  }  
} //此处自动解锁
```

所以结合规则 4——管程中锁的规则，可以这样理解：假设 x 的初始值是 10，线程 A 执行完代码块后 x 的值会变成 12（执行完自动释放锁），线程 B 进入代码块时，能够看到线程 A 对 x 的写操作，也就是线程 B 能够看到 x==12。这个也是符合我们直觉的，应该不难理解。

### 5. 线程 start() 规则

这条是关于线程启动的。它是指主线程 A 启动子线程 B 后，子线程 B 能够看到主线程在启动子线程 B 前的操作。

换句话说就是，如果线程 A 调用线程 B 的 start() 方法（即在线程 A 中启动线程 B），那么该 start() 操作 Happens-Before 于线程 B 中的任意操作。具体可参考下面示例代码。

```java
public class StartHappenBefore {

    public static void main(String[] args) {
        int i = 0;
        Thread B = new Thread(() -> {
            // 主线程调用B.start()之前
            // 所有对共享变量的修改，此处皆可见
            // 此例中，i==1
            System.out.println(i);
        });

        ThreadUtil.sleep(500);
        // 此处对共享变量i修改
        i = 1;
        // 主线程启动子线程
        B.start();
    }
}
```

### 6. 线程 join() 规则

这条是关于线程等待的。它是指主线程 A 等待子线程 B 完成（主线程 A 通过调用子线程 B 的 join() 方法实现），当子线程 B 完成后（主线程 A 中 join() 方法返回），主线程能够看到子线程的操作。当然所谓的“看到”，指的是对**共享变量**的操作。

换句话说就是，如果在线程 A 中，调用线程 B 的 join() 并成功返回，那么线程 B 中的任意操作 Happens-Before 于该 join() 操作的返回。具体可参考下面示例代码。

```java
public class JoinHappenBefore {

    public static void main(String[] args) {
        int i = 0;
        Thread B = new Thread(() -> {
            // 此处对共享变量i修改
            i = 1;
        });
        
        // 主线程启动子线程
        B.start();
        B.join();
        // 子线程所有对共享变量的修改
        // 在主线程调用B.join()之后皆可见
        // 此例中，i==1
        System.out.println(i);
    }
}
```

## volatile关键字

在Java中，volatile关键字有特殊的内存语义。volatile主要有以下两个功能：

- 保证变量的**内存可见性**
- 禁止volatile变量与普通变量**重排序**（JSR133提出，Java 5 开始才有这个“增强的volatile内存语义”）

### 内存可见性

```java
public class VolatileExample {
    int a = 0;
    volatile boolean flag = false;

    public void writer() {
        a = 1; // step 1
        flag = true; // step 2
    }

    public void reader() {
        if (flag) { // step 3
            System.out.println(a); // step 4
        }
    }
}
```

在这段代码里，我们使用`volatile`关键字修饰了一个`boolean`类型的变量`flag`。

所谓内存可见性，指的是当一个线程对`volatile`修饰的变量进行**写操作**（比如step 2）时，JMM会立即把该线程对应的本地内存中的共享变量的值刷新到主内存；当一个线程对`volatile`修饰的变量进行**读操作**（比如step 3）时，JMM会把立即该线程对应的本地内存置为无效，从主内存中读取共享变量的值。

### 禁止重排序

在JSR-133之前的旧的Java内存模型中，是允许volatile变量与普通变量重排序的。那上面的案例中，可能就会被重排序成下列时序来执行：

1. 线程A写volatile变量，step 2，设置flag为true；
2. 线程B读同一个volatile，step 3，读取到flag为true；
3. 线程B读普通变量，step 4，读取到 a = 0；
4. 线程A修改普通变量，step 1，设置 a = 1；

可见，如果volatile变量与普通变量发生了重排序，虽然volatile变量能保证内存可见性，也可能导致普通变量读取错误。

所以在旧的内存模型中，volatile的写-读就不能与锁的释放-获取具有相同的内存语义了。为了提供一种比锁更轻量级的**线程间的通信机制**，**JSR-133**专家组决定增强volatile的内存语义：严格限制编译器和处理器对volatile变量与普通变量的重排序。

编译器还好说，JVM是怎么还能限制处理器的重排序的呢？它是通过**内存屏障**来实现的。

什么是内存屏障？硬件层面，内存屏障分两种：读屏障（Load Barrier）和写屏障（Store Barrier）。内存屏障有两个作用：

1. 阻止屏障两侧的指令重排序；
2. 强制把写缓冲区/高速缓存中的脏数据等写回主内存，或者让缓存中相应的数据失效。

编译器在**生成字节码时**，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。编译器选择了一个**比较保守的JMM内存屏障插入策略**，这样可以保证在任何处理器平台，任何程序中都能得到正确的volatile内存语义。这个策略是：

- 在每个volatile写操作前插入一个StoreStore屏障；
- 在每个volatile写操作后插入一个StoreLoad屏障；
- 在每个volatile读操作后插入一个LoadLoad屏障；
- 在每个volatile读操作后再插入一个LoadStore屏障。

源码验证：https://cr.openjdk.org/~coleenp/8139203.01/src/share/vm/interpreter/bytecodeInterpreter.cpp-.html

![image-20250422145035084](https://blog-1304855543.cos.ap-guangzhou.myqcloud.com/blog/image-20250422145035084.png)

再介绍一下volatile与普通变量的重排序规则:

1. 如果第一个操作是volatile读，那无论第二个操作是什么，都不能重排序；
2. 如果第二个操作是volatile写，那无论第一个操作是什么，都不能重排序；
3. 如果第一个操作是volatile写，第二个操作是volatile读，那不能重排序。

举个例子，我们在案例中step 1，是普通变量的写，step 2是volatile变量的写，那符合第2个规则，这两个steps不能重排序。而step 3是volatile变量读，step 4是普通变量读，符合第1个规则，同样不能重排序。

但如果是下列情况：第一个操作是普通变量读，第二个操作是volatile变量读，那是可以重排序的：

```java
// 声明变量
int a = 0; // 声明普通变量
volatile boolean flag = false; // 声明volatile变量

// 以下两个变量的读操作是可以重排序的
int i = a; // 普通变量读
boolean j = flag; // volatile变量读
```

### 原子性

对任意单个volatile变量的读/写具有原子性，但类似于volatile++这种复合操作不具有原子性（基于这点，我们通过会认为volatile不具备原子性）。volatile仅仅保证对单个volatile变量的读/写具有原子性，而锁的互斥执行的特性可以确保对整个临界区代码的执行具有原子性。 

64位的long型和double型变量，只要它是volatile变量，对该变量的读/写就具有原子性。

### DCL

```java
public class SingleDcl {

    // volatile 防止指令重排序
    private volatile static SingleDcl singleDcl = null;

    private static SingleDcl getInstance() {
        if (singleDcl == null) {
            synchronized (SingleDcl.class) {
                if (singleDcl == null) {
                    // 1.开辟内存空间
                    // 2.对象初始化
                    // 3.singleDcl指向内存空间的地址
                    singleDcl = new SingleDcl();
                }
            }
        }
        return singleDcl;
    }
}
```

图示如下：

<img src="https://blog-1304855543.cos.ap-guangzhou.myqcloud.com/blog/image-20250422113805462.png" alt="image-20250422113805462" style="zoom:67%;" />

## final关键字

final 修饰变量时，初衷是告诉编译器：这个变量生而不变。

final 字段的可见性由 JMM 的初始化安全性保证，效果类似于内存屏障，但实现上依赖编译器约束和硬件内存模型。在 x86 上，通常不需要显式屏障；在 ARM 等弱内存模型架构上，可能插入 StoreStore 屏障。

# 内存屏障

内存屏障有两种含义，一种是JVM层面的内存屏障，一种是硬件层面的内存屏障。

## JVM层面的内存屏障

在JSR规范中定义了4种内存屏障：

- **LoadLoad屏障**：（指令Load1; LoadLoad; Load2），在Load2及后续读取操作要读取的数据被访问前，保证Load1要读取的数据被读取完毕
- **LoadStore屏障**：（指令Load1; LoadStore; Store2），在Store2及后续写入操作被刷出前，保证Load1要读取的数据被读取完毕
- **StoreStore屏障**：（指令Store1; StoreStore; Store2），在Store2及后续写入操作执行前，保证Store1的写入操作对其它处理器可见
- **StoreLoad屏障**：（指令Store1; StoreLoad; Load2），在Load2及后续所有读取操作执行前，保证Store1的写入对所有处理器可见。它的开销是四种屏障中最大的。在大多数处理器的实现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能。

由于x86只有store load可能会重排序，所以只有JSR的StoreLoad屏障对应它的mfence或lock前缀指令，其他屏障对应空操作。

**硬件层内存屏障**

硬件层提供了一系列的内存屏障 memory barrier / memory fence(Intel的提法)来提供一致性的能力。拿X86平台来说，有几种主要的内存屏障：

1. lfence，是一种Load Barrier 读屏障
2. sfence, 是一种Store Barrier 写屏障
3. mfence, 是一种全能型的屏障，具备lfence和sfence的能力
4. Lock前缀，Lock不是一种内存屏障，但是它能完成类似内存屏障的功能。Lock会对CPU总线和高速缓存加锁，可以理解为CPU指令级的一种锁。它后面可以跟ADD, ADC, AND, BTC, BTR, BTS, CMPXCHG, CMPXCH8B, DEC, INC, NEG, NOT, OR, SBB, SUB, XOR, XADD, and XCHG等指令。

## 内存屏障的实现原理

### 不同架构实现内存屏障

参考：https://gee.cs.oswego.edu/dl/jmm/cookbook.html

![image-20250422143211027](https://blog-1304855543.cos.ap-guangzhou.myqcloud.com/blog/image-20250422143211027.png)

### X86架构内存屏障的实现原理

[orderAccess_linux_x86.inline.hpp](https://hg.openjdk.org/jdk8/jdk8/hotspot/file/87ee5ee27509/src/os_cpu/linux_x86/vm/orderAccess_linux_x86.inline.hpp)

```cpp
inline void OrderAccess::storeload()  { fence(); }
inline void OrderAccess::fence() {
  if (os::is_MP()) {
    // always use locked addl since mfence is sometimes expensive
#ifdef AMD64
    __asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
#else
    __asm__ volatile ("lock; addl $0,0(%%esp)" : : : "cc", "memory");
#endif
  }
```

x86处理器中利用lock实现类似内存屏障的效果。

**lock前缀指令的作用**

1. 确保后续指令执行的原子性。在Pentium及之前的处理器中，带有lock前缀的指令在执行期间会锁住总线，使得其它处理器暂时无法通过总线访问内存，很显然，这个开销很大。在新的处理器中，Intel使用缓存锁定来保证指令执行的原子性，缓存锁定将大大降低lock前缀指令的执行开销。
2. LOCK前缀指令具有类似于内存屏障的功能，禁止该指令与前面和后面的读写指令重排序。
3. LOCK前缀指令会等待它之前所有的指令完成、并且所有缓冲的写操作写回内存(也就是将store buffer中的内容写入内存)之后才开始执行，并且根据缓存一致性协议，刷新store buffer的操作会导致其他cache中的副本失效。

32位的IA-32处理器支持对系统内存中的位置进行锁定的原子操作。这些操作通常用于管理共享的数据结构(如信号量、段描述符、系统段或页表)，在这些结构中，两个或多个处理器可能同时试图修改相同的字段或标志。处理器使用三种相互依赖的机制来执行锁定的原子操作:

- 有保证的原子操作
- 总线锁定，使用LOCK#信号和LOCK指令前缀
-  LOCK前缀指令会等待它之前所有的指令完成、并且所有缓冲的写操作写回内存(也就是将store buffer中的内容写入内存)之后才开始执行，并且根据缓存一致性协议，刷新store buffer的操作会导致其他cache中的副本失效。

## 内存屏障的总结

内存屏障有两个作用：

1. 阻止屏障两边的指令重排序
2. 刷新处理器缓存/冲刷处理器缓存

对Load Barrier来说，在读指令前插入读屏障，可以让高速缓存中的数据失效，重新从主内存加载数据；对Store Barrier来说，在写指令之后插入写屏障，能让写入缓存的最新数据写回到主内存。

Lock前缀实现了类似的能力，它先对总线和缓存加锁，然后执行后面的指令，最后释放锁后会把高速缓存中的数据刷新回主内存。在Lock锁住总线的时候，其他CPU的读写请求都会被阻塞，直到锁释放。

不同硬件实现内存屏障的方式不同，Java内存模型屏蔽了这种底层硬件平台的差异，由JVM来为不同的平台生成相应的机器码。

# **JMM与硬件内存架构的关系**

Java内存模型与硬件内存架构之间存在差异。硬件内存架构没有区分线程栈和堆。对于硬件，所有的线程栈和堆都分布在主内存中。部分线程栈和堆可能有时候会出现在CPU缓存中和CPU内部的寄存器中。如下图所示，Java内存模型和计算机硬件内存架构是一个交叉关系：

![image-20250422145321430](https://blog-1304855543.cos.ap-guangzhou.myqcloud.com/blog/image-20250422145321430.png)

# 并发特性总结

**可见性** 

当一个线程修改了共享变量的值，其他线程能够看到修改的值。Java 内存模型是通过在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值这种依赖主内存作为传递媒介的方法来实现可见性的。

**如何保证可见性**

- 通过 volatile 关键字保证可见性。
- 通过 内存屏障保证可见性。
- 通过 synchronized 关键字保证可见性。
- 通过 Lock保证可见性。
- 通过 final 关键字保证可见性

**有序性**

即程序执行的顺序按照代码的先后顺序执行。JVM 存在指令重排，所以存在有序性问题。

**如何保证有序性**

- 通过 volatile 关键字保证可见性。
- 通过 内存屏障保证可见性。
- 通过 synchronized关键字保证有序性。
- 通过 Lock保证有序性。

**原子性**

一个或多个操作，要么全部执行且在执行过程中不被任何因素打断，要么全部不执行。在 Java 中，对基本数据类型的变量的读取和赋值操作是原子性操作（64位处理器）。不采取任何的原子性保障措施的自增操作并不是原子性的。

**如何保证原子性**

- 通过 synchronized 关键字保证原子性。
- 通过 Lock保证原子性。
- 通过 CAS保证原子性。

# 参考链接

https://gee.cs.oswego.edu/dl/jmm/cookbook.html

https://www.cs.umd.edu/~pugh/java/memoryModel/CommunityReview.pdf

https://freegeektime.com/100026001/109874/

http://ifeve.com/wp-content/uploads/2014/03/JSR133%E4%B8%AD%E6%96%87%E7%89%88.pdf

# 思考题

1.为什么在 32 位的机器上对 long 型变量进行加减操作存在并发隐患？

2.final关键字没有使用内存屏障是如何保证可见性的？

参考：https://github.com/CL0610/Java-concurrency/blob/master/06.%E4%BD%A0%E4%BB%A5%E4%B8%BA%E4%BD%A0%E7%9C%9F%E7%9A%84%E4%BA%86%E8%A7%A3final%E5%90%97%EF%BC%9F/java%E5%85%B3%E9%94%AE%E5%AD%97--final.md

