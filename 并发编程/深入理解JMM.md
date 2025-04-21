# 可见性、有序性、原子性

## 可见性

可见性的问题：当前线程对共享变量的操作会存在读不到，或者不能立即读到另外一个线程对此变量的写操作。

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

## 有序性

指令重排序

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

# 什么是内存模型？

一般来说，编程语言也可以直接复用操作系统层面的内存模型。不过，不同的操作系统内存模型不同。如果直接复用操作系统层面的内存模型，就可能会导致同样一套代码换了一个操作系统就无法执行了。Java 语言是跨平台的，它需要自己提供一套内存模型以屏蔽系统差异。

一般来说，编程语言也可以直接复用操作系统层面的内存模型。不过，不同的操作系统内存模型不同。如果直接复用操作系统层面的内存模型，就可能会导致同样一套代码换了一个操作系统就无法执行了。Java 语言是跨平台的，它需要自己提供一套内存模型以屏蔽系统差异。

这只是 JMM 存在的其中一个原因。实际上，对于 Java 来说，你可以把 JMM 看作是 Java 定义的并发编程相关的一组规范，除了抽象了线程和主内存之间的关系之外，其还规定了从 Java 源代码到 CPU 可执行指令的这个转化过程要遵守哪些和并发相关的原则和规范，其主要目的是为了简化多线程编程，增强程序可移植性的。

**为什么要遵守这些并发相关的原则和规范呢？** 这是因为并发编程下，像 CPU 多级缓存和指令重排这类设计可能会导致程序运行出现一些问题。就比如说我们上面提到的指令重排序就可能会让多线程程序的执行出现问题，为此，JMM 抽象了 happens-before 原则（后文会详细介绍到）来解决这个指令重排序问题。

JMM 说白了就是定义了一些规范来解决这些问题，开发者可以利用这些规范更方便地开发多线程程序。对于 Java 开发者说，你不需要了解底层原理，直接使用并发相关的一些关键字和类（比如 `volatile`、`synchronized`、`final`各种 `Lock`）即可开发出并发安全的程序。

Java线程之间的通信由Java内存模型（简称JMM）控制，从抽象的角度来说，JMM定义了线程和主内存之间的抽象关系。JMM的抽象示意图如图所示：

<img src="https://blog-1304855543.cos.ap-guangzhou.myqcloud.com/blog/image-20250421233422584.png" alt="image-20250421233422584" style="zoom:50%;" />

**什么是主内存？什么是本地内存？**

- **主内存**：所有线程创建的实例对象都存放在主内存中，不管该实例对象是成员变量，还是局部变量，类信息、常量、静态变量都是放在主内存中。为了获取更好的运行速度，虚拟机及硬件系统可能会让工作内存优先存储于寄存器和高速缓存中。
- **本地内存**：每个线程都有一个私有的本地内存，本地内存存储了该线程以读 / 写共享变量的副本。每个线程只能操作自己本地内存中的变量，无法直接访问其他线程的本地内存。如果线程间需要通信，必须通过主内存来进行。本地内存是 JMM 抽象出来的一个概念，并不真实存在，它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器优化。

从图中可以看出： 1. 所有的共享变量都存在主内存中。 2. 每个线程都保存了一份该线程使用到的共享变量的副本。 3. 如果线程A与线程B之间要通信的话，必须经历下面2个步骤： 1. 线程A将本地内存A中更新过的共享变量刷新到主内存中去。 2. 线程B到主内存中去读取线程A之前已经更新过的共享变量。

**所以，线程A无法直接访问线程B的工作内存，线程间通信必须经过主内存。**

注意，根据JMM的规定，**线程对共享变量的所有操作都必须在自己的本地内存中进行，不能直接从主内存中读取**。

所以线程B并不是直接去主内存中读取共享变量的值，而是先在本地内存B中找到这个共享变量，发现这个共享变量已经被更新了，然后本地内存B去主内存中读取这个共享变量的新值，并拷贝到本地内存B中，最后线程B再读取本地内存B中的新值。

那么怎么知道这个共享变量的被其他线程更新了呢？这就是JMM的功劳了，也是JMM存在的必要性之一。**JMM通过控制主内存与每个线程的本地内存之间的交互，来提供内存可见性保证**。	

# volatile语义

# 再谈内存屏障

# happen before



# 参考链接

https://gee.cs.oswego.edu/dl/jmm/cookbook.html

https://www.cs.umd.edu/~pugh/java/memoryModel/CommunityReview.pdf

https://freegeektime.com/100026001/109874/