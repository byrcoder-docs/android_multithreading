### Android多线程
使用多线程的好处通常包括提高了代码的效率，充分利用计算资源，减少系统响应时间等。譬如可以将问题划分为子问题并将子问题交给不同的线程进行处理，如果这些线程需要共享一些争用资源，那么通常对这些争用资源的访问(读或者写操作)是需要进行**互斥操作**的，如果这些线程在某些时候需要以一定次序进行，那么则需要进行**同步操作**。互斥操作和同步操作一般都称为**多线程同步问题**。
Android为了保证系统对用户保持高响应性，更是强制规定了在Activity的主线程中的操作不能超过5秒，Service的主线程中的操作不能超过10秒，否则会抛出`ANR异常`。这使得我们必须要将耗时操作转移到工作线程中去。
>**Tip:** Android甚至都不允许在主线程中进行网络操作，当然这是可以理解的，因为网络操作的耗时往往是无法预期的，这取决于网络状况。

#### 1. 显式创建线程
在Java/Android中显式创建线程通常有两种方式:
1. 继承Thread类
2. 实现Runnable接口

第二种方式比第一种方式要更加优雅。对于Android中隐式创建线程的方法，譬如`AyncTask`类，将在后面章节做讲解。下面展示分别用两种方式显式创建线程的例子。
##### 1.1 继承Thread类
```java
public class MyThread extends Thread {
    @Override
    public void run() {
        super.run();
        LogUtil.logThreadId("MyThread::run()");
    }
}
```

```java
new MyThread().start();
```

##### 1.2 实现Runnable接口
```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        LogUtil.logThreadId("MyRunnable::run()");
    }
}
```

```java
new Thread(new MyRunnable()).start();
```
通常也可以使用匿名函数简化代码
```java
        new Thread(new Runnable() {
            @Override
            public void run() {
                //TODO...
            }
        }).start();
```

#### 2. 线程池
创建和销毁线程本身是一个耗时耗资源的操作，持有过多线程本身也需要消耗资源，因此应该尽可能避免创建过多的线程。一个方法是使用线程池来复用已创建的线程。Java的`ThreadPoolExecutor`类可以帮助我们创建线程池并利用线程池来执行多线程任务。
>**Tip:** `ThreadPoolExecutor`继承自`AbstractExecutorService`，而`AbstractExecutorService`实现了`ExecutorService`接口。

先来看看`ThreadPoolExecutor`这个核心类。

##### 2.1 ThreadPoolExecutor构造函数
在ThreadPoolExecutor类中提供了多种构造方法用来创建线程池，典型的有：
```java
        public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize,
                                  long keepAliveTime, TimeUnit unit,
                                  BlockingQueue<Runnable> workQueue);
```
其中`corePoolSize`是指线程池维持的线程数量；`maximumPoolSize`是线程池最大线程数；`keepAliveTime`是线程池闲置线程存活时间；`unit`是这个存活时间的单位；`workQueue`是等待队列。
需要指出的是通常超时只意味着销毁那些超出`corePoolSize`数量的线程，当`corePoolSize`与`maximumPoolSize`相等时，`keepAliveTime`通常没有什么意义，设为`0L`即可。
`workQueue`典型类型有以下两种：
1. **ArrayBlockingQueue：**基于数组的先进先出队列，此队列创建时必须指定大小；
2. **LinkedBlockingQueue：**基于链表的先进先出队列，不需要指定此队列大小；
3. **synchronousQueue：**这个队列比较特殊，它不会保存提交的任务，而是将直接新建一个线程来执行新来的任务。

>**Tip:** 当ArrayBlockingQueue满时，再有新的任务到来，会抛出异常。


##### 2.2 ThreadPoolExecutor几个重要方法
```java
execute()
shutdown()
shutdownNow()
```
`execute()`接受一个`Runnable`对象，用来向线程池提交一个任务，交由线程池去执行。
`shutdown()`和`shutdownNow()`都是用来关闭线程池的。他们的区别在于`shutdown()`不会立刻关闭线程池，而等到所有的等待队列的任务执行完才终止，但不会接受新的任务了；`shutdownNow()`会立刻关闭线程池，试图打断正在执行的任务，清空等待队列，返回尚未执行的任务。

##### 2.3 使用ThreadPoolExecutor的例子
```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        LogUtil.logThreadId("MyRunnable::run()");
    }
}
```
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(5, 5, 200, TimeUnit.MILLISECONDS,
        new ArrayBlockingQueue<Runnable>(5));

