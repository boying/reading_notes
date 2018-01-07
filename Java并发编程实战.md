---
title: Java并发编程实战
tags: java,并发,多线程
grammar_cjkRuby: true
---

## 2 线程安全性

### 竞态条件

当某个计算的正确性取决于多个线程的交替执行时序时，那么就会发生竞态条件。就是正确的结果要取决于运气。
最常见的竞态条件类型是“先检查后执行（Check-Then-Act）”操作，即通过一个可能失败的观测结果来决定下一步的动作。

## 5 基础构建模块

### 同步容器类

Vector, Hashtable
实现线程安全的方式：
对每个共有方法都进行同步，使得每次只有一个线程能访问容器的状态
同步容器将所有对容器状态的访问都串行化，以实现线程安全。代价是严重降低并发性，当多个线程竞争容器的锁时，吞吐量将严重降低。

#### 抛出ArrayIndexOutOfBoundsException

在没有同步的情况下，for循环可能会抛出ArrayIndexOutOfBoundsException
```
for(int i = 0; i < vector.size(); ++i){
    doSomething(vector.get(i));
}
```

#### 迭代器与ConcurrentModificationException

并发情况下，迭代的过程中操作容器，可能会引起“及时失败”（fail-fast），抛出ConcurrentModificationException

```
List list = Collections.synchronizedList(new ArrayList());
for(Object obj : list){
    doSomething(obj);
}
```

#### 隐藏迭代器

容器在使用迭代器的时候，不是那么直观
容器的toString,containsAll,removeAll,retainAll,把容器作为参数的构造函数，都会对容器进行迭代

### 并发容器

ConcurrentHashMap 
CopyOnWriteArrayList
ConcurrentLinkedQueue PriorityQueue

并发容器是针对多线程并发访问设计的。
通过并发容器来代替同步容器，可以极大地提高伸缩性并降低风险

#### 与同步容器的区别

迭代**不会**抛出ConcurrentModificationException

个人理解：同步容器的迭代器创建的时候，相当于创建了一个snapshot，保证每个线程都有各自独立的snapshot，从而线程安全。

http://stackoverflow.com/questions/3768554/is-iterating-concurrenthashmap-values-thread-safe

#### ConcurrentHashMap

使用分段锁机制实现

原子接口：
V putIfAbsent(K key, V value)
boolean remove(K key, V value)
boolean replace(K key, V oldValue, V newValue)
V replace(K key, V newValue)

#### CopyOnWriteArrayList

在每次修改时，都会创建并重新发布一个新的容器副本，从而实现可变性。
每次修改都会复制底层数组，需要一定开销。
仅当迭代操作远远多于修改操作时，才应该使用“写入时复制”容器。例如多事件通知系统。

### 阻塞方法与中断方法

中断是一种协作机制。一个线程不能强制其他线程停止正在执行的操作而去执行其他的操作。
当线程A中断B时，A仅仅是要求B在执行到某个可以暂停的地方停止正在执行的操作——前提是如果线程B愿意停止下来。

当代码调用了一个将抛出InterruptedException异常方法是，你的方法也就变成了一个阻塞方法，并且必须要处理对中断的响应。处理方式有两种基本选择：
1. 传递InterruptedException。避开异常，通常是明智的，将异常传递给调用者。
2. 恢复中断。捕获异常，并通过调用当前线程的**interrupt**方法恢复中断状态，这样调用栈中更高层的代码将看到引发了一个中断
```
public class Task implements Runnable{
    BlockingQueue queue;
    
    public void run(){
        try{
            queue.take();
        }catch (InterruptedException e){
            Thread.currentThread.interrupt();
        }
    }

}
```

### 同步工具类

#### 闭锁 CountDownLatch

CountDownLatch可以使一个或多个线程等待一组事件发生。

适用场景：
* 同时开始多个线程；
* 在多个线程都结束（达到某种状态）后，开始一段逻辑。

```
    public static long timeTasks(int nThreads, Runnable task) throws InterruptedException {
        CountDownLatch startGate = new CountDownLatch(1);
        CountDownLatch endGate = new CountDownLatch(nThreads);

        for (int i = 0; i < nThreads; ++i) {
            Thread t = new Thread() {
                @Override
                public void run() {
                    try {
                        startGate.await();
                        try {
                            task.run();
                        } finally {
                            endGate.countDown();
                        }
                    } catch (InterruptedException e) {
                    }
                }
            };
            t.start();
        }

        long start = System.nanoTime();
        startGate.countDown();
        endGate.await();
        long end = System.nanoTime();
        return end - start;
    }
```

#### FutureTask

public class FutureTask\<V> implements RunnableFuture\<V>
public interface RunnableFuture\<V> extends Runnable, Future\<V>

