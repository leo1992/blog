---
layout: post
title: AsyncTask, ThreadHandler, IntentService源码解读
date: 2018-06-05
img: blog_head_4.jpeg 
tags: [Android, 多线程]
---

# AsyncTask, ThreadHandler, IntentService源码解读 
## AsyncTask

```java
private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
private static final int KEEP_ALIVE_SECONDS = 30;
```

AsyncTask中核心线程数的范围为 2-4个，当依据cpu核数来决定核心线程数是，会-1留一个防止cpu饱和。  
keepalivetime就是线程数大于corePoolSize时空闲线程存活的最大时间,但是如果设置了allowCoreThreadTimeOut，核心线程在指定时间内没有取到也会被认为超时。AsyncTask中THREAD_POOL_EXECUTOR中默认是true的。  

```java
boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
Runnable r = timed ? 
            workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
            workQueue.take();
    if (r == null) timeout = true;//(这行改写的)
```

AsyncTask中有两个线程池executor，能够执行并行任务的Executor，和一次执行一个的串行的Executor，AsyncTask默认的是串行Executor。
```java
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
```

![image](https://raw.githubusercontent.com/leo1992/blog/master/_posts/blog_image/AsyncTask-class.jpg)

IntentHandler继承handler，在构造函数中传入了Looper.getMainLooper()用主线程的looper，所以intenthandler是在主线程中执行的。另外覆写了handleMessage方法，接受两个消息：执行结果和状态更新，调用finish()回调和onProgressUpdate()回调，finish()方法中根据情况会调用onCancelled(）和 onPostExecuete方法，说明这两个方法也是在主线程中执行.  
AsyncTask中的WorkRnnable实现了call方法，在方法中调用doInBackground, 将返回结果直接通过postResult返回。  
所以问题：为什么要设置两次结果？查找了一下futuretask中的结果设置，即done方法的调用（finishCompletion方法）发现，除了运行结果的时候会执行外，在出现很多异常的时候也会执行，所以认为，futureTask也就是在执行各种异常的情况下，能够设置运行结果，而且WorkRnnable的执行是futureTask去调用的，所以我认为：1. 用futuretask封装了整个生命周期的状态，并且要支持在调用前出现异常时的结果处理 ；2. 支持在任何时机去主动获取结果，即使没有执行结束 3. 在期间的任何异常都能够有对应的异常状态结果的返回。 
AsyncTask中的future实现了done方法，在方法中通过futureTask的get方法取到结果，然后调用postResultIfNotInvoked来设置结果，即向handler发送post_result消息。   
FutureTask用于异步获取执行结果或取消执行任务的场景，它的主要功能有：判断任务是否完成，获取任务执行结果（结果只可以在计算完成之后获取，get方法会阻塞当计算没有完成的时候，一旦计算已经完成， 那么计算就不能再次启动或是取消）, 中断任务。

-------- 
*注释
多线程是否能并发的线程数量，需要返回可用处理器的Java虚拟机的数量。方法：Runtime.getRuntime().availableProcessors()。  
**Runnable, Callable, Future 和 FutureTask。**  
参考：https://blog.csdn.net/codershamo/article/details/51901057  
Runnable 接口中要实现的是run()方法，但是返回值是void，这样就无法在执行后返回执行结果，即使可以返回结果，但是如果任务存在延迟，例如开新线程中执行不能同步的情况，就无法直接取得运行结果（一般在线程中，这种情况会再直接结束后通过回调来返回数据？所以并不是没有办法解决的），**Callable**接口定义的call方法是支持返回结果的。  
有时候在执行长时间的操作时，有需要取消任务的情况。在Executor框架中，已提交但尚未开始的任务可以取消，对于已经开始执行的任务，只有当它们响应中断时才能取消。**Future**表示一个任务的生命周期，并提供了方法来判断是否已经完成或取消，以及获取任务的结果和取消任务等。其中的get方法用来获取执行结果，**这个方法会产生阻塞，会一直等到任务执行完毕才返回**。get(long timeout, TimeUnit unit)用来获取执行结果，如果在指定时间内，还没获取到结果，就直接返回null。（类似于executor中的poll的两个方法）  
FutureTask实现了 future、runnable接口，并且持有callale成员变量。是一个可取消的异步计算。    
**FutureTask**
FutureTask有以下几个状态，对应的迁移如注释，

```java
/* Possible state transitions:
 * NEW -> COMPLETING -> NORMAL
 * NEW -> COMPLETING -> EXCEPTIONAL
 * NEW -> CANCELLED
 * NEW -> INTERRUPTING -> INTERRUPTED
 */
private volatile int state;
private static final int NEW          = 0;//表示内部成员callable已成功赋值
//woker thread在处理task时设定的中间状态，处于该状态时,说明worker thread正准备设置result
private static final int COMPLETING   = 1;
//当设置result结果完成后，FutureTask处于该状态，代表过程结果
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;
```
1. 判断任务是否完成
	isDone: state != NEW  futuretask的构造函数中初始化为new，到normal之后就都属于完成状态，completing时已经处于正在设置result，认为属于已完成
	isCanceled: 返回当前的state是否>=cancelled（CANCELLED, INTERRUPTING, INTERRUPTED）  
2. 获取任务执行结果
方法如下，如果已经完成，则直接根据结果状态返回，如果未完成，等待然后返回结果。进入awaitDone方法后会进入死循环，直到状态未completing时返回状态，或遇到异常时抛出异常。中间会创建waitnode（没来得及细看，目前状态认为是，加入一个等待链表节点，在完成后会被取消），  
completing时正在设置，直接返回有问题么？同步加锁了？  
callable在这里的作用是在run方法中执行callbale.call，然后依据结果设置result。  

```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}

private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }
```
futuretask在创建时，有一个会获取executor的callable，其callable实现如下，（如果task.run已经在子线程中调用，那么return result就是顺序执行后返回的，也就是run执行完成，这个时候能保证结果。result会在task.run中被改变？）  
所以这个结构中，future会根据状态支持等待执行完成返回result；callable的作用就是等待执行完成设置结果。这个有什么意义？

```java
static final class RunnableAdapter<T> implements Callable<T> {
    final Runnable task;
    final T result;
    RunnableAdapter(Runnable task, T result) {
        this.task = task;
        this.result = result;
    }
    public T call() {
        task.run();
        return result;
    }
}
```

在AsyncTask创建时，callable的实现如下，调用doInBackground的结果，返回，在返回时会通过postResult设置结果
  
```java
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

             Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };
```
3. 支持可取消
如果不是new的状态，即done的状态时，返回false，否则通过t.interrupt方法中断任务 。 

```java
public boolean cancel(boolean mayInterruptIfRunning) {
    if (!(state == NEW &&
          U.compareAndSwapInt(this, STATE, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {    // in case call to interrupt throws exception
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    t.interrupt();
            } finally { // final state
                U.putOrderedInt(this, STATE, INTERRUPTED);
            }
        }
    } finally {
        finishCompletion();
    }
    return true;
}
```