for (int i = 0; i < 10; ++i) {
    MyRunnable myRunnable = new MyRunnable();
    executor.execute(myRunnable);
    Log.d("gyw", "poolSize = " + executor.getPoolSize() +
            "; queueSize = " + executor.getQueue().size() +
            "; accomplish = " + executor.getCompletedTaskCount());
}
executor.shutdown();
```
执行以后可以看到当线程池线程会被重用，当线程池中线程都被占用时，任务会被缓存到等待队列。

##### 2.4 Executors类的几个静态创建线程池方法
ThreadPoolExecutor的构造函数参数众多，可能还不够便于使用，事实上可以利用Executors类的几个静态方法来创建线程池。
```java
Executors.newSingleThreadExecutor();   //创建容量为1的线程池
Executors.newFixedThreadPool(int threadNum);    //创建固定容量大小的线程池
Executors.newCachedThreadPool();   //创建一个缓冲池
```
前两个方法创建的线程池`corePoolSize`和`maximumPoolSize`值是相等的，使用`LinkedBlockingQueue`；最后一个方法`corePoolSize = 0`，`maximumPoolSize = Integer.MAX_VALUE`，使用`SynchronousQueue`，超时为60s，这意味着来了任务就会创建线程运行，当线程空闲60s以后就会被销毁。
这里给一个简单的使用例子，和`2.3节`例子是一致的。
```java
ExecutorService executor = Executors.newFixedThreadPool(5);
for (int i = 0; i < 10; ++i) {
    MyRunnable myRunnable = new MyRunnable();
    executor.execute(myRunnable);
}
executor.shutdown();
```

>**Resource:** 对于Java线程池更多的知识，可以阅读以下网址：
>[http://www.cnblogs.com/dolphin0520/p/3932921.html](http://www.cnblogs.com/dolphin0520/p/3932921.html)

#### 3. Java内存模型与线程可见性
先来看个典型的例子
```java
//Ex3.1	线程可见性例子
public class Test extends Thread {

    boolean keepRunning = true;

    public static void main(String[] args) throws InterruptedException {
        Test t = new Test();
        t.start();
        Thread.sleep(1000);
        t.keepRunning = false;
        System.out.println(System.currentTimeMillis() + ": keepRunning is false");
    }

    public void run() {
        while (keepRunning)
            System.out.println(System.currentTimeMillis() + ": " + keepRunning);
    }
}
```
从类设计者角度来说，其本意是在Test线程运行后1秒后在主线程中将`keepRunning`变量设为`false`,以此来终止Test线程中一直保持的循环。然而这对于Java来说并不总能做到。事实上，**Java的内存模型并不能保证一个线程中对变量的修改可以立即对其他线程可见**。`Ex3.1`运行的结果将取决于运行环境，一种可能的情况是Test线程的循环将一直循环下去，这将背离类设计者的意图。
解决这个问题的途径有两个：
1. 使用**volatile**关键字
2. 使用**synchronzied**关键字  

第二种方案留在后面做讲解，这里只讲解使用volatile关键字。
本质上将，volatile保证了变量在cache中的值总是最新的，从而可以**保证对volatile变量的修改可以立刻对所有线程可见**。通常在**一个或多个读线程/一个写线程**的情况下，将共享对象设置为volatile就可以解决问题，这时候volatile的作用类似于一个写锁。上面的代码只需简单修改为如下即可：
```
volatile boolean keepRunning = true;
```
>**Caution:** 必须值得注意的一点是，volatile相当于写锁有效的前提是，写的操作必须是**原子的**。如果写的操作不是原子的(譬如写操作不是`t.keepRunning = false`而是类似于`mIntIndex++`，`'++'`操作不是原子操作)，则会造成同步问题。


#### 4. 基本并发模型
上面说到volatile关键字可以保证线程可见性，但并不保证线程是同步的，譬如对共享对象的并发修改，volatile无法保证修改是互斥进行的。这时候需要使用互斥操作。
通常意义上讲，多线程并发有三种基本情况：
1. Read / Read 并发模型
2. Read / Write 并发模型
3. Write / Write 并发模型

对于**Read/Read**情形并不需要使用锁来进行，仅存在对数据的读操作是线程安全的。**Write/Write**情形需要使用互斥锁来解决并发问题。**Read/Write**情形虽然也可以使用互斥锁来解决并发问题，但是使用读写锁则并发效率更高。

#### 5. 互斥锁
互斥锁可以进行互斥操作，在Java中可以通过`synchronized`关键字或者`ReentrantLock`来通过对**临界区**加互斥锁的方式解决并发问题。这对于Read/Write并发模型和Write/Write并发模型都是有效的。
先来看一个对共享对象并发修改有问题的例子：
```java
public class Test1 {

