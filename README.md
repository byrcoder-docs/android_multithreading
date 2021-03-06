## Android多线程
使用多线程的好处通常包括提高了代码的效率，充分利用计算资源，减少系统响应时间等。譬如可以将问题划分为子问题并将子问题交给不同的线程进行处理，如果这些线程需要共享一些争用资源，那么通常对这些争用资源的访问(读或者写操作)是需要进行**互斥操作**的（解决并发问题）；如果这些线程在某些时候需要以一定次序进行，那么则需要进行**同步操作**（解决协作问题）。互斥操作和同步操作一般都称为**多线程同步问题**；另外当工作线程工作结束后，主线程可能还想知道工作线程的执行结果，不同的工作线程之间可能也希望进行**通信**(解决通信问题)。   
Android为了保证系统对用户保持高响应性，更是强制规定了在`Activity`的主线程中的操作不能超过5秒，`Service`的主线程中的操作不能超过10秒，否则会抛出`ANR异常`。这使得我们必须要将耗时操作转移到工作线程中去。
>**Tip:** Android甚至都不允许在主线程中进行网络操作，当然这是可以理解的，因为网络操作的耗时往往是无法预期的，这取决于网络状况。

### 1. 线程基本操作
这里介绍几个最常用的线程基本操作，包括创建线程、中断线程、休眠线程、`join()`、阻塞/唤醒线程等方法。  
#### 1.1 创建线程
在Java/Android中显式创建线程通常有两种方式:  
1. 继承Thread类  
2. 实现Runnable接口  

第二种方式比第一种方式要更加优雅。对于Android中隐式创建线程的方法，譬如`AyncTask`类，将在后面章节做讲解。下面展示分别用两种方式显式创建线程的例子。
##### 继承Thread类
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

##### 实现Runnable接口
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

#### 1.2 中断线程
可以通过对指定线程调用`interrupt()`的方法来试图中断线程。
```
MyThread thread = new MyThread();
thread.interrupt();
```
调用`interrupt()`方法通常并不能使得正在运行中的线程停下来，但`interrupt()`方法可以使得处于阻塞的线程抛出`InterruptedException`异常进而终止。

>**Tip:** `interrupt()`仅仅是通过优雅的方式试图中断线程，正如前文说到，通常情况下`interrupt()`方法并不能使得正在运行的线程停下来。当然，**请不要调用`stop()`或者`destroy()`方法粗鲁地终止线程**，这些方法是废弃的方法，会造成对象状态不一致等诸多问题。

如果确实需要终止线程的运行，可以增加一个`shouldStop`属性通过循环判断语句来达到终止线程运行的目的。以下给出一个例子。
```java
public class MyThread extends Thread {
    private volatile boolean shouldStop = false;
    @Override
    public void run() {
        super.run();
        Log.d("gyw", "worker thread started");
        int i = 0;
        while (!shouldStop) {
            LogUtil.logThreadId("i = " + i);
            i++;
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        Log.d("gyw", "worker thread finished");
    }

    public void setStop() {
        shouldStop = true;
    }
}
```

```java
MyThread myThread = new MyThread();
myThread.start();

try {
    Thread.sleep(500);
} catch (InterruptedException e) {
    e.printStackTrace();
} finally {
    myThread.setStop();
}
```
代码中让主线程在启动`myThread`工作线程后就进入睡眠状态500ms，醒来后就通过`setStop()`方法试图终端工作线程的工作，而工作线程的`while`循环大约每100ms会执行一次，这样大概会执行5次左右后工作线程终止，`log日志`会打印大约5次左右后结束。事实上运行结果与我们预期完全相符。
```
09-18 09:14:14.986  11955-11969/? D/gyw﹕ worker thread started
09-18 09:14:14.986  11955-11969/? I/gyw﹕ TID:8109; i = 0
09-18 09:14:15.086  11955-11969/? I/gyw﹕ TID:8109; i = 1
09-18 09:14:15.186  11955-11969/? I/gyw﹕ TID:8109; i = 2
09-18 09:14:15.286  11955-11969/? I/gyw﹕ TID:8109; i = 3
09-18 09:14:15.386  11955-11969/? I/gyw﹕ TID:8109; i = 4
09-18 09:14:15.486  11955-11969/? D/gyw﹕ worker thread finished
```

#### 1.3 sleep和yield方法
`sleep()`的调用可以使线程进入睡眠状态，会交出CPU给别的线程去使用。在`1.2节`的例子中已经提前使用了`sleep()`方法。值得注意的是，调用`sleep()`方法并不会释放已经持有的锁。  
`yield()`类似于`sleep()`，也会交出CPU给别的线程并且也不会释放已经持有的锁，不同的是`yield()`方法并不指出线程应该睡眠多长时间，仅仅是让出CPU以使得同一优先级的线程有机会获得CPU。

>**Tip:** 关于什么是锁？锁在多线程中有什么具体用途用法？如果尚且不清楚或者不理解，没有关系，请继续往后阅读，后面会详细叙述。

#### 1.4 join方法
当在主线程中创建工作线程后，可以在主线程中调用`join()`方法来等待工作线程完成，然后再执行接下来的代码。以下是一个简单的例子。
```java
public class MyThread extends Thread {
    @Override
    public void run() {
        super.run();
        LogUtil.logThreadId(" worker thread");
    }
}
```

```java
MyThread myThread = new MyThread();
myThread.start();

try {
    myThread.join();
} catch (InterruptedException e) {
    e.printStackTrace();
}
```
运行结果如下：
```
09-18 09:36:40.926  12322-12335/? I/gyw﹕ TID:8115;  worker thread
09-18 09:36:40.926  12322-12322/? I/gyw﹕ TID:1;  main thread
```
可以看到，主线程等待工作线程执行完以后，才继续执行之后的代码。

#### 1.5 wait/notify方法
`wait()`方法使得调用的线程被阻塞，并监视调用该方法的`Object`是否有`notify`消息，如果有`notify`消息，则唤醒被阻塞的线程。  
`notify()`会唤醒通过同一个`Object`来调用`wait()`而被阻塞的线程。

>**Tip:** `notify()`和`wait()`必须作用在同一个对象上才能配合起来阻塞/唤醒某一个线程。

`notify()`和`notifyAll()`的区别在于前者唤醒第一个被阻塞的线程；而后者唤醒所有线程，拥有最高优先级的线程将获得CPU。  
以下还是通过一个简单的例子来加深对wait/notify方法的理解。
```java
public class MyThread extends Thread {
    public volatile Object object = new Object();

    @Override
    public void run() {
        super.run();
        try {
            synchronized (object) {
                object.wait();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        LogUtil.logThreadId(" worker thread");
    }
}
```

```java
MyThread myThread = new MyThread();
myThread.start();

LogUtil.logThreadId("start worker thread");

try {
    Thread.sleep(500);
} catch (InterruptedException e) {
    e.printStackTrace();
}

LogUtil.logThreadId("notify worker thread");
synchronized (myThread.object) {
    myThread.object.notify();
}
```
运行结果如下：
```
09-18 10:19:31.745  13737-13737/? I/gyw﹕ TID:1; start worker thread
09-18 10:19:32.245  13737-13737/? I/gyw﹕ TID:1; notify worker thread
09-18 10:19:32.245  13737-13751/? I/gyw﹕ TID:8181;  worker thread
```

>**Warning:** 通过某个对象使用`wait()`或者`notify()`时，必须给该对象加锁，否则将会抛出异常。

#### 1.6 线程状态转换模型图
本节最后以一个较详细的线程状态转换模型图来做结尾，以加深对线程状态及状态之间转移关系的理解。