Demo 使用FutureTask提前加载稍后需要的数据
```
    public static class Preloader{
        public static class DataLoadException extends Exception{}

        private FutureTask future = new FutureTask(new Callable() {
            @Override
            public Object call() throws DataLoadException{
                return new Object();
            }
        });

        private final Thread thread = new Thread(future);

        public void start(){
            thread.start();
        }

        public Object get() throws InterruptedException, DataLoadException {
            try {
                return future.get();
            } catch (InterruptedException e) {
                throw e;
            } catch (ExecutionException e) {
                Throwable cause = e.getCause();
                if(cause instanceof DataLoadException){
                    throw (DataLoadException) e.getCause();
                }else{
                    throw launderThrowable(e);
                }

            }
        }

        public static RuntimeException launderThrowable(Throwable t){
            if(t instanceof RuntimeException){
                return (RuntimeException) t;
            }else if(t instanceof Error){
                throw (Error) t;
            }else{
                throw new IllegalStateException("Not unchecked", t);
            }
        }
    }
```

#### 栅栏 CyclicBarrier

所有线程必须同时到达栅栏的位置，才能继续执行。
闭锁用于等待事件，而栅栏用于等待其他线程。

#### 信号量 Semaphore

计数信号量用来控制同时访问某个特定资源的操作数量，或者同时执行某个制定操作的数量。计数信号量哈可以用来实现某种资源池，或者对容器施加边界。

Demo 使用Semaphore为容器设置边界

```
    public static class BoundedHashSet<T>{
        private final Set<T> set;
        private final Semaphore sem;

        public BoundedHashSet(int bound) {
            set = Collections.synchronizedSet(new HashSet<T>());
            sem = new Semaphore(bound);
        }
        
        public boolean add(T o) throws InterruptedException {
            sem.acquire();
            boolean wasAdded = false;
            try {
                wasAdded = set.add(o);
                return wasAdded;
            } finally {
                if(!wasAdded){
                    sem.release();
                }
            }
        }
        
        public boolean remove(T o){
            boolean wasRemoved = set.remove(o);
            if(wasRemoved){
                sem.release();
            }
            return wasRemoved;
        }
    }
```

### 缓存封装器的实现

思考点、评价点：
* **伸缩性**如何 锁定范围，持有时间
* 会不会重复计算
* 缓存污染 缓存逾期

```
    public interface Computable<A, V>{
        V compute(A arg) throws InterruptedException;
    }

    public static class Memoizer<A, V> implements Computable<A, V>{

        private final ConcurrentHashMap<A, Future<V>> cache = new ConcurrentHashMap<A, Future<V>>();
        private final Computable<A, V> c;

        public Memoizer(Computable<A, V> c) {
            this.c = c;
        }

        @Override
        public V compute(A arg) throws InterruptedException {
            while (true){
                Future<V> f = cache.get(arg);
                if(f == null){
                    Callable<V> eval = new Callable<V>() {
                        @Override
                        public V call() throws InterruptedException{
                            return c.compute(arg);
                        }
                    };
                    FutureTask<V> ft = new FutureTask<V>(eval);
                    f = cache.putIfAbsent(arg, ft);
                    if(f == null){
                        f = ft;
                        ft.run();
                    }
                }
                try {
                    return f.get();
                } catch (ExecutionException e) {
                    throw launderThrowable(e);
                } catch (CancellationException e){
                    cache.remove(arg, f);
                }
            }
        }
    }
```

## 6 任务执行

### Executor框架

#### Executor

Executor基于生产者，消费者模式，提交任务的操作相当于生产者，执行任务的线程相当于消费者。

```
public interface Executor {
    void execute(Runnable command);
}
```

#### ExecutorService

Executor的一个扩展，提供了用于管理Executor和返回Future的方法。

```

/**
 * An {@link Executor} that provides methods to manage termination and
 * methods that can produce a {@link Future} for tracking progress of
 * one or more asynchronous tasks.
 
 */
public interface ExecutorService extends Executor {

    void shutdown();


    List<Runnable> shutdownNow();


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

#### Executors

创建ExecutorService的工厂，提供其他一些方法（参考api）

```

public class Executors {
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

    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }

}

```

#### AbstractExecutorService

```
public abstract class AbstractExecutorService implements ExecutorService
```

#### ThreadPoolExecutor

```
public class ThreadPoolExecutor extends AbstractExecutorService
```

#### Future FutureTask

可以通过Future来查看任务的状态，以及取消任务
通过Executor.execute(FutureTask)即可执行任务，同时又能获取任务的返回值，执行状态等。因为FutureTask既是Runnable，同时也是Future

```
public class FutureTask<V> implements RunnableFuture<V> 

public interface RunnableFuture<V> extends Runnable, Future<V>

public interface Future<V> {

    boolean cancel(boolean mayInterruptIfRunning);

    boolean isCancelled();

    boolean isDone();