    private volatile int count = 0;

    public void increaseCount() {
        for (int i = 0; i < 500; ++i) {
            count++;
            Log.d("gyw", "count = " + count);
        }
    }
}
```

```java
final Test1 test1 = new Test1();

Thread t1 = new Thread(new Runnable() {
    @Override
    public void run() {
        test1.increaseCount();
    }
});

Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        test1.increaseCount();
    }
});

t1.start();
t2.start();
```
这是一个典型的Write/Write问题，当多个线程同时执行对count的写操作时，就会发生同步问题，解决这个问题可以使用互斥操作。即多个线程对count的写操作是互斥的。

##### 5.1 对方法加synchronized关键字修饰
```java
public class Test1 {

    private volatile int count = 0;

    public synchronized void increaseCount() {
        for (int i = 0; i < 500; ++i) {
            count++;
            Log.d("gyw", "count = " + count);
        }
    }
}
```
对`increaseCount()`方法加`synchronized`关键词以后使得对该方法的访问变成互斥操作，同一时间只允许有一个线程执行该方法体。

##### 5.2 静态方法和非静态方法加synchronized的区别
对静态方法加synchronized关键字，使得所有该类的实例在不同线程中对该静态方法的调用都是互斥的(相当于**给类加互斥锁**)。而非静态方法加synchronized关键字，只保证该类的同一个实例对非静态方法的访问是互斥的(相当于**给this加互斥锁**)。

##### 5.3 synchronized块
对方法加synchronized可以简单、快捷地对该方法实现多线程互斥操作，然而很多时候并不是方法中的所有语句都需要进行互斥操作，更准确地说，只有那些处于**临界区**的语句需要互斥操作，因而可以使用synchronized块来构建临界区，这样做提高了并发性和吞吐量。
```java
public class Test1 {

    private volatile int count = 0;
    private Object myLock = new Object();

    public synchronized void increaseCount() {
        for (int i = 0; i < 500; ++i) {
            synchronized (myLock) {
                count++;
                Log.d("gyw", "count = " + count);
            }
        }
    }
}
```
>**Caution:** 使用Synchronized是一种简单有效地支持多线程同步的方法，然而同步操作会降低吞吐量，并不是所有的东西都需要加锁保护，应该将**锁粒度最小化**。另外，更糟糕的是，加锁操作有可能会导致**死锁**。事实上最容易发生死锁的情况就是在一个同步代码块中调用了另一个对象的方法，而那个对象已经被锁住并等待当前对象释放该锁。因此尽量**不要在synchronized同步块中调用另一个对象的方法**，除非你非常清楚另一个对象的类的代码不会发生死锁。

##### 5.4 ReentrantLock
ReentrantLock是JDK1.5新增加的互斥锁，通常情况下，`ReentrantLock`会比`synchronized`效率要高，尤其是在搞资源争用情形下。然而，这并不表示我们要在所有情形下使用ReentrantLock替代synchronized，编写程序有一个重要的够用原则，只有在需要优化的情形下才进行优化，通常情形下synchronized的效率已经完全够用，此时并不需要改用ReentrantLock。辩证唯物地看，synchronized也具有不需要显式释放互斥锁，支持低版本JDK等优点。
>**Resource:** 关于ReentrantLock和synchronized的效率对比以及什么时候该使用ReentrantLock的更多知识，可以参见以下链接：
>[http://www.ibm.com/developerworks/library/j-jtp10264/](http://www.ibm.com/developerworks/library/j-jtp10264/)

以下代码用来展示使用ReentrantLock解决本节的并发互斥问题。
```java
public class Test1 {

    private volatile int count = 0;
    private Object myLock = new Object();

    private ReentrantLock lock = new ReentrantLock();

    public synchronized void increaseCount() {
        for (int i = 0; i < 500; ++i) {
            lock.lock();
                count++;
                Log.d("gyw", "count = " + count);
            lock.unlock();
        }
    }
}
```

#### 6. 读写锁
对于**Read/Write**问题可以考虑使用读写锁来解决并发问题，对于**读写锁可以保证可以有多个线程同时对资源进行读操作，而只有一个线程可以在某个时刻对资源进行写操作。**写锁类似于synchronized，获得写锁的线程可以独享写的操作，其他线程无法获得写锁或者读锁，直到写锁被释放。获得读锁的线程可以允许其他线程获得读锁，但直到所有的读锁被释放才允许某个线程获得写锁。
首先看一个简单的并发读写例子。
```java
public class Test2 {

