### Android多线程
使用多线程的好处通常包括提高了代码的效率，充分利用计算资源，减少系统响应时间等。譬如可以将问题划分为子问题并将子问题交给不同的线程进行处理，如果这些线程需要共享一些争用资源，那么通常对这些争用资源的访问(读或者写操作)是需要进行**互斥操作**的（解决并发问题），如果这些线程在某些时候需要以一定次序进行，那么则需要进行**同步操作**（解决协作问题）。互斥操作和同步操作一般都称为**多线程同步问题**。   
Android为了保证系统对用户保持高响应性，更是强制规定了在Activity的主线程中的操作不能超过5秒，Service的主线程中的操作不能超过10秒，否则会抛出`ANR异常`。这使得我们必须要将耗时操作转移到工作线程中去。
>**Tip:** Android甚至都不允许在主线程中进行网络操作，当然这是可以理解的，因为网络操作的耗时往往是无法预期的，这取决于网络状况。

#### 1. 线程基本操作
这里介绍几个最常用的线程基本操作，包括创建线程、中断线程、休眠线程、`join()`、阻塞/唤醒线程等方法。  
##### 1.1 创建线程
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

##### 1.2 中断线程
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

##### 1.3 sleep和yield方法
`sleep()`的调用可以使线程进入睡眠状态，会交出CPU给别的线程去使用。在`1.2节`的例子中已经提前使用了`sleep()`方法。值得注意的是，调用`sleep()`方法并不会释放已经持有的锁。  
`yield()`类似于`sleep()`，也会交出CPU给别的线程并且也不会释放已经持有的锁，不同的是`yield()`方法并不指出线程应该睡眠多长时间，仅仅是让出CPU以使得同一优先级的线程有机会获得CPU。

>**Tip:** 关于什么是锁？锁在多线程中有什么具体用途用法？如果尚且不清楚或者不理解，没有关系，请继续往后阅读，后面会详细叙述。

##### 1.4 join方法
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

##### 1.5 wait/notify方法
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

##### 1.6 线程状态转换模型图
本节最后以一个较详细的线程状态转换模型图来做结尾，以加深对线程状态及状态之间转移关系的理解。

![](https://lh3.googleusercontent.com/--CAG2wf6fIw/Vft8ooslvKI/AAAAAAAAA9k/JKP37-9myyw/s720-Ic42/061046391107893.jpg)

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
`workQueue`典型类型有以下三种：  
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

#### 9. 信号量(Semaphore)
信号量是较为低级的多线程同步机制，但是仍然表现出强大的性能，信号量既可以充当互斥锁来解决并发问题，也可以进行同步操作解决多线程协作问题。  
信号量使用两个简单的原语操作，即**wait操作**(也叫P操作/acquire操作)和**post操作**(也叫V操作/signal操作/release操作)来实现对争用资源的互斥访问和多线程间的同步操作。先来讲解下**wait/post原语**操作。

##### 9.1 wait/post原语
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

##### 9.2 Java semaphore APIs
只列举几个最常用的API
```java
Semaphore(int permits) //构造函数，permits为信号量初值
Semaphore(int permits, boolean fair) //fair为公平参数

public void acquire() throws InterruptedException //wait操作，信号量减1
public void acquire(int n) throws InterruptedException //wait操作，信号量减n

public void release()  //post操作，信号量加1
public void release(int n)  //post操作，信号量加n
```

##### 9.3 信号量充当互斥锁
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

##### 9.4 用信号量实现线程顺序执行
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

##### 9.5 无限长队列的生产者-消费者问题
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

##### 9.6 有限长队列的生产者-消费者问题
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

#### 10. 条件变量(Condition)
Java条件变量可以用来进行线程间的同步控制，尤其是在反馈调节上较`semaphore`更为强大。

##### 10.1 条件变量常用APIs
```java
//初始化条件变量
ReentrantLock lock = new ReentrantLock();
Condition cond_canConsume = lock.newCondition();

await() //wait操作 
signal() //post操作
```

以下展示如何使用条件变量解决生产者-消费者问题。

##### 10.2 有限长队列的生产者-消费者问题
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
```
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

##### 10.3 有限长队列的多生产者-多消费者问题
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

##### 10.4 假唤醒问题
如果将10.3节的生产者和消费者代码中的
```
while (queue.size() == 0) { //queue is empty
    Log.d("gyw", "queue is empty, waiting...");
    cond_canConsume.await();
}
``` 
替换成
```
if (queue.size() == 0) { //queue is empty
    Log.d("gyw", "queue is empty, waiting...");
    cond_canConsume.await();
}
```
则代码存在假唤醒的潜在问题，所谓假唤醒是由于可能存在多个阻塞线程等待条件变量的信号，并在收到条件变量信号后被唤醒，而程序可能只允许其中一个线程在醒来后获得工作机会，其他的线程被唤醒后发现自己无事可做，只得再次进入阻塞状态。  
当将代码中的`while`替换成`if`后，线程被唤醒后就不会再检查`queue.size()`是否为`0`了，想当然认为自己是被唤醒来获得工作机会的，从而直接执行后面代码，殊不知还有其他线程也被唤醒，而只有其中一个线程能获得工作机会，这将造成潜在错误。

>**Warning:** 使用条件变量时，尽量使用`while`来判断条件是否达到，而不是`if`，从而避免假唤醒带来的错误。

#### 11. 阻塞队列(Block Queue)
阻塞队列用途多种多样，在`2.1`节中就提到了Java线程池使用阻塞队列作为线程缓存队列。不只是如此，在多线程同步控制上，阻塞队列也是可以大显身手的。  
事实上，阻塞队列的实现就是使用到了`第10节`讲到的条件变量，使用一个`ReentrantLock`来控制队列的并发，用两个`Condition`变量来控制队列的同步。

##### 11.1 几种常用的阻塞队列
1. **ArrayBlockingQueue：** 基于数组实现的一个阻塞队列，在创建ArrayBlockingQueue对象时必须指定容量大小。另外可以指定公平性，即不保证等待时间最长的队列最优先能够访问队列。
2. **LinkedBlockingQueue：** 基于链表实现的一个阻塞队列，在创建LinkedBlockingQueue对象时如果不指定容量大小，则默认大小为Integer.MAX_VALUE。由于内存是动态分配的，通常情况下无需指定容量大小。
3. **PriorityBlockingQueue：** 基于堆实现的一个阻塞队列，不同于`ArrayBlockingQueue`和`LinkedBlockingQueue`的先进先出性质，`PriorityBlockingQueue`会按照队列元素优先级顺序出队，每次出队的元素都是优先级最高的元素。无需指定容量大小。
4. **DelayQueue：** 基于`PriorityBlockingQueue`，`DelayQueue`中的元素只有当其指定的延迟时间到了，才能够从队列中获取到该元素。无需指定容量大小。

>**Tip:** 以上所说的常用阻塞队列中，前两种是有界队列，后两种是无界队列。

##### 11.2 阻塞队列基本APIs
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

##### 11.3 使用阻塞队列解决多生产者-多消费者同步问题
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