![](https://lh3.googleusercontent.com/--CAG2wf6fIw/Vft8ooslvKI/AAAAAAAAA9k/JKP37-9myyw/s720-Ic42/061046391107893.jpg)

### 2. 线程池
创建和销毁线程本身是一个耗时耗资源的操作，持有过多线程本身也需要消耗资源，因此应该尽可能避免创建过多的线程。一个方法是使用线程池来复用已创建的线程。Java的`ThreadPoolExecutor`类可以帮助我们创建线程池并利用线程池来执行多线程任务。
>**Tip:** `ThreadPoolExecutor`继承自`AbstractExecutorService`，而`AbstractExecutorService`实现了`ExecutorService`接口。

先来看看`ThreadPoolExecutor`这个核心类。

#### 2.1 ThreadPoolExecutor构造函数
在ThreadPoolExecutor类中提供了多种构造方法用来创建线程池，典型的有：
```java
public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize,
                          long keepAliveTime, TimeUnit unit,
                          BlockingQueue<Runnable> workQueue);
```
其中`corePoolSize`是指线程池维持的线程数量；`maximumPoolSize`是线程池最大线程数；`keepAliveTime`是线程池闲置线程存活时间；`unit`是这个存活时间的单位；`workQueue`是等待队列。   
需要指出的是通常超时只意味着销毁那些超出`corePoolSize`数量的线程，当`corePoolSize`与`maximumPoolSize`相等时，`keepAliveTime`通常没有什么意义，设为`0L`即可。   
`workQueue`典型类型有以下三种：  
1. **ArrayBlockingQueue：**基于数组的先进先出队列，此队列创建时必须指定大小；     
2. **LinkedBlockingQueue：**基于链表的先进先出队列，不需要指定此队列大小；     
3. **synchronousQueue：**这个队列比较特殊，它不会保存提交的任务，而是将直接新建一个线程来执行新来的任务。     

>**Tip:** 当ArrayBlockingQueue满时，再有新的任务到来，会抛出异常。


#### 2.2 ThreadPoolExecutor几个重要方法
```java
execute()
shutdown()
shutdownNow()
```
`execute()`接受一个`Runnable`对象，用来向线程池提交一个任务，交由线程池去执行。   
`shutdown()`和`shutdownNow()`都是用来关闭线程池的。他们的区别在于`shutdown()`不会立刻关闭线程池，而等到所有的等待队列的任务执行完才终止，但不会接受新的任务了；`shutdownNow()`会立刻关闭线程池，试图打断正在执行的任务，清空等待队列，返回尚未执行的任务。

#### 2.3 使用ThreadPoolExecutor的例子
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

#### 2.4 Executors类的几个静态创建线程池方法
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

### 3. Java内存模型与线程可见性
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


### 4. 基本并发模型
上面说到volatile关键字可以保证线程可见性，但并不保证线程是同步的，譬如对共享对象的并发修改，volatile无法保证修改是互斥进行的。这时候需要使用互斥操作。
通常意义上讲，多线程并发有三种基本情况：  
1. Read / Read 并发模型  
2. Read / Write 并发模型  
3. Write / Write 并发模型  

对于**Read/Read**情形并不需要使用锁来进行，仅存在对数据的读操作是线程安全的。**Write/Write**情形需要使用互斥锁来解决并发问题。**Read/Write**情形虽然也可以使用互斥锁来解决并发问题，但是使用读写锁则并发效率更高。

### 5. 互斥锁
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

#### 5.1 对方法加synchronized关键字修饰
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

#### 5.2 静态方法和非静态方法加synchronized的区别
对静态方法加synchronized关键字，使得所有该类的实例在不同线程中对该静态方法的调用都是互斥的(相当于**给类加互斥锁**)。而非静态方法加synchronized关键字，只保证该类的同一个实例对非静态方法的访问是互斥的(相当于**给this加互斥锁**)。

#### 5.3 synchronized块
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

#### 5.4 ReentrantLock
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

### 6. 读写锁
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

### 7. 无锁操作
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

### 8. 线程安全的数据结构
`第7节`讲到了一些基本的`Atomic`变量可以用于并发操作，Java支持对一些更高级数据结构的安全访问，譬如`CopyOnWriteArrayList`、`ConcurrentLinkedQueue`和`ConcurrentHashMap`,分别支持数组和映射数据结构进行线程安全地访问。
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
public void not_a_transaction() {
    safeMap.put(someKey1, someValue1);
    //other threads could execute between those two statements
    safeMap.get(someKey2);
}
>```
>在put和get之间是有可能存在cpu被其他线程占用并执行对safeMap操作的情形，如果需要确保这样的操作是事物性的，可以考虑使用`synchronized`。

### 9. 信号量(Semaphore)
信号量是较为低级的多线程同步机制，但是仍然表现出强大的性能，信号量既可以充当互斥锁来解决并发问题，也可以进行同步操作解决多线程协作问题。  
信号量使用两个简单的原语操作，即**wait操作**(也叫P操作/acquire操作)和**post操作**(也叫V操作/signal操作/release操作)来实现对争用资源的互斥访问和多线程间的同步操作。先来讲解下**wait/post原语**操作。

#### 9.1 wait/post原语
```
wait原语
----------
Procedure sem_wait(Semaphore S)
     S = S - 1;
     if(S < 0)
          阻塞进程
```

```
post原语
----------
Procedure sem_post(Semaphore S)
     S = S + 1;
     if(S >= 0)
          唤醒进程
```

#### 9.2 Java semaphore APIs
只列举几个最常用的API
```java
Semaphore(int permits) //构造函数，permits为信号量初值
Semaphore(int permits, boolean fair) //fair为公平参数

public void acquire() throws InterruptedException //wait操作，信号量减1
public void acquire(int n) throws InterruptedException //wait操作，信号量减n

public void release()  //post操作，信号量加1
public void release(int n)  //post操作，信号量加n
```

#### 9.3 信号量充当互斥锁
`第5节`讲到了Java中典型关于互斥锁的应用，这里展示如何使用信号量充当互斥锁。  
**充当互斥锁需要将信号量初始值置为1**
```java
public class Test5 {
    private volatile int count = 0;

    private Semaphore sem_mutex = new Semaphore(1);

    public void increaseCount() {
        for (int i = 0; i < 200; ++i) {
            try {
                sem_mutex.acquire(); //critial area
                count++;
                Log.d("gyw", "count = " + count);
                sem_mutex.release(); //critical area
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```java
final Test5 test5 = new Test5();

ExecutorService executor = Executors.newFixedThreadPool(5);
for (int i = 0; i < 5; ++i) {
    Runnable runnable = new Runnable() {
        @Override
        public void run() {
            test5.increaseCount();
        }
    };
    executor.execute(runnable);
}
executor.shutdown();
```

#### 9.4 用信号量实现线程顺序执行
信号量可以保证若干线程按顺序执行。
**信号量初始值设为0**
```java
final Semaphore sem_sync12 = new Semaphore(0);

Thread thread1 = new Thread(new Runnable() {
    @Override
    public void run() {
        Log.d("gyw", "I am thread1");
        sem_sync12.release();
    }
});

Thread thread2 = new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            sem_sync12.acquire(); //wait thread1 having finished
            Log.d("gyw", "I am thread2");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
});

thread2.start();
thread1.start();
```
程序运行结果是确定的，尽管thread2先thread1启动，打印结果如下：
```
09-16 09:32:15.996  31560-31575/? D/gyw﹕ I am thread1
09-16 09:32:15.996  31560-31574/? D/gyw﹕ I am thread2
```

#### 9.5 无限长队列的生产者-消费者问题
生产者-消费者问题是用一个线程模拟生产者，每次生产一个产品放入队列中；用另一个线程模拟消费者，每次从队列中取走一个产品来消费。这个问题既涉及到**并发控制**，也涉及到**同步控制**。  
因为队列是共享的，所以在多线程都对其进行操作时需要进行并发控制，可以使用并发锁；也因为生产者，消费者这两个线程间存在反馈抑制，因此需要进行同步控制。对于无限长队列来说，不用担心队列溢出的问题，因此生产者可以放肆进行生产，因此不存在对生产者的反馈抑制；而对于消费者，必须是队列中已经存在了产品才可以进行消费，因此存在生产者对消费者的反馈抑制，即生产者生产了一个产品向消费者发出`post`通知，消费者必须首先调用`wait`操作来确认是否存在产品进行消费，否则必须阻塞自己来安静等待生产者的`post`通知的到来。  
如果使用`semaphore`来进行并发控制，只需要设置一个`sem_canConsume`信号量即可，**初值设为0**，含义是一开始的时候并没有产品生产出来。以下是一个`Demo`。
```java
final Queue<Integer> queue = new LinkedList<>();

final Semaphore sem_canConsume = new Semaphore(0);
final Object lock = new Object();

Thread producer = new Thread(new Runnable() {
    @Override
    public void run() {
        for (int i = 0; i < 500; ++i) {
            synchronized (lock) {
                queue.offer(i); //add a product to the queue
                Log.d("gyw", "produced; queueSize = " + queue.size());
            }
            sem_canConsume.release(); //signal that one product is added
        }
    }
});

Thread consumer = new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            for (int i = 0; i < 500; ++i) {
                sem_canConsume.acquire(); //wait that there is at least one product
                synchronized (lock) {
                    queue.poll(); //consume a product
                    Log.d("gyw", "consumed; queueSize = " + queue.size());
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
});

producer.start();
consumer.start();
```

#### 9.6 有限长队列的生产者-消费者问题
和无限长队列同类问题比较起来，由于队列是有限长的，因此存在了消费者对生产者的反馈抑制。如果使用`semaphore`来进行并发控制，除了设置一个`sem_canConsume`信号量来实现生产者对消费者的反馈抑制，**初值设为0**，含义是一开始的时候并没有产品生产出来；还需要设置一个`sem_canProduce`信号量来实现消费者对生产者的反馈抑制，**初值设为队列长度QUEUE_LEN**，含义是最开始的时候有QUEUE_LEN个位置可以供生产者使用。以下是一个`Demo`。
```java
final int QUEUE_LEN = 3;

final Queue<Integer> queue = new LinkedList<>();

final Semaphore sem_canConsume = new Semaphore(0);
final Semaphore sem_canProduce = new Semaphore(QUEUE_LEN);
final Object lock = new Object();

Thread producer = new Thread(new Runnable() {
    @Override
    public void run() {
        for (int i = 0; i < 15; ++i) {
            try {
                sem_canProduce.acquire(); //wait until queue is not full
                synchronized (lock) {
                    queue.offer(i); //add a product to the queue
                    Log.d("gyw", "produced; queueSize = " + queue.size());
                }
                sem_canConsume.release(); //signal that one product is added
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
});

Thread consumer = new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            for (int i = 0; i < 15; ++i) {
                sem_canConsume.acquire(); //wait that there is at least one product
                synchronized (lock) {
                    queue.poll(); //consume a product
                    Log.d("gyw", "consumed; queueSize = " + queue.size());
                }
                sem_canProduce.release();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
});

producer.start();
consumer.start();
```
运行结果如下：
```
09-16 10:22:08.116    1027-1041/? D/gyw﹕ produced; queueSize = 1
09-16 10:22:08.116    1027-1041/? D/gyw﹕ produced; queueSize = 2
09-16 10:22:08.116    1027-1041/? D/gyw﹕ produced; queueSize = 3
09-16 10:22:08.116    1027-1042/? D/gyw﹕ consumed; queueSize = 2
09-16 10:22:08.116    1027-1042/? D/gyw﹕ consumed; queueSize = 1
09-16 10:22:08.116    1027-1042/? D/gyw﹕ consumed; queueSize = 0
09-16 10:22:08.116    1027-1041/? D/gyw﹕ produced; queueSize = 1
09-16 10:22:08.116    1027-1041/? D/gyw﹕ produced; queueSize = 2
09-16 10:22:08.116    1027-1041/? D/gyw﹕ produced; queueSize = 3
09-16 10:22:08.116    1027-1042/? D/gyw﹕ consumed; queueSize = 2
09-16 10:22:08.116    1027-1042/? D/gyw﹕ consumed; queueSize = 1
09-16 10:22:08.116    1027-1041/? D/gyw﹕ produced; queueSize = 2
09-16 10:22:08.116    1027-1041/? D/gyw﹕ produced; queueSize = 3
09-16 10:22:08.116    1027-1042/? D/gyw﹕ consumed; queueSize = 2
09-16 10:22:08.116    1027-1042/? D/gyw﹕ consumed; queueSize = 1
09-16 10:22:08.116    1027-1042/? D/gyw﹕ consumed; queueSize = 0
09-16 10:22:08.116    1027-1041/? D/gyw﹕ produced; queueSize = 1
09-16 10:22:08.116    1027-1041/? D/gyw﹕ produced; queueSize = 2
09-16 10:22:08.116    1027-1041/? D/gyw﹕ produced; queueSize = 3
09-16 10:22:08.116    1027-1042/? D/gyw﹕ consumed; queueSize = 2
09-16 10:22:08.116    1027-1042/? D/gyw﹕ consumed; queueSize = 1
09-16 10:22:08.116    1027-1041/? D/gyw﹕ produced; queueSize = 2
09-16 10:22:08.116    1027-1041/? D/gyw﹕ produced; queueSize = 3
09-16 10:22:08.116    1027-1042/? D/gyw﹕ consumed; queueSize = 2
09-16 10:22:08.116    1027-1042/? D/gyw﹕ consumed; queueSize = 1
09-16 10:22:08.116    1027-1041/? D/gyw﹕ produced; queueSize = 2
09-16 10:22:08.116    1027-1042/? D/gyw﹕ consumed; queueSize = 1
09-16 10:22:08.116    1027-1042/? D/gyw﹕ consumed; queueSize = 0
09-16 10:22:08.116    1027-1041/? D/gyw﹕ produced; queueSize = 1
09-16 10:22:08.116    1027-1042/? D/gyw﹕ consumed; queueSize = 0
```
可以看到，`queueSize`从来没有超过程序中设置的`QUEUE_LEN`值，意味着成功实现了消费者对生产者的反馈抑制；`queueSize`也从来没有是负值或者程序抛出异常，意味着成功实现了生产者对消费者的反馈抑制。
>**Tip:** 简单地使用信号量无法做到**多生产者-多消费者**的同步控制。这个问题将留在后面章节解决。

### 10. 条件变量(Condition)
Java条件变量可以用来进行线程间的同步控制，尤其是在反馈调节上较`semaphore`更为强大。

#### 10.1 条件变量常用APIs
```java
//初始化条件变量
ReentrantLock lock = new ReentrantLock();
Condition cond_canConsume = lock.newCondition();

await() //wait操作 
signal() //post操作
```

以下展示如何使用条件变量解决生产者-消费者问题。

#### 10.2 有限长队列的生产者-消费者问题
先来看看生产者和消费者的代码，这里面包含了如何用锁来解决并发控制，以及如何用条件变量来解决同步控制。
```java
private static final int QUEUE_LEN = 3;
private Queue<Integer> queue = new LinkedList<>();

private ReentrantLock lock = new ReentrantLock();
private Condition cond_canConsume = lock.newCondition();
private Condition cond_canProduce = lock.newCondition();


public void producer() {
     for (int i = 0; i < 5; ++i) {
         lock.lock();

         try {
             while (queue.size() == QUEUE_LEN) { //queue is full
                 Log.d("gyw", "queue is full, waiting...");
                 cond_canProduce.await();
             }

             //queue is not full now
             queue.offer(i); //add a product to the queue
             Log.d("gyw", "produced; queueSize = " + queue.size());
             cond_canConsume.signal(); //signal that a product has been produced

         } catch (InterruptedException e) {
             e.printStackTrace();
         } finally {
             lock.unlock();
         }

     }
 }

 public void consumer() {
     for (int i = 0; i < 5; ++i) {
         lock.lock();

         try {
             while (queue.size() == 0) { //queue is empty
                 Log.d("gyw", "queue is empty, waiting...");
                 cond_canConsume.await();
             }

             //queue is not empty now
             queue.poll(); //consume a product
             Log.d("gyw", "consumed; queueSize = " + queue.size());
             cond_canProduce.signal(); //signal that a product has been produced

         } catch (InterruptedException e) {
             e.printStackTrace();
         } finally {
             lock.unlock();
         }

     }
 }
```
接着来看看启动代码
```java
ExecutorService executor = Executors.newFixedThreadPool(2);

for (int i = 0; i < 1; ++i) { //add one producer thread
    executor.execute(new Runnable() {
        @Override
        public void run() {
            producer();
        }
    });
}

for (int i = 0; i < 1; ++i) { //add one consumer thread
    executor.execute(new Runnable() {
        @Override
        public void run() {
            consumer();
        }
    });
}

executor.shutdown();
```
程序运行结果如下：
```
09-17 08:15:29.105  21003-21016/? D/gyw﹕ produced; queueSize = 1
09-17 08:15:29.105  21003-21016/? D/gyw﹕ produced; queueSize = 2
09-17 08:15:29.105  21003-21016/? D/gyw﹕ produced; queueSize = 3
09-17 08:15:29.105  21003-21016/? D/gyw﹕ queue is full, waiting...
09-17 08:15:29.105  21003-21017/? D/gyw﹕ consumed; queueSize = 2
09-17 08:15:29.105  21003-21017/? D/gyw﹕ consumed; queueSize = 1
09-17 08:15:29.105  21003-21017/? D/gyw﹕ consumed; queueSize = 0
09-17 08:15:29.115  21003-21017/? D/gyw﹕ queue is empty, waiting...
09-17 08:15:29.115  21003-21016/? D/gyw﹕ produced; queueSize = 1
09-17 08:15:29.115  21003-21016/? D/gyw﹕ produced; queueSize = 2
09-17 08:15:29.115  21003-21017/? D/gyw﹕ consumed; queueSize = 1
09-17 08:15:29.115  21003-21017/? D/gyw﹕ consumed; queueSize = 0
```
从结果中可以看出，当队列满的时候，生产者被抑制；当队列空的时候，消费者被抑制。

#### 10.3 有限长队列的多生产者-多消费者问题
`第9节`结尾提到，简单使用信号量无法解决**多生产者-多消费者**的同步控制问题，然而简单使用**条件变量**却很容易解决这个问题，简单到事实上这个问题已经解决了！没错就是在刚刚讲解的`10.2节`，**多生产者-多消费者**和**单生产者-单消费者**问题使用**条件变量**来解决同步控制问题是完全一致的。   
其中生产者、消费者、条件变量申明无需变化，仅仅修改下启动代码就可以，修改也只是添加多了生产者和消费者线程来做演示。
```java
ExecutorService executor = Executors.newFixedThreadPool(4);

for (int i = 0; i < 2; ++i) { //add producer threads
    executor.execute(new Runnable() {
        @Override
        public void run() {
            producer();
        }
    });
}

for (int i = 0; i < 2; ++i) { //add consumer threads
    executor.execute(new Runnable() {
        @Override
        public void run() {
            consumer();
        }
    });
}

executor.shutdown();
```
运行结果如下：
```
09-17 08:20:17.875  21155-21168/? D/gyw﹕ produced; queueSize = 1
09-17 08:20:17.875  21155-21168/? D/gyw﹕ produced; queueSize = 2
09-17 08:20:17.875  21155-21168/? D/gyw﹕ produced; queueSize = 3
09-17 08:20:17.875  21155-21168/? D/gyw﹕ queue is full, waiting...
09-17 08:20:17.875  21155-21169/? D/gyw﹕ queue is full, waiting...
09-17 08:20:17.875  21155-21170/? D/gyw﹕ consumed; queueSize = 2
09-17 08:20:17.875  21155-21170/? D/gyw﹕ consumed; queueSize = 1
09-17 08:20:17.875  21155-21170/? D/gyw﹕ consumed; queueSize = 0
09-17 08:20:17.875  21155-21170/? D/gyw﹕ queue is empty, waiting...
09-17 08:20:17.885  21155-21168/? D/gyw﹕ produced; queueSize = 1
09-17 08:20:17.885  21155-21168/? D/gyw﹕ produced; queueSize = 2
09-17 08:20:17.885  21155-21169/? D/gyw﹕ produced; queueSize = 3
09-17 08:20:17.885  21155-21169/? D/gyw﹕ queue is full, waiting...
09-17 08:20:17.885  21155-21171/? D/gyw﹕ consumed; queueSize = 2
09-17 08:20:17.885  21155-21171/? D/gyw﹕ consumed; queueSize = 1
09-17 08:20:17.885  21155-21171/? D/gyw﹕ consumed; queueSize = 0
09-17 08:20:17.885  21155-21171/? D/gyw﹕ queue is empty, waiting...
09-17 08:20:17.885  21155-21170/? D/gyw﹕ queue is empty, waiting...
09-17 08:20:17.885  21155-21169/? D/gyw﹕ produced; queueSize = 1
09-17 08:20:17.885  21155-21169/? D/gyw﹕ produced; queueSize = 2
09-17 08:20:17.885  21155-21169/? D/gyw﹕ produced; queueSize = 3
09-17 08:20:17.885  21155-21171/? D/gyw﹕ consumed; queueSize = 2
09-17 08:20:17.885  21155-21171/? D/gyw﹕ consumed; queueSize = 1
09-17 08:20:17.885  21155-21170/? D/gyw﹕ consumed; queueSize = 0
09-17 08:20:17.885  21155-21170/? D/gyw﹕ queue is empty, waiting...
09-17 08:20:17.885  21155-21169/? D/gyw﹕ produced; queueSize = 1
09-17 08:20:17.885  21155-21170/? D/gyw﹕ consumed; queueSize = 0
```

#### 10.4 假唤醒问题
如果将10.3节的生产者和消费者代码中的
```java
while (queue.size() == 0) { //queue is empty
    Log.d("gyw", "queue is empty, waiting...");
    cond_canConsume.await();
}
``` 
替换成
```java
if (queue.size() == 0) { //queue is empty
    Log.d("gyw", "queue is empty, waiting...");
    cond_canConsume.await();
}
```
则代码存在假唤醒的潜在问题，所谓假唤醒是由于可能存在多个阻塞线程等待条件变量的信号，并在收到条件变量信号后被唤醒，而程序可能只允许其中一个线程在醒来后获得工作机会，其他的线程被唤醒后发现自己无事可做，只得再次进入阻塞状态。  
当将代码中的`while`替换成`if`后，线程被唤醒后就不会再检查`queue.size()`是否为`0`了，想当然认为自己是被唤醒来获得工作机会的，从而直接执行后面代码，殊不知还有其他线程也被唤醒，而只有其中一个线程能获得工作机会，这将造成潜在错误。

>**Warning:** 使用条件变量时，尽量使用`while`来判断条件是否达到，而不是`if`，从而避免假唤醒带来的错误。

### 11. 阻塞队列(Block Queue)
阻塞队列用途多种多样，在`2.1`节中就提到了Java线程池使用阻塞队列作为线程缓存队列。不只是如此，在多线程同步控制上，阻塞队列也是可以大显身手的。  
事实上，阻塞队列的实现就是使用到了`第10节`讲到的条件变量，使用一个`ReentrantLock`来控制队列的并发，用两个`Condition`变量来控制队列的同步。

#### 11.1 几种常用的阻塞队列
1. **ArrayBlockingQueue：** 基于数组实现的一个阻塞队列，在创建ArrayBlockingQueue对象时必须指定容量大小。另外可以指定公平性，即不保证等待时间最长的队列最优先能够访问队列。
2. **LinkedBlockingQueue：** 基于链表实现的一个阻塞队列，在创建LinkedBlockingQueue对象时如果不指定容量大小，则默认大小为Integer.MAX_VALUE。由于内存是动态分配的，通常情况下无需指定容量大小。
3. **PriorityBlockingQueue：** 基于堆实现的一个阻塞队列，不同于`ArrayBlockingQueue`和`LinkedBlockingQueue`的先进先出性质，`PriorityBlockingQueue`会按照队列元素优先级顺序出队，每次出队的元素都是优先级最高的元素。无需指定容量大小。
4. **DelayQueue：** 基于`PriorityBlockingQueue`，`DelayQueue`中的元素只有当其指定的延迟时间到了，才能够从队列中获取到该元素。无需指定容量大小。

>**Tip:** 以上所说的常用阻塞队列中，前两种是有界队列，后两种是无界队列。

#### 11.2 阻塞队列基本APIs
阻塞队列支持常规队列的基本操作，包括
```
add(E e)：队尾插入元素，插入失败则抛出异常。

remove()：移除队首元素，溢出失败则抛出异常。

offer(E e)：队尾插入元素，如果插入成功，则返回true；如果插入失败，则返回false。

poll()：移除并获取队首元素；若成功，则返回队首元素；否则返回null。

peek()：获取队首元素(并不移出)，若成功，则返回队首元素；否则返回null。
```
对于阻塞队列，还支持阻塞存取方法
```
put(E e)：向队尾插入元素，如果队列满，则等待。

take()：从队首取元素，如果队列为空，则等待。
```
阻塞队列初始化，类似于
```java
BlockingQueue<Integer> queue = new ArrayBlockingQueue<Integer>(3)
BlockingQueue<String> queue = new LinkedBlockingDeque<>()
```

#### 11.3 使用阻塞队列解决多生产者-多消费者同步问题
使用阻塞队列来解决**生产者-消费者**问题大概从编程上来说最简单的了，由于并发控制和同步控制代码已经被阻塞队列自身实现所封装，我们从代码的表面已经看不到并发控制和同步控制了。
消费者和生产者代码如下：
```java
private static final int QUEUE_LEN = 3;
private BlockingQueue<Integer> queue = new ArrayBlockingQueue<Integer>(3);

public void producer() {
    for (int i = 0; i < 5; ++i) {
        try {
            queue.put(i);
            Log.d("gyw", "produced; queueSize = " + queue.size());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

public void consumer() {
    for (int i = 0; i < 5; ++i) {
        try {
            queue.take();
            Log.d("gyw", "consumed; queueSize = " + queue.size());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
启动代码和`10.3节`完全一致，这里就不再累赘列举了。


### 12. 线程特有数据(TSD: Thread-Specific Data)
在上面很多章节中都存在多个线程对共享对象的访问，那么究竟哪些数据是可以在线程间共享的，哪些数据又是每个线程所特有的呢？一般来说，**堆区和静态区内存是可以在线程间进行共享的，而栈区内存是属于每个线程所特有的**。以下详述几种常见情形。
#### 12.1 静态成员变量在不同实例化对象间是线程共享的，而非静态变量在不同实例化对象间是线程特有的
这是由静态成员变量属于类，而非静态成员变量属于对象这一基本特性所决定的。事实上，即使在同一个线程中，情形也是完全一样的，因此这种情形和线程关系并不是很大，唯一需要注意的是对共享变量的多线程并发修改是需要加锁的，下同。
```java
public class Foo {
    private static volatile int sharedInteger = 0; //data shared among different threads
    private int notSharedInteger = 0; //thread specific data

    public synchronized void increase() {
        sharedInteger++;
        notSharedInteger++;
    }

    public void print() {
        Log.d("gyw", "sharedInteger = " + sharedInteger +
                     "; notSharedInteger = " + notSharedInteger);
    }
}
```

```java
final Foo foo1 = new Foo();
final Foo foo2 = new Foo();

Thread thread1 = new Thread(new Runnable() {
 @Override
 public void run() {
     foo1.increase();
 }
});

Thread thread2 = new Thread(new Runnable() {
 @Override
 public void run() {
     foo2.increase();
 }
});

thread1.start();
thread2.start();


try {
	thread1.join();
	thread2.join();
} catch (InterruptedException e) {
	e.printStackTrace();
}

//now thread1 and thread2 finished running
foo1.print();
foo2.print();
```
程序执行结果如下
```
09-18 12:17:51.155  15895-15895/? D/gyw﹕ sharedInteger = 2; notSharedInteger = 1
09-18 12:17:51.155  15895-15895/? D/gyw﹕ sharedInteger = 2; notSharedInteger = 1
```

#### 12.2 非静态成员变量对同一个实例化对象是线程共享的
非静态成员变量虽然在不同实例化对象是拥有自己的一份拷贝，然而对于同一个实例对象来说(譬如函数间)，它是线程共享的。
```java
public class Foo {
    private volatile int sharedInteger = 0; //data shared among different threads

    public synchronized void increase() {
        sharedInteger++;
    }

    public void print() {
        Log.d("gyw", "sharedInteger = " + sharedInteger);
    }
}
```

```java
final Foo foo = new Foo();

 Thread thread1 = new Thread(new Runnable() {
     @Override
     public void run() {
         foo.increase();
     }
 });

 Thread thread2 = new Thread(new Runnable() {
     @Override
     public void run() {
         foo.increase();
     }
 });

 thread1.start();
 thread2.start();


 try {
     thread1.join();
     thread2.join();
 } catch (InterruptedException e) {
     e.printStackTrace();
 }

 //now thread1 and thread2 finished running
 foo.print();
```

程序执行结果如下
```
09-18 12:26:18.905  16071-16071/? D/gyw﹕ sharedInteger = 2
```

#### 12.3 局部变量是线程特有的
局部变量位于栈区，因此是线程特有的。
```java
public class Foo {

    public synchronized void increase() {
        int localVariable = 0;
        localVariable++;
        Log.d("gyw", "localVariable = " + localVariable);
    }

}
```

```java
final Foo foo = new Foo();

Thread thread1 = new Thread(new Runnable() {
    @Override
    public void run() {
        foo.increase();
    }
});

Thread thread2 = new Thread(new Runnable() {
    @Override
    public void run() {
        foo.increase();
    }
});

thread1.start();
thread2.start();
```
运行结果如下：
```
09-18 12:36:27.295  16281-16294/? D/gyw﹕ localVariable = 1
09-18 12:36:27.295  16281-16295/? D/gyw﹕ localVariable = 1
```

#### 12.4 ThreadLocal对象
`12.1节`说到静态成员变量在不同实例化对象间是线程共享的(当然对同一个实例化对象更是线程共享了)，`12.1节`非静态成员变量对同一个实例化对象是线程共享的。事实上使用`ThreadLocal`就可以将这些共享的成员变量变成线程特有的(也成为线程私有的)。`ThreadLocal`提供以下常见方法：
```java
public T get();  //获取当前线程的ThreadLocal绑定值
public void set(T value); //设置当前线程的ThreadLocal绑定值
public void remove(); //删除当前线程ThreadLocal绑定的值
protected T initialValue() //可以通过子类继承来初始化ThreadLocal变量值
```

以下举一个使用`ThreadLocal`的例子，通过这个例子就可以体会`ThreadLocal`如何将共享的成员变量线程特有化。
```java
public class Foo {
    private static ThreadLocal<Integer> threadLocalInteger = new ThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return Integer.valueOf(0);
        }
    };

    public synchronized void increase() {
        threadLocalInteger.set(threadLocalInteger.get() + 1);
        Log.d("gyw", "threadLocalInteger = " + threadLocalInteger.get());
    }

}
```

```java
final Foo foo = new Foo();

Thread thread1 = new Thread(new Runnable() {
    @Override
    public void run() {
        foo.increase();
    }
});

Thread thread2 = new Thread(new Runnable() {
    @Override
    public void run() {
        foo.increase();
    }
});

thread1.start();
thread2.start();
```
运行结果如下
```
09-18 21:35:39.695  24036-24049/? D/gyw﹕ threadLocalInteger = 1
09-18 21:35:39.695  24036-24050/? D/gyw﹕ threadLocalInteger = 1
```
运行结果清楚地表示出，尽管`threadLocalInteger`是一个静态变量，然而由于它是一个`ThreadLocal对象`，因而实现了线程特有化而成为线程特有数据。

>**Resource:** 关于`ThreadLocal`更多的知识，可以访问以下网址：
[http://qifuguang.me/2015/09/02/[Java%E5%B9%B6%E5%8F%91%E5%8C%85%E5%AD%A6%E4%B9%A0%E4%B8%83]%E8%A7%A3%E5%AF%86ThreadLocal/](http://qifuguang.me/2015/09/02/[Java%E5%B9%B6%E5%8F%91%E5%8C%85%E5%AD%A6%E4%B9%A0%E4%B8%83]%E8%A7%A3%E5%AF%86ThreadLocal/)

### 13. 主线程与工作线程之间同步通信
`1.1节`讲述了Java两种显式创建线程的方法，`第2节`讲述了如何使用线程池来在新线程中执行`Runnable`任务。当主线程不需要了解工作线程执行任务的情况，譬如结果，那么`第2节`的关于线程池的知识就已经足够了。然而，如果我们希望能在主线程中当工作线程结束时知道工作线程的任务结果，又能怎么办呢？本节将讲述使用`Callable`对象配合`Future`来实现这一点。  

>**Tip:** 通过共享变量也可以做到这一点，但是一方面这破坏了封装性，另一方面在多线程中使用共享变量容易出错。使用`Callable`对象配合`Future`可以简单方便地实现主线程和工作线程之间的通信。

在`第2节`介绍`ExecutorService`的时候，我们讲述了可以利用`execute(Runnable r)`方法来使用线程池执行`Runnable`任务。事实上，`ExecutorService`还有一个`Future<T> submit(Callable<T> task)`方法也可以利用线程池执行新线程任务，`T`是任务执行完成返回结果的类型。  

#### 13.1 Callable对象
可以看到`submit()`方法可以接受一个`Callable`对象，类似于`Runnable`可以在其`run()`方法里写进那些需要在新线程中执行的代码，也可以在`Callable`的`call()`方法中写进那些需要在新线程中执行的代码。唯一不同的是`run()`方法没有返回值，而`call()`方法可以返回泛型类型为`T`的结果对象。
```java
//define callable
Callable<Integer> callable = new Callable<Integer>() {
    @Override
    public Integer call() throws Exception {
        LogUtil.logThreadId("call() is executed in worker thread");
        return 1;
    }
};
```

#### 13.2 Future
另外我们可以看到`submit()`返回了一个`Future<T>`类型的对象，我们可以通过返回的这个`Future`对象来在主线程中获取工作线程的执行结果，事实上，通过`Future`可以允许我们做得更多。首先来看看`Future`接口。
```java
public interface Future<V> {
    //取消任务
    boolean cancel(boolean mayInterruptIfRunning);

    //获取任务是否已经被取消
    boolean isCancelled();

    //获取任务是否已完成
    boolean isDone();

    //获取任务执行结果，阻塞等待结果
    V get() throws InterruptedException, ExecutionException;

	//获取任务执行结果，阻塞等待结果，若超时则get()直接返回null
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
>**Tip:** `get()`方法是一个**阻塞同步方法(Blocked Synchronous Method)**，这意味着在主线程中对该方法的调用会使得主线程阻塞来等待工作线程返回结果，如果工作线程迟迟不返回结果，则主线程会一直被阻塞。

这里举一个简单的例子，该例子将任务使用线程池分配给工作线程去做，然后主线程调用`get()`方法等待工作线程完成并返回执行结果。其中`callable`对象定义见`13.1`节。
```java
ExecutorService executor = Executors.newFixedThreadPool(1);
Future<Integer> result = executor.submit(callable);

try {
    //main thread is blocked to wait for result from worker thread
    LogUtil.logThreadId("result = " + result.get());
} catch (InterruptedException e) {
    e.printStackTrace();
} catch (ExecutionException e) {
    e.printStackTrace();
}

//this won't be logged until the result is returned from worker thread
LogUtil.logThreadId("main thread done all works");
```
程序运行结果如下
```
09-18 22:04:06.355  24500-24513/? I/gyw﹕ TID:8451; call() is executed in worker thread
09-18 22:04:06.355  24500-24500/? I/gyw﹕ TID:1; result = 1
09-18 22:04:06.355  24500-24500/? I/gyw﹕ TID:1; main thread done all works
```

#### 13.3 利用多线程计算Fibonacci数列
先来看下单线程计算Fibonacci数列的情形，最简单的版本是可以利用递归去实现，从算法角度来讲就是**分治法(Divide and Conquer)**。由于计算数列算法本身非常简单，直接给出递归实现的代码。
```java
public long computeRecursively(int n) {
    if (n > 1) {
        return computeRecursively(n-1) + computeRecursively(n-2);
    }
    return n;
}
```
事实上容易知道上面计算方法的时间复杂度是**指数级**的，这意味着当n较大的时候，计算时间将非常恐怖。为了直观对比各算法的性能，使用下面一小段代码才测量cpu执行时间并在驱动代码中将n设置为30。
```java
long startTime = Debug.threadCpuTimeNanos();
long result = computeRecursively(30);
long duration_ns = Debug.threadCpuTimeNanos() - startTime;
long duration_ms = duration_ns / 1000000;
Log.d("gyw", "result = " + result + "; duration = " + duration_ns + "ns = " + duration_ms + "ms");
```
在作者的Android机器上，运行结果如下：
```
09-19 08:03:34.995  32488-32488/? D/gyw﹕ result = 832040; duration = 357233029ns = 357ms
```
可以看到当计算第30项值的时候，机器就已经显得步履维艰了。考虑一个不那么困难的改进方案，在递归过程中计算`第n项`数列值的时候，事实上递归调用了计算`第n-1项`和`第n-2项`的值；而譬如计算`第n-1项`值时又会递归计算`第n-2项`值，这造成了大量的重复计算，事实上前面算法效率低下的症结就是在这个地方，cpu将计算时间几乎完全耗费在了一次又一次重复计算递归项上了。一个简单的改进就是在计算中将每次计算的值记录下来，在需要的时候取用，如果存取是高效的，那么必然效率会有质的跃升。这样改进后的算法就是**动态规划(Dynamic programming)**了，具体地说是使用了**备忘录(memo)**技术的动态规划算法。

>**Tip:** 也可以使用从第一项至最后一项**迭代**计算技术来实现动态规划算法，而且效率通常会更高一点。这里是为了算法横向改进对比的方便采用**备忘录**技术。当然两种技术细节对**时间复杂度的阶(Order)**没有任何影响，只微弱影响时间复杂度的常数因子。

下面就利用Android中的`SparseArray`来作为缓存数组实现**备忘录**技术，代码如下：
```java
    private SparseArray<Long> cache = new SparseArray<>();

    public long computeRecursively(int n) {
        if (n > 1) {
            Long fac1 = cache.get(n-1);
            Long fac2 = cache.get(n-2);
            
            if (fac1 == null) {
                fac1 = computeRecursively(n-1);
                cache.put(n-1, fac1);
            }
            
            if (fac2 == null) {
                fac2 = computeRecursively(n-2);
                cache.put(n-2, fac2);
            }

            return fac1 + fac2;
        }
        return n;
    }
```

>**Tip:** 也可以使用Java中的`HashMap`来作为缓存，这样通用性会更好一点，Android的`SparseArray`相对效率更高一些。

同样条件下让我们来看看Android机器上的运行结果吧。
```
09-19 08:13:26.205  32678-32678/? D/gyw﹕ result = 832040; duration = 297742ns = 0ms
```
对我们设定的条件下，执行效率提高了**1200倍**，**哇！令人印象深刻啊！**。从算法时间复杂度上来说，第二种算法是**线性**的，即**O(n)**，难怪会如此高效。  
那么是否可以进一步榨干cpu的运算性能呢？对于多核cpu，也许我们可以使用多线程来进一步提高性能。当然这里说的是也许，如果我们可以将并行的计算置于不同的cpu核心上去运行，那也许有可能我们就提高了整体的性能。在试图实现我们这个想法，先把刚才那个令人印象深刻的迭代算法做些许改进，之所以需要改进是因为当计算数列的项n太大时(譬如`n = 100`)，使用`Long`来存储会出现溢出而无法得到正确的结果，这里简单使用`BigInteger`来替代`Long`。改进后的代码如下：
```java
public BigInteger computeRecursively(int n) {
    if (n > 1) {
        BigInteger fac1 = cache.get(n-1);
        BigInteger fac2 = cache.get(n-2);
        
        if (fac1 == null) {
            fac1 = computeRecursively(n-1);
            cache.put(n-1, fac1);
        }
        
        if (fac2 == null) {
            fac2 = computeRecursively(n-2);
            cache.put(n-2, fac2);
        }
        
        return fac1.add(fac2);
    }
    return (n == 0) ? BigInteger.ZERO : BigInteger.ONE;
}
```
一个可能的利用多线程来改进效率来自于这样的事实，计算`第n项`数列值的时候，事实上需要计算`第n-1项`和`第n-2项`的值，可以将`第n-1项`和`第n-2项`放到不同的线程来进行计算(当然只有放到不同核心才是真正的多核并发)。

>**Tip:** 将并发计算分配到多线程并不一定意味着cpu会将其调度到不同核心上去进行计算，这取决于操作系统的调度算法。一个好的消息就是对于Android来说，这样的核心调度算法是优化的，而且随着版本提高还在不断优化中，当然操作系统也需要照顾到别的程序的运行。同一时间并不只存在你所写的那一个程序在执行。

基于这样的想法，可以写出如下的多线程计算Fibonacci数列的代码
```java
private ConcurrentHashMap<Integer, BigInteger> cache = new ConcurrentHashMap<Integer, BigInteger>();

public BigInteger computeRecursively(int n) {
   //The same as before
}

private static final int CPU_NUM = Runtime.getRuntime().availableProcessors();
private ExecutorService executorService = Executors.newFixedThreadPool(CPU_NUM + 1);

public BigInteger computeRecursivelyWithTwoThreads(final int n) {
    Callable<BigInteger> callable = new Callable<BigInteger>() {
        @Override
        public BigInteger call() throws Exception {
            return computeRecursively(n-1);
        }
    };
    
    Future<BigInteger> fac1 = executorService.submit(callable); //submit first task ASAP
    
    callable = new Callable<BigInteger>() {
        @Override
        public BigInteger call() throws Exception {
            return computeRecursively(n-2);
        }
    };
    
    Future<BigInteger> fac2 = executorService.submit(callable); //submit second task
    
    try {
        //combine results
        return fac1.get().add(fac2.get());
    } catch (InterruptedException | ExecutionException e) {
        e.printStackTrace();
        return BigInteger.valueOf(-1); //if fails
    }
}
```
由于多个线程会对`cache`值进行访问，因此存在并发访问问题，这里使用`ConcurrentHashMap`来解决`cache`的并发访问问题，使得`cache`是线程安全的。

>**Tip:** 对于多线程，设置多少个线程数合适的问题，这个依赖具体情况。一个指导原则是  
>1. 对CPU密集任务，设置线程数为 **CPU_NUM + 1**；  
>2. 对于I/O密集任务， 设置线程数为 **2 * CPU_NUM**。  

到了这里，似乎一切都很完美，然而还差一个问题，使用多线程写出来的程序真的效率比单线程高很多么？不得不泼一盆凉水，尽管代码变复杂了，愿望也是美好的，然而事实往往不尽如人意。在两核cpu上执行效率非但往往达不到单线程单核cpu的2倍，**很多情况下比单线程效率还要低下**！这一方面由于线程创建、调度、销毁需要费事费力，cpu调度算法也要消耗计算资源，更因为很多时候由于并发、同步等问题使得线程不得不阻塞等待，这些都导致了多线程效率的低下。

>**Tip:** 这样的事实再次告诫我们，只有在需要优化的情况下才考虑对效率的优化，并且**优化的首要步骤是改进算法而不是改进实现**，通常项目进度、异常处理、代码的可维护性是更重要的因子，**优化需要抱着务实的态度**。

当然我们也不应该过分悲观，很多时候使用多线程确实可以提高程序的效率，实践表明，那些可以并发执行的任务，即**不需要大量同步需求的并行执行任务是适合多线程的**，而且通常来说由于各种开销缘故，**只有当问题规模较大的时候才适合进行多线程**，当问题规模小的时候往往应该调用单线程版本，后者的考量类似于快速排序在规模较小时候应该直接调用简单排序算法来实现。  
最后还是让我们来看一下我们使用多线程改进的算法与单线程的算法在实际运行中的效率对比吧。驱动和测量时间的代码如下：
```java
final int N = 30; //we test 30, 128 and 400

BigInteger result = BigInteger.valueOf(-2);

long startTime = Debug.threadCpuTimeNanos();
for (int i = 0; i < 50; ++i) {
   result = computeRecursively(N);
}
long duration_ns = Debug.threadCpuTimeNanos() - startTime;
long duration_ms = duration_ns / 1000000;
Log.d("gyw", "result = " + result + " computeRecursively; duration = " + duration_ns + "ns = " + duration_ms + "ms");

startTime = Debug.threadCpuTimeNanos();
for (int i = 0; i < 50; ++i) {
    result = computeRecursivelyWithTwoThreads(N);
}
duration_ns = Debug.threadCpuTimeNanos() - startTime;
duration_ms = duration_ns / 1000000;
Log.d("gyw", "result = " + result + " computeRecursivelyWithTwoThreads; duration = " + duration_ns + "ns = " + duration_ms + "ms");
```
我们改变问题规模N，测试当N分别为30，128和400情况下，程序在机器上执行的结果(`CPU_NUM = 4`)。
```
//N = 30
09-19 09:58:25.205    4381-4381/? D/gyw﹕ result = 832040 computeRecursively; duration = 1874932ns = 1ms
09-19 09:58:25.215    4381-4381/? D/gyw﹕ result = 832040 computeRecursivelyWithTwoThreads; duration = 5444422ns = 5ms

//N = 128
09-19 10:00:46.655    4473-4473/? D/gyw﹕ result = 251728825683549488150424261 computeRecursively; duration = 5750731ns = 5ms
09-19 10:00:46.665    4473-4473/? D/gyw﹕ result = 251728825683549488150424261 computeRecursivelyWithTwoThreads; duration = 4643587ns = 4ms

//N = 400
09-19 10:01:02.375    4539-4539/? D/gyw﹕ result = 176023680645013966468226945392411250770384383304492191886725992896575345044216019675 computeRecursively; duration = 17544497ns = 17ms
09-19 10:01:02.385    4539-4539/? D/gyw﹕ result = 176023680645013966468226945392411250770384383304492191886725992896575345044216019675 computeRecursivelyWithTwoThreads; duration = 5307339ns = 5ms
```
从测试的结果中我们可以看出，当问题规模较小的时候，单线程的执行效率更高；当问题规模趋于增大的时候，多线程的优势逐渐显现，知道`N = 400`的时候，多线程执行效率高出了单线程的3倍还多。

### 14. 主线程与工作线程之间异步通信
上一节说到了主线程与工作线程之间可以通过`Callable`和`Future`进行同步通信，主线程会阻塞自己等待工作线程的执行结果。然而这样的同步通信在很多情况下是不合适的，如果主线程负载其他重要工作而不得挂起等待工作线程结束以后才继续执行，这一方面影响了主线程执行效率，更会影响到其他在主线程中的重要工作的执行。在Android中，这一点尤为明显，主线程主要负责UI工作，如果主线程被阻塞等待，则UI会失去响应，这将严重影响用户体验。更何况Android为了强化这一点，强制要求在`Activity`中主线程不能被持有超过5s，`Service`中主线程不能被持有超过10s，一旦超过，Android系统就会抛出`ANR`异常。  
异步通信就可以解决这样的问题，使得主线程执行效率得到提高并且可以始终响应譬如UI事件这样更重要的工作。工作线程在完成工作后通知主线程工作已完成并将结果以某种方式告知主线程，主线程拿到结果以后可以用来譬如刷新UI显示。

>**Tip:** Android规定对UI的操作必须在主线程内完成，因此有时候主线程也会被称为**UI线程**。

下面就来看一下Android中封装的`AsyncTask`类，这个类是多线程异步通信的典范。
#### 14.1 AsyncTask类
`AsyncTask`类强化了这样的封装：将需要处理的事情方法到新建的工作线程中去做，当工作完成后将结果发送给主线程，在主线程中刷新UI。`AsyncTask`类会自动创建工作线程，发送结果也是封装好的，使用者只需要覆写几个方法就可以了，现在具体来看一看这几个方法。
```java
public class MyAsyncTask extends AsyncTask<Integer, Void, BigInteger> {

    @Override
    protected BigInteger doInBackground(Integer... integers) {
        //Work in worker thread
        //publishProgress() method can be applied here to inform execution progress
        return null;
    }

    @Override
    protected void onPreExecute() {
        super.onPreExecute();
        //Work in main thread
        //This method is called before worker thread begins
    }

    @Override
    protected void onPostExecute(BigInteger bigInteger) {
        super.onPostExecute(bigInteger);
        //Work in main thread
        //This method is called after worker thread finishes
        //Para is the worker thread execution result
        //UI is refreshed here
    }

    @Override
    protected void onProgressUpdate(Void... values) {
        super.onProgressUpdate(values);
        //Work in main thread
        //Got called when publishProgress() is invoked
    }
}
```
`AsyncTask`类需要指定泛型参数类型，分别对应于**工作线程传入参数类型**(`doInBackground()`传入参数)、**工作线程执行中刷新参数值**(`onProgressUpdate()`参数值)和**工作线程返回结果类型**(`doInBackground()`返回值和`onPostExecute()`参数)。  
`doInBackground()`里面就是需要在工作线程中执行的代码，并将结果返回给主线程，从而可以在`onPostExecute()`中将执行结果刷新到UI，事实上除了`doInBackground()`方法执行在工作线程，其他几个方法都执行在主线程上。   
下面就给出一个使用`AsyncTask`计算Fibonacci数列并将结果刷新到`TextView`上显示的简单例子。
```java
public class MyAsyncTask extends AsyncTask<Integer, Void, BigInteger> {

    private TextView mTextView;

    public MyAsyncTask(TextView mTextView) {
        this.mTextView = mTextView;
    }

    @Override
    protected BigInteger doInBackground(Integer... n) {
        Log.d("gyw", "MyAsyncTask::doInBackground()");
        return computeRecursively(n[0]);  //See Chapter 13.3
    }

    @Override
    protected void onPreExecute() {
        super.onPreExecute();
        Log.d("gyw", "MyAsyncTask::onPreExecute()");
    }

    @Override
    protected void onPostExecute(BigInteger result) {
        super.onPostExecute(result);
        Log.d("gyw", "MyAsyncTask::onPostExecute(); result = " + result);
        mTextView.setText(result.toString()); //update textView
    }
}
```

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        TextView textView = (TextView) findViewById(R.id.display);

        MyAsyncTask myAsyncTask = new MyAsyncTask(textView);
        myAsyncTask.execute(30);
    }
    
}
```
顺便看下`LogCat`日志
```
09-20 09:56:38.754  23657-23657/? I/gyw﹕ TID:1; MyAsyncTask::onPreExecute()
09-20 09:56:38.754  23657-23670/? I/gyw﹕ TID:8865; MyAsyncTask::doInBackground()
09-20 09:56:38.764  23657-23657/? I/gyw﹕ TID:1; MyAsyncTask::onPostExecute(); result = 832040
```

>**Warning:** 需要指出的是给出的`Demo`例子当遇到`Configuration Change`的时候(譬如屏幕旋转)是有bug的，潜在的bug包括`onCreate()`会重新被调用从而使得第一次的任务没有完成或者取消的情况下，又执行一次同样的任务。另外，更大的问题在于`MyAsyncTask`的中的`mTextView`将仍然指向第一次创建的`Activity`，这使得刷新UI存在潜在问题，另外也造成了第一次创建的`Activity`内存泄露了。这些问题的解决并非本文的重点，也超出本文的范畴，这里不再赘述。

#### 14.2 Handler、Looper与消息队列(1)
`AsyncTask`类虽然是多线程异步通信的典范，其最大优点便是使用简单，但是过多的封装也造成灵活性的下降，另外当存在多个工作线程时代码也显得非常臃肿。事实上`AysncTask`实现中就是使用了`Handler`、`Looper`和消息队列。为了厘清`Handler`、`Looper`和消息队列之间的关系，先来看一个关系图。

![](https://lh3.googleusercontent.com/-8TW7OiMn5g4/Vf4ayHT9uoI/AAAAAAAAA9w/RC5xaQXDHC0/s640-Ic42/192914506.png)

这个关系图清晰地展示了主线程和工作线程如何通过`Handler`、`Looper`和消息队列进行通信的，当然这里展示的仅仅是工作线程向主线程发送消息，而这正是`AsyncTask`所抽象的东西。  
每一个`Looper`会直接维护一个消息队列并启动消息队列循环，即不断取出消息来消化。图中的`Looper`是主线程的`Looper`，而消息是不断发往主线程的消息队列中等待主线程处理的。而工作线程正是通过主线程的`Handler`来将消息发往主线程的消息队列。   
下面就用一个`Demo`来实现与`14.1节`用`AsyncTask`来实现的完全一样的功能，只不过这一次，我们自己来创建工作线程去执行任务，执行完任务我们自己使用`Handler`将消息发送回主线程，并让主线程去刷新UI。
```java
public class MyThreadTask extends Thread {

    public static final int MYTHREADTASK_RESULT = 0X001;
    
    private int n;
    private Handler mainThreadHandler = null;

    public MyThreadTask(Handler mainThreadHandler, int n) {
        this.mainThreadHandler = mainThreadHandler;
        this.n = n;
    }

    @Override
    public void run() {
        super.run();
        LogUtil.logThreadId("MyThreadTask::run()");
        BigInteger result = computeRecursively(n);
        mainThreadHandler.sendMessage(mainThreadHandler.obtainMessage(MYTHREADTASK_RESULT, result));
    }
}
```

```java
public class MainActivity extends AppCompatActivity {

    private static class SimpleHandler extends Handler {
        // If outter-context will be referred; consider using weak reference to avoid context memory leak
        private TextView mtextView;

        public SimpleHandler(TextView mtextView) {
            this.mtextView = mtextView;
        }

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            switch (msg.what) {
                case MyThreadTask.MYTHREADTASK_RESULT:
                    //digest result
                    BigInteger result = (BigInteger) msg.obj;
                    LogUtil.logThreadId("message recieved; result = " + result.toString());
                    mtextView.setText(result.toString());
                    break;
            }
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        TextView textView = (TextView) findViewById(R.id.display);

        //mainThreadHandler is created in main thread and binds to main thread looper
        //Handler and Looper belong to main thread
        SimpleHandler mainThreadHandler = new SimpleHandler(textView);

        //deploy task to run in new thread
        MyThreadTask myThreadTask = new MyThreadTask(mainThreadHandler, 30);
        myThreadTask.start();
    }
}
```
来看看`Logcat`输出吧
```
09-20 11:01:39.514  24734-24852/? I/gyw﹕ TID:8907; MyThreadTask::run()
09-20 11:01:39.564  24734-24734/? I/gyw﹕ TID:1; message recieved; result = 832040
```
结果正如我们的预期。在`14.1节`的最后我们提到了`MyAsyncTask`的中的`mTextView`将仍然指向第一次创建的Activity，造成了第一次创建的Activity内存泄露的问题。事实上，在本节这个例子中我们已经避免了这样的泄露。注意到当Activity重建的时候`textView`重新指向新创建的`Activity`的`TextView`，尔后`mainThreadHandler`也重新实例化，新的`mainThreadHandler`包含的是新的`textView`，从而使得UI刷新正确，并且不阻止GC之前那个旧的`Activity`。

>**Warning:** 在代码注释中我们提到如果需要包含指向外围`Activity`的`Context`时，请使用弱引用，否则会造成泄漏，这是因为有可能存在延迟消息，使得`Handler`寿命远远长于`Activity`，从而可能造成虽然`Activity`已经被意图上销毁，但由于`Handler`所包含的`Context`引用从而造成`Context`对应的资源无法被释放。而事实上，`Handler`手上持有的`Context`意图上已经是没有意义的了。

#### 14.3 Handler、Looper与消息队列(2)
`14.2节`讲述了如何将消息从工作线程发往主线程并由主线程消化消息。其中我们并没有为主线程的`Handler`指定一个`Looper`对象(包括`Looper`维护的主线程消息队列)，这是由于主线程已经存在了自己的消息队列，因此不需要我们去显式去创建、绑定。然而，对于我们自己的创建的线程，则不存在这样默认的消息队列了，需要我们手动去设置。正因为如此，从主线程将消息发送到工作线程的时候，除了之前那些工作以外，还多出设置消息队列的工作。  
这里我们使用`HandlerThread`类来帮助我们，这个类继承自`Thread`类，除了使得我们能获得一个新的线程，还使得我们可以使用`getLooper()`方法获得对应于新建线程的`Looper`，留给我们需要完成的工作是需要将这个`Looper`与`Handler`进行绑定，这个工作可以在`start()`同步方法中完成，这个同步方法仍然工作在主线程上。完成以后使得我们拥有一个工作在工作线程的`Looper`维护一个消息队列，并且主线程(或者其他线程)可以通过`Handler`来将消息发送到工作线程的消息队列中进而被消化。  
下面的例子展示了当工作线程维护自己的`Looper`和消息队列并绑定到`Handler`上，主线程通过这个`Handler`将消息发送到工作线程，从消息中获取参数并在工作线程执行工作。
```java
public class MyHandlerThread extends HandlerThread {

    public static final int INITIAL_WHAT = 0X001;

    private SimpleHandler workerThreadHandler = null;

    private static class SimpleHandler extends Handler {

        public SimpleHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            switch (msg.what) {
                case INITIAL_WHAT:
                    //This works in the worker thread now!
                    int number = msg.arg1;
                    BigInteger result = computeRecursively(number); //see chapter 13.3
                    LogUtil.logThreadId("result = " + result);
                    break;
            }
        }
    }

    public MyHandlerThread(String name) {
        super(name);
    }

    @Override
    public synchronized void start() {
        //This still works in the main thread
        super.start();
        workerThreadHandler = new SimpleHandler(getLooper()); //bind to worker thread looper
        LogUtil.logThreadId("start()");
    }

    public Handler getHandler() {
        return workerThreadHandler;
    }
}
```
`getLooper()`方法会一直阻塞到Looper对象初始化完成，然后传给`Handler`。
```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        
        MyHandlerThread myHandlerThread = new MyHandlerThread("gyw");
        myHandlerThread.start();

        Handler workerThreadHandler = myHandlerThread.getHandler();
        workerThreadHandler.sendMessage(
                workerThreadHandler.obtainMessage(MyHandlerThread.INITIAL_WHAT,
                        30, 0));
    }

}
```
程序运行结果如下
```
09-20 22:13:22.913    2769-2769/? I/gyw﹕ TID:1; start()
09-20 22:13:22.913    2769-2782/? I/gyw﹕ TID:9066; result = 832040
```