    private static final int SIZE = 300;
    private volatile int a[][] = new int[SIZE][SIZE];

    public void write() {
        for (int i = 0; i < SIZE; i++) {
            for (int j = 0; j < SIZE; ++j) {
                a[i][j]++;
                a[j][i]--;
            }
		}
    }

    public void read() {
        int value = 0;
        for (int i = 0; i < SIZE; i++) {
            for (int j = 0; j < SIZE; ++j) {
                value += a[i][j];
                value += a[j][i];
            }
        }
        Log.d("gyw", "value = " + value);
    }
}
```

```java
final Test2 test2 = new Test2();

Thread t1 = new Thread(new Runnable() {
    @Override
    public void run() {
        test2.write();
    }
});

Thread t2 = new Thread(new Runnable() {
    @Override
    public void run() {
        test2.read();
    }
});

t1.start();
t2.start();
```

在这个例子中打印出来的value结果将通常不是0，解决这样的并发问题当然可以使用`第5节`中加`synchronized`或者`ReentrantLock`的方法，譬如：
```java
public synchronized void write() {
    for (int i = 0; i < SIZE; i++) {
        for (int j = 0; j < SIZE; ++j) {
            a[i][j]++;
            a[j][i]--;
        }
	}
}

public synchronized void read() {
    int value = 0;
    for (int i = 0; i < SIZE; i++) {
        for (int j = 0; j < SIZE; ++j) {
            value += a[i][j];
            value += a[j][i];
        }
    }
    Log.d("gyw", "value = " + value);
}
```
然而使用读写锁可以进一步提高并发吞吐量，提高程序的效率。在Java中可以使用`ReentrantReadWriteLock`来实现读写锁；配对使用`readWriteLock.readLock().lock()`和`readWriteLock.readLock().unlock()`来获取/释放读锁；配对使用`readWriteLock.writeLock().lock();`和`readWriteLock.writeLock().unlock();`来获取/释放写锁。
使用读写锁的例子如下所示：
```java
public class Test2 {

    private static final int SIZE = 300;
    private volatile int a[][] = new int[SIZE][SIZE];

    private ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();

    public void write() {
        for (int i = 0; i < SIZE; i++) {
            for (int j = 0; j < SIZE; ++j) {
                readWriteLock.writeLock().lock();
                a[i][j]++;
                a[j][i]--;
                readWriteLock.writeLock().unlock();
            }
		}
    }

    public synchronized void read() {
        int value = 0;
        for (int i = 0; i < SIZE; i++) {
            for (int j = 0; j < SIZE; ++j) {
                readWriteLock.readLock().lock();
                value += a[i][j];
                value += a[j][i];
                readWriteLock.readLock().unlock();
            }
        }
        Log.d("gyw", "value = " + value);
    }
}
```

#### 7. 无锁操作
`第5节`和`第6节`分别详述了使用互斥锁和读写锁来解决对共享变量的多线程并发问题。事实上，加锁操作意味着线程需要等待，一般来讲，会损失效率，当然在绝大多数情况下，这并不是特别大的问题。
>**Tip:** 还是重复之前那个原则，**程序编写的原则是够用原则**，只有在需要优化的情况下才考虑对效率的优化，并且优化的首要步骤是改进算法而不是改进实现，通常项目进度、异常处理、代码的可维护性是更重要的因子，**优化需要抱着务实的态度**。

本节将讲述无锁操作，无锁操作保证线程安全性的前提下不使用锁机制，从而具有很高的并发效率。在Java中这可以通过`Atomic`变量来实现。Java定义了一系列的`Atomic`变量和相应的对这些变量的原子操作。这些变量包括`AtomicBoolean`，`AtomicInteger`，`AtomicLong`等。
以下兹举一个使用`AtomicInteger`进行安全并发操作的例子。
```java
public class Test3 {

    private volatile AtomicInteger count = new AtomicInteger(0);