    V get() throws InterruptedException, ExecutionException;

    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

#### CompletionService 

CompletionService将Executor和BlockingQueue的功能融合在一起。可以将任务提交给它执行，然后使用类似于队列操作的take和poll方法获取已完成的结果。提交的顺序与结果的顺序**不一定一致**，取决于任务完成的快慢。
ExecutorCompletionService是它的一个实现。

```
public interface CompletionService<V> {
    Future<V> submit(Callable<V> task);

    Future<V> submit(Runnable task, V result);

    Future<V> take() throws InterruptedException;

    Future<V> poll();

    Future<V> poll(long timeout, TimeUnit unit) throws InterruptedException;
}

```

## 7 取消与关闭

### 取消任务

#### 使用volatile类型的域来保存取消状态

```
    public static class CancelableTask implements Runnable{

        private volatile boolean cancelled = false;
        private BlockingQueue queue = new LinkedBlockingDeque();

        @Override
        public void run() {
            try {
                while (!cancelled) {
                    queue.take();
                    doSomething();
                }
            } catch (InterruptedException e){}
        }

        public void cancel(){
            cancelled = true;
        }
    }
```

**潜在问题**
方法调用了阻塞方法，任务可能永远不会检查取消标识，因此永远不会结束。
如果仅仅使用了取消标识作为取消工作的方式，一定要确保程序一定会运行到检测标识的代码。

#### 使用中断来取消执行

通常，中断是实现取消最合理的方式。

```
    public static class CancelableThread extends Thread {
        private BlockingQueue queue = new LinkedBlockingDeque();

        @Override
        public void run() {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    queue.take();
                    doSomething();
                }
            } catch (InterruptedException e){}
        }

        public void cancel(){
            interrupt();
        }
    }
```

每个线程都有一个boolean类型的中断状态。

Thread中的中断方法：
1. public void interrupt()
2. public boolean isInterrupted()
3. public static boolean interrputed()
    该方法会**清除**当前线程的中断状态，返回之前的中断状态。如果返回了true，除非你想屏蔽这个中断，否则必须对它进行处理——抛出InterruptedException，或者再次调用interrupt方法恢复中断状态。

#### 中断策略

中断策略规定**线程**如何解释某个中断。

使用中断的一些原则：
* 只有线程拥有者才有资格决定中断策略。例如上面Thread CancelableThead中run方法
* 线程只能由其所有者中断。所有者可以将中断封装到某个合适的取消机制中，例如CancelableThread中cancel方法
* 对于非线程所有者（例如任务、Runnable对象）的代码来说，应该小心地保存中断状态，这样拥有线程的代码才能对中断做出响应
* 任务不应该对执行该任务的线程的中断策略做出任何假设
* 任务检查到中断时，可以推迟处理中断请求。可以记住中断请求，在完成当前任务后抛出InterruptedException或者表示已经收到中断请求。
* 只有实现了线程中断策略的代码才可以屏蔽中断请求。在常规的任务和代码库中都不应该屏蔽中断请求（但可以推迟处理）

响应中断技巧：
如果代码不会调用可中断的阻塞方法，可以通过在任务代码中轮训当前线程的中断状态来响应中断。

#### 通过Future实现取消

```
    private static ExecutorService exec = Executors.newCachedThreadPool();
    public static void timedRun(Runnable r, long timeout, TimeUnit unit) throws InterruptedException {
        Future task = exec.submit(r);
        try {
            task.get(timeout, unit);
        } catch (ExecutionException e) {
            launderThrowable(e.getCause());
        } catch (TimeoutException e) {
        } finally {
            task.cancel(true);
        }
    }
```

Future.cancle(boolean mayInterruptIfRunning) 参数如何设置？
```
 * @param mayInterruptIfRunning {@code true} if the thread executing this
 * task should be interrupted; otherwise, in-progress tasks are allowed
 * to complete
```
个人理解，设置为true的话，将会发送中断请求给执行该任务的线程。用户应根据执行该任务的线程的中断策略来设置true或false。
如果执行任务的线程是由Executor创建的，它实现了一种中断策略使得任务可以通过中断被取消，所以如果任务在**标准**Executor中运行，并通过它们的Future来取消任务，那么可以设置mayInterruptIfRunning为true。

#### 处理不可中断的阻塞 

并不是所有的可阻塞方法都能响应中断。线程执行同步的Socket I/O 或者等待获取内置锁而阻塞，中断请求interrupt()只能设置线程的中断状态，除此之外没有其他作用。

要取消阻塞在不响应中断的线程，需要根据具体的阻塞的原因来处理：
* Java.io包中的同步Socket I/O。socket的读取写入阻塞，可以通过关闭底层的套接字，使得执行的read或write等方法抛出SocketException
* Java.io包中的同步I/O。InterruptibleChannel可以响应中断请求，抛出ClosedByInterruptException，当它被关闭时，将抛出AsynchronousException
* Selector的异步IO。调用close或wakeup方法回使线程抛出closedSelectorException
* 获取某个锁。获取内置锁（同步块）阻塞将无法响应中断。但Lock类提供了lockInterruptibly方法，可以响应中断

#### 用封装的方式来取消不响应中断请求的线程

封装，还是封装

通过改写interrupt方法将非标准的取消操作封装在Thread中

```
    public class ReaderThread extends Thread {
        private final Socket socket;
        private final InputStream in;

        public ReaderThread(Socket socket) throws IOException {
            this.socket = socket;
            this.in = socket.getInputStream();
        }

        @Override
        public void interrupt() {
            try {
                socket.close();
            } catch (IOException e) {
            } finally {
                super.interrupt();
            }
        }

        @Override
        public void run() {
            byte[] buf = new byte[100];
            while (true) {
                int count;
                try {
                    count = in.read(buf);
                } catch (IOException e) {
                    return;
                }

                if (count < 0) {
                    break;
                } else if (count > 0) {
                    doSomething();
                }
            }
        }
    }
```

扩展ThreadPoolExecutor，采用newTaskFor来封装非标准取消
newTaskFor返回一个RunnableFuture，可以定制Future的cancle方法来取消

```
    public interface CancellableTask<T> extends Callable<T> {
        void cancel();

        RunnableFuture<T> newTask();
    }

    public static class CancellingExecutor extends ThreadPoolExecutor {

        public CancellingExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue);
        }

        @Override
        protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
            if (callable instanceof CancellableTask) {
                return ((CancellableTask) callable).newTask();
            } else {
                return super.newTaskFor(callable);
            }
        }
    }

    public static abstract class SocketUsingTask<T> implements CancellableTask<T> {
        private Socket socket;

        protected synchronized void setSocket(Socket socket) {
            this.socket = socket;
        }

        @Override
        public RunnableFuture<T> newTask() {
            return new FutureTask<T>(this) {
                @Override
                public boolean cancel(boolean mayInterruptIfRunning) {
                    try {
                        SocketUsingTask.this.cancel();
                    } finally {
                        return super.cancel(mayInterruptIfRunning);
                    }
                }
            };
        }

        @Override
        public void cancel() {
            try {
                if (socket != null) {
                    socket.close();
                }
            } catch (Throwable t) {
            }
        }
    }
```

### 停止基于线程的服务

线程服务，例如线程池

#### 原则

线程的所有权是不可以传递的。应用程序拥有服务，服务拥有工作者线程，但是应用程序不能拥有工作者线程。因此应用程序不能直接停止工作者线程，服务应该提供生命周期方法来关闭它自己以及停止工作者线程。

#### 关闭ExecutorService

shutdown shutdownNow

#### shutdownNow的局限性及解决方案

shutdownNow会尝试取消正在执行的任务，并返回所有已提交但尚未开始的任务，但是对于无法知道具体是哪些任务被取消。

```
    public class TrackingExecutor extends AbstractExecutorService {
        private final ExecutorService exec;
        private final Set<Runnable> tasksCancelledAtShutdown = Collections.synchronizedSet(new HashSet<Runnable>());

        public TrackingExecutor(ExecutorService exec) {
            this.exec = exec;
        }

        public List<Runnable> getCancelledTasks() {
            if (!isTerminated()) {
                throw new IllegalStateException();
            }
            return new ArrayList<>(tasksCancelledAtShutdown);
        }

        @Override
        public void execute(Runnable command) {
            exec.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        command.run();
                    } finally {
                        if (isShutdown() && Thread.currentThread().isInterrupted()) {
                            tasksCancelledAtShutdown.add(command);
                        }
                    }
                }
            });
        }

        @Override
        public void shutdown() {
            exec.shutdown();
        }

        @Override
        public List<Runnable> shutdownNow() {
            return exec.shutdownNow();
        }

        @Override
        public boolean isShutdown() {
            return exec.isShutdown();
        }

        @Override
        public boolean isTerminated() {
            return exec.isTerminated();
        }

        @Override
        public boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException {
            return exec.awaitTermination(timeout, unit);
        }
    }
```

### 处理非正常的线程终止

```
    in Thrad.java

    public interface UncaughtExceptionHandler {
        void uncaughtException(Thread t, Throwable e);
    }

    public void setUncaughtExceptionHandler(UncaughtExceptionHandler eh)
    
```

只有线程的所有者能够改变线程的UncaughtExceptionHandler。
标准线程池允许当发生未捕获异常时结束线程，但由于使用了一个try-finally代码块来接受通知，因此当线程结束时，将有新的线程来代替它。如果没有提供捕获异常处理器或者其他的故障通知机制，那么任务会悄悄失败，从而导致极大混乱。如果希望在任务由于发生异常而失败时获得通知，并且执行一些特定于任务的回复操作，那么可以讲任务封装在能捕获异常的Runnable或Callable中，或者改写ThreadPoolExecutor的afterExecute方法。


### JVM关闭

#### 正常关闭 强行关闭

正常关闭：
* 最后一个非守护线程结束
* System.exit
* 收到SIGINT信号

强行关闭：
* Runtime.halt
* 收到SGIKILL信号

正常关闭会出发已注册的关闭钩子，强行关闭则不会

#### 关闭钩子 Shutdown Hook

```
Runtime.getRuntime().addShutdownHook(new Thread{
    public void run(){
        doSometing();
    }
});
```

JVM 不会停止或者中断任何在关闭时仍然运行的应用程序线程。当JVM最终结束时，这些线程被强行结束。

钩子的适用场景：
关闭钩子可以用于实现服务或者应用程序的清理工作，例如删除临时文件，清除无法由操作系统自动清除的资源。

使用技巧：
被注册的关闭钩子将并发执行，为了避免关闭钩子线程之间出现竞态条件或死锁等问题，可以对所有的服务使用同一个钩子，保证关闭操作在单线程中串行执行。

#### 守护线程

java线程分两种，普通线程和守护线程。

JVM启动时，除了主线程，其他所有线程（垃圾回收器等）都是守护线程。创建新线程，新线程继承创建它的线程的守护状态，因此主线程创建的线程都是普通线程。

区别：
* 当所有普通线程结束时，JVM也就结束了。
* 线程退出时发生的操作。当JVM停止时，所有任然存在的守护线程将被抛弃，既不会执行finally代码块，也不会执行回卷栈，JVM只是直接退出

是用场景：
最好用于执行“内部”任务，例如周期性地从内存的缓存中移除逾期数据

#### 终结器

终结器不能保证它们将在何时运行设置是否会运行，避免是用终结器

## 8 线程池的使用

### 不当的使用

#### 线程饥饿死锁

只要线程池中的任务需要无限期地等待一些必须有线程池中其他任务才能提供的资源或条件，例如某个任务等待另一个任务的返回值或执行结果，那么除非线程池足够大，否则将发生饥饿死锁。

```
    public static class ThreadDeadlock{
        ExecutorService exec = Executors.newSingleThreadExecutor();
        
        public static class LoadFileTask implements Callable<String>{
            private String desc;
            public LoadFileTask(String desc) {
                this.desc = desc;
            }

            @Override
            public String call() throws Exception {
                return desc + " done";
            }
        }
        
        public class RenderPageTask implements Callable<String>{
            @Override
            public String call() throws Exception {
                Future<String> header, footer;
                header = exec.submit(new LoadFileTask("header.html"));
                footer = exec.submit(new LoadFileTask("footer.html"));
                return header.get() + " " + footer.get();
            }
        }
    }
```

#### 运行时间较长的任务

如果线程池中的线程的数量远小于在稳定状态下执行时间较长任务的数量，那么到最后可能所有的线程都会运行这些执行时间较长的任务，从而影响整体性能。

#### 恰当的使用

只有当任务都是**同类型**的并且**互相独立**时，线程池的性能才能达到最佳。
同类型能够避免运行时间较长的任务长期占用线程池，导致运行时间较短的任务得不到线程去执行。
互相独立能够避免饥饿死锁。

### 设置线程池的大小

#### 影响线程池大小的因素

线程池的理想大小取决于被提交任务的类型和所部属系统的特性。

* 计算密集型、I/O密集型，二者兼有
* cpu的个数（核数）
* 任务需要的资源：内存，文件句柄，socket连接，数据库连接。类似于木桶原理

#### 公式

Ncpu = number of CPUs
Ucpu = target CPU utilization, 0 <= Ucpu <= 1  CPU利用率
W/C = ratio of wait time to compute time
Nthreads = Ncpu * Ucpu * (1 + W/C)

在某个基准的负载下，分别设置不同大小的线程池来运行应用程序，观察CPU的利用率水平

### 配置ThreadPoolExecutor

```
    public ThreadPoolExecutor(int corePoolSize,
                                   int maximumPoolSize,
                                   long keepAliveTime,
                                   TimeUnit unit,
                                   BlockingQueue<Runnable> workQueue,
                                   ThreadFactory threadFactory,
                                   RejectedExecutionHandler handler){}
```

corePoolSize  基本大小，目标大小，默认情况下，创建的线程是不会被回收。只有当调用ThreadPoolExecutor.allowCoreThreadTimeOut（true）后，才会回收core thread中的空闲线程
maximunPoolSize 最大大小，当corePoolSize已被占用，队列已满，将启动超出的corePoolSize的线程。非core线程空闲的话，竟会被回收
keepAliveTime, unit  空闲线程超时回收时间
workQueue 任务队列，当core线程满的时，被提交的任务将会被放到workQueue中
threadFactory 创建线程池中线程的工厂
handler 当core thread，max thread，queue都已经满的时候，再提交任务，将会采用拒绝策略

#### Executors创建4种典型的ThreadPool

创建ExecutorService的工厂，提供其他一些方法（参考api）

```

public class Executors {
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

    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }

}

```

#### 线程池的工作队列

基本的任务排队方法有4种：
1. 有界队列
    ArrayBlockingQueue, LinkedBlockingQueue(capcity)

2. 无界队列
    LinkdedBlockingQueue()。newFixedThreadPool,newSingleThreadExecutor使用LinkdedBlockingQueue。

3. 同步移交队列
    SynchronousQueue。 newCachedThreadPool。采用SynchronousQueue的原因有两个：一是保证提交的任务立马执行，二是能够复用工作者线程（工作和线程会从queue中取任务）

4. 优先级队列
    PriorityBlockingQueue

#### 线程池工作过程 个人理解

1. 检查当前工作线程是不是小于corePoolSize，如果小于的话，将直接创建一个工作线程来运行任务
2. 如果当前工作线程大于corePoolSize，将尝试将任务放入工作队列中。
3. 如果工作队列拒绝接受任务（满了，或者SynchronousQueue没有需求者），将尝试用创建非core的工作线程来执行任务。
4. 如果工作队列接受了任务，判断线程池中是否有工作者线程，如果没有的话，将创建一个非core的工作者线程。
5. 任务既不能放入队列中，又没有工作者线程来处理它，将会拒绝任务，调用拒绝逻辑

#### 饱和策略

在任务无法立即被工作者线程执行，并且无法放入队列中，将会触发饱和策略
可以在ThreadPoolExecutor constructor或setRejectedExecutionHandler来设置饱和策略
框架提供了下面4中饱和策略
1. AbortPoliciy
    默认。提交任务是将会抛出RejectedExecutionException异常
2. CallerRunsPolicy
    在提交这线程中运行。例子：WebServer如果使用在主线程（accept线程）中采用这种策略，当线程池满了时，主线程来执行任务，将暂时不会调用accept方法，将请求保存在TCP层的队列中。持续过载的话，TCP层请求队列被填满，将开始抛弃请求。这样可以实现**平缓的性能降低**。
3. DiscardPolicy
    悄悄抛弃提交任务。submit返回的Future对象是？
4. DiscardOldestPolicy
    抛弃下一个将要被执行的任务（优先级队列的话，将抛弃优先级最高的任务），尝试重新提交任务

#### 线程工厂

线程池中的线程是通过线程工厂创建的。
我们可以定制线程工厂，用它来创建我们需要的定制的线程，例如，为线程添加一个UncaughtExceptionHandler，添加调试信息，为线程去取一个有意义的名字

下面的线程工厂创建的线程添加了一些特性，获取被创建过的线程个数，活着的线程个数，开关debug日志

```
   public static class MyAppThread extends Thread {
        public static final String DEFAULT_NAME = "MyAppThread";
        private static volatile boolean debugLifecycle = false;
        private static final AtomicInteger created = new AtomicInteger();
        private static final AtomicInteger alive = new AtomicInteger();
        private static final Logger log = Logger.getAnonymousLogger();

        public MyAppThread(Runnable target) {
            this(target, DEFAULT_NAME);
        }

        public MyAppThread(Runnable runnable, String name) {
            super(runnable, name + "-" + created.incrementAndGet());
            this.setUncaughtExceptionHandler(new UncaughtExceptionHandler() {
                @Override
                public void uncaughtException(Thread t, Throwable e) {
                    log.log(Level.SEVERE, "UNCAUGHT in thread " + t.getName() + e);
                }
            });
        }

        @Override
        public void run() {
            boolean debug = debugLifecycle; // 确保一致性
            if (debug) log.log(Level.FINE, "Created " + getName());
            try {
                alive.incrementAndGet();
                super.run();
            } finally {
                alive.decrementAndGet();
                if (debug) log.log(Level.FINE, "Exiting " + getName());
            }
        }
        
        public static int getThreadsCreated(){
            return created.get();
        }
        
        public static int getThreadAlive(){
            return alive.get();
        }
        
        public static boolean getDebug(){
            return debugLifecycle;
        }
        
        public static void setDebug(boolean b){
            debugLifecycle = b;
        }
    }
    
    public static class MyThreadFactor implements ThreadFactory{
        private final String poolName;

        public MyThreadFactor(String poolName) {
            this.poolName = poolName;
        }

        @Override
        public Thread newThread(Runnable r) {
            return new MyAppThread(r, poolName);
        }
    
```

#### 继承ThreadPoolExecutor

子类可以改写ThreadPoolExecutor中的beforeExecute，afterExecute，terminated方法，来定制化自己的行为。
**工作者线程**（执行任务的线程）在开始执行任务调用beforeExecute，结束任务后调用afterExecute，可以添加一些功能，如：添加日志、计时、监视、统计信息收集等功能。
线程池结束（任务都已完成，工作者线程都关闭）将调用terminated，可以用来释放在Executor分配的资源，发送通知，记录日志。

demo（注意Threadlocal的使用）
```
   public static class TimingThreadPool extends ThreadPoolExecutor {
        private final ThreadLocal<Long> startTime = new ThreadLocal<>();
        private final Logger log = Logger.getLogger("TimingThreadPool");
        private final AtomicLong numTasks = new AtomicLong();
        private final AtomicLong totoalTime = new AtomicLong();

        @Override
        protected void beforeExecute(Thread t, Runnable r) {
            super.beforeExecute(t, r);
            log.fine(String.format("Thread %s: start %s", t, r));
            startTime.set(System.nanoTime());
        }

        @Override
        protected void afterExecute(Runnable r, Throwable t) {
            try {
                long endTime = System.nanoTime();
                long taskTime = endTime - startTime.get();
                numTasks.incrementAndGet();
                totoalTime.addAndGet(taskTime);
                log.fine(String.format("Thread %s: end %s, time=%dns", t, r, taskTime));
            } finally {
                super.afterExecute(r, t);
            }
        }

        @Override
        protected void terminated() {
            try {
                log.info(String.format("Terminated: avg time = %dns", totoalTime.get() / numTasks.get()));
            } finally {
                super.terminated();
            }
        }

        public TimingThreadPool(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue);
        }

        public TimingThreadPool(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory);
        }

        public TimingThreadPool(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, RejectedExecutionHandler handler) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, handler);
        }

        public TimingThreadPool(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);
        }
    
```

## 10 避免活跃性危险

### 死锁

当**多个**线程**相互持有**彼此正在等待的锁而又**不释放**自己已持有的锁时，会发生死锁

#### 产生死锁的四个必要条件

1. 互斥条件。一个资源（锁，数据库连接）只能由一个进程占有
2. 不可抢占条件。进程不能抢占被其他线程占有的资源
3. 请求与保持条件。一个进程因请求资源而阻塞时，对已获得的资源保持不放。
4. 循环等待条件。

互斥，不可抢占 描述的是资源特点
占有申请，循环等待描述的是线程的行为

#### 锁顺序锁

如果所有线程以固定的顺序来获得锁，那么程序就不会出现锁顺序死锁的问题。也就是说，如果线程不是以固定顺序来获得锁，将存在死锁的风险。

如果在持有锁的情况下调用某个外部方法，那么需要这将出现活跃性问题，在这个外部方法中可能会获取其他锁，或者阻塞时间过长，导致其他线程无法及时获得当前被持有的锁。

#### 资源死锁

资源（池）在也可视为一种锁。

有两类死锁的形式：
1. 资源的顺序锁。
    类似锁顺序死锁
2. 线程饥饿死锁。
    如果某些任务需要等待其他任务的结果，那么这些任务往往是产生线程饥饿死锁的主要来源，有界线程池/资源池与相互依赖的任务不能一起使用。

### 死锁的避免与诊断

#### 用支持定时的锁避免死锁

#### 线程转储（Thread Dump）分析死锁

### 其他活跃性问题

#### 饥饿

#### 低响应性

#### 活锁

活锁（LiveLock）不会阻塞线程，但线程也不能继续执行，因为线程将不断**重复**执行**相同**的操作，而且**总是失败**。

当多个相互协作的线程都对彼此进行响应从而修改各自的状态，并使得任何一个线程都无法继续执行时，就发生了活锁。类比两个人在路上相遇，他们彼此都让出对方的路，然而又在另外一条路上相遇了，因此反复的避让下去，谁都无法继续前进。

解决活锁的问题，需要在重试机制中引入随机性。

## 11 性能与可伸缩性

### 可伸缩性含义

当增加计算资源时（cpu、内存、I/O带宽等），程序的吞吐量或处理能力能相应地增加

### 性能衡量指标——多快，多少

多快——一些指标（服务时间，等待时间）用于衡量程序的“运行速度”，即某个指定的**任务单元**需要“多快”才能处理完成。多快指的是单个的运行效率。

多少——一些指标（生产量、吞吐量更、可伸缩性）用于程序的“处理能力”，即在计算资源一定的情况下，能完成“多少”任务。多少指的是总量。

“多快”，“多少”是**完全独立**的，有时甚至是**矛盾**的。
要实现更高的可伸缩性或硬件利用率，通常会增加各个任务索要处理的工作量。

对于服务器应用程序来说，“多少”——可伸缩性、吞吐量和生产量，往往比“多快”更重视。通常会接受每个工作单元执行更长的时间或小号更多的计算资源，以换取应用程序在增加更多资源的情况下处理更高的负载。

### Amdahl定律 阿姆达尔定律

在增加计算资源的情况下，程序在理论上能够实现最高加速比，这个值取决于程序中可并行组件与串行化组件所占的比重。

Speedup <= 1 / (F + (1-F)/N)
F 必须串行执行的部分
N 表示处理器个数

加速比是有**极限**的，当N趋近于无穷大时，最大的加速比趋近于1/F

**程序的可伸缩性取决于在所有代码中必须被串行执行的代码比例。**

### 提高伸缩性的方法——减少锁竞争

在并发程序中，对伸缩性的最主要威胁就是独占方式的资源锁。

#### 锁竞争带来的影响

在锁上发生竞争，就是在锁范围内的代码将在多线程中**串行执行**，并且会导致上下文切换降低性能，因此减少锁的竞争能够提高性能和可伸缩性。

#### 降低锁竞争的方法

降低锁的粒度，可以从下面这些方面考虑

* 减少锁持有时间
* 缩小锁代码块的范围（快进快出）
* 降低锁请求的频率
* 避免通用锁，将一个锁变为多个锁，每个锁各司其职。
    ConcurrentHashMap的实现中使用了一个包含16个锁的数组，每个锁保护所有散列桶的1/16，其中第N个散列桶由（N mod 16）来保护。因此它可以支持大多16个并发写，提高了并发性。
    这也是有风险的，更多的锁，意味着死锁的可能性更高
* 采用非独占锁或非阻塞锁来代替独占锁
    ReadWriteLock，原子变量


## 13 显式锁

### 特性（与Synchronized区别）

#### 轮询锁与定时锁

tryLock 非阻塞

#### 可中断的锁获取操作

lockInterruptly

#### 非块结构的加锁

lock()
unlock()

#### 公平性

在公平锁上，线程将按照他们发出请求的顺序来获得锁，非公平锁允许插队。

非公平锁性能要高于公平锁性能。

#### 非公平锁性能高的原因

恢复一个被挂起的线程与该线程真正开始之间存在严重的**延迟**。假设线程A持有一个锁，线程B请求这个锁。由于锁被A持有，B将被挂起。当A释放时，B将被唤醒，因此会再次尝试获取锁。与此同时，如果C也请求这个锁，那么C很可能会在B被完全唤醒之前获得、使用及释放这个锁。这是一种“双赢”的局面：B获得锁的时候并没有推迟，C更早地获得了锁，因此吞吐率获得提高。

A (own lock)  -----------------> relase lock    
B acquire lock -------> blocked              -------------------------------------------------------> waked up and try acquire lock -----> won lock
------------------------------------ C acquire lock --> own lock --> relase lock

### synchronized与ReentrantLock的选择

java5.0 ReentrantLock性能远远高于synchronized。Java6略有胜出。

synchronized使用方便，由于同步块的缘故，不容易出错。

在不需要ReentrantLock特性的时，优先使用synchronized。

### 读写锁

#### 适用情况
一个资源可以被多个读操作访问，或这个被一个写操作访问，但二者不能同时进行。

#### 性能比较

比独占锁性能低一点，因为复杂性高


## 15 原子变量与非阻塞同步机制

### 原子变量与Volatile

原子变量提供了与votatile类型变量相同的内存语义（线程可见性，禁用指令重拍），还支持原子的更新操作，更加适用于实现计数器、序列发生器和统计数据收集

### 原子变量与锁

#### 锁的缺点（特点）

锁定方式对于**细粒度**的操作（例如递增计数器）来说是一种**高开销**的机制。

锁在挂起和恢复线程过程中，需要线程调度、上下文切换，存在着很大的开销。如果被锁包裹的操作少粒度细，那么当锁上存在激烈竞争时，调度开销与工作开销的比值会非常高。

#### CAS 比较并交换指令

#### CAS与锁的比较

优点

1. 在无竞争情况下，锁的开销比CAS大。实现锁定时，需要遍历JVM中一条非常复杂的代码路径
2. 在有竞争情况下，线程挂起，上线文切换，开销更大


缺点

它将调用者处理竞争问题（通过重试、回退、放弃），而在所终能自动处理竞争问题（线程在获得锁之前将一直阻塞）

#### JVM对CAS的支持

在支持CAS的平台上，运行时把他们编译成相应机器指令。在最坏的情况下，如果不支持CAS指令，JVM将使用**自旋锁**。

#### AtomicInteger 实现原理

Unsafe中native底层的cas支持

```
AtomicInteger.java

    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }
    
Unsafe.class

    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }    
    
    public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```

## Other 

### 竞态条件

是指多个进程或者线程并发访问和操作同一数据且执行结果与访问发生的特定顺序有关的现象。换句话说，就是线程或进程之间访问数据的先后顺序决定了数据修改的结果

### 双重检查锁模式 double-checked locking

http://www.infoq.com/cn/articles/double-checked-locking-with-delay-initialization

keyword
指令重排
votatile
第一个if (instance == null)条件判断是有问题。

### final

指令重排规则：
1. 在构造函数内对一个final field的写入，与随后把这个被构造的对象的引用赋值给一个引用变量，这两个操作不能重排序
2. 初次读一个包含field filed的对象，与随后初次读这个final field，这两个操作之间不能重排序