    public synchronized void increaseCount() {
        for (int i = 0; i < 500; ++i) {
            count.addAndGet(1); //类似于++count
            Log.d("gyw", "count = " + count);
        }
    }
}
```
>**Warnning** 注意使用`volatile`关键字来保证线程可见性。

#### 8. 线程安全的数据结构
`第7节`讲到了一些基本的`Atomic`变量可以用于并发操作，Java支持对一些更高级数据结构的安全访问，譬如`CopyOnWriteArrayList`和`ConcurrentHashMap`,分别支持数组和映射数据结构进行线程安全地访问。
>**Tip:** `HashTable`也是线程安全的，然而其效率要低于`ConcurrentHashMap`，这是由于`ConcurrentHashMap`采取了部分锁机制，而非使用全锁。更深入的了解可以点击下面的网址：
>[http://wiki.jikexueyuan.com/project/java-collection/concurrenthashmap.html](http://wiki.jikexueyuan.com/project/java-collection/concurrenthashmap.html)

先看一个使用`HashMap`导致线程不安全的例子
```java
public class Test4 {

    private volatile HashMap<Integer, Integer> unsafeMap = new HashMap<>();

    public void writeElements() {
        for (int i = 0; i < 100; ++i) {
            unsafeMap.put(new Integer(i), i);
        }
        LogUtil.logThreadId( "map size = " + unsafeMap.size());
    }
}
```

```java
final Test4 test4 = new Test4();

for (int i = 0; i < 20; ++i) {
    new Thread(new Runnable() {
        @Override
        public void run() {
            test4.writeElements();
        }
    }).start();
}
```
打印出的结果(每一次运行结果不同)：
```
09-14 23:37:49.387  32099-32113/? I/gyw﹕ TID:6354; map size = 103
09-14 23:37:49.387  32099-32114/? I/gyw﹕ TID:6355; map size = 103
09-14 23:37:49.387  32099-32115/? I/gyw﹕ TID:6356; map size = 103
09-14 23:37:49.387  32099-32117/? I/gyw﹕ TID:6358; map size = 103
09-14 23:37:49.387  32099-32116/? I/gyw﹕ TID:6357; map size = 103
09-14 23:37:49.397  32099-32118/? I/gyw﹕ TID:6359; map size = 103
09-14 23:37:49.397  32099-32119/? I/gyw﹕ TID:6360; map size = 103
09-14 23:37:49.397  32099-32120/? I/gyw﹕ TID:6361; map size = 103
09-14 23:37:49.397  32099-32121/? I/gyw﹕ TID:6362; map size = 103
09-14 23:37:49.397  32099-32122/? I/gyw﹕ TID:6363; map size = 103
09-14 23:37:49.397  32099-32123/? I/gyw﹕ TID:6364; map size = 103
09-14 23:37:49.397  32099-32124/? I/gyw﹕ TID:6365; map size = 103
09-14 23:37:49.397  32099-32127/? I/gyw﹕ TID:6368; map size = 103
09-14 23:37:49.397  32099-32125/? I/gyw﹕ TID:6366; map size = 103
09-14 23:37:49.397  32099-32126/? I/gyw﹕ TID:6367; map size = 103
09-14 23:37:49.397  32099-32128/? I/gyw﹕ TID:6369; map size = 103
09-14 23:37:49.397  32099-32129/? I/gyw﹕ TID:6370; map size = 103
09-14 23:37:49.397  32099-32130/? I/gyw﹕ TID:6371; map size = 103
09-14 23:37:49.397  32099-32131/? I/gyw﹕ TID:6372; map size = 103
09-14 23:37:49.397  32099-32132/? I/gyw﹕ TID:6373; map size = 103
```
如果线程操作是安全的，则map size应该为100。为了得到正确的结果，使用`ConcurrentHashMap`替代`HashMap`即可。
```java
public class Test4 {

    private volatile ConcurrentHashMap<Integer, Integer> safeMap = new ConcurrentHashMap<>();

    public void writeElements() {
        for (int i = 0; i < 100; ++i) {
            safeMap.put(new Integer(i), i);
        }
        LogUtil.logThreadId( "map size = " + safeMap.size());
    }
}
```

>**Warning:** 使用线程安全的数据结构可以确保多数据结构的多线程访问是安全的，通常单个对数据结构的操作是原子的，但这**绝不保证多个对数据结构的操作也是原子的**。如果后面的操作是依赖于前面操作的结果，需要清醒地意识到在别的线程中可以会存在对数据的修改。
>```java
>    public void not_a_transaction() {
        safeMap.put(someKey1, someValue1);
        //other threads could execute between those two statements
        safeMap.get(someKey2);
    }
>```
>在put和get之间是有可能存在cpu被其他线程占用并执行对safeMap操作的情形，如果需要确保这样的操作是事物性的，可以考虑使用`synchronized`。
