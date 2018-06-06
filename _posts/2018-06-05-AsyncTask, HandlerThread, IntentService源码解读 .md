---
layout: post
title: AsyncTask, HandlerThread, IntentService源码解读
date: 2018-06-05
img: blog_head_4.jpeg 
tags: [Android, 多线程]
---

# AsyncTask, HandlerThread, IntentService源码解读 

## AsyncTask

## 线程池的配置参数
```java
private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
private static final int KEEP_ALIVE_SECONDS = 30;
```

AsyncTask中**核心线程数**的范围为 2-4个，当依据cpu核数来决定核心线程数是，会-1留一个防止cpu饱和。  
**keepalivetime** 就是线程数大于corePoolSize时空闲线程存活的最大时间,但是如果设置了allowCoreThreadTimeOut，核心线程在指定时间内没有取到也会被认为超时。AsyncTask中默认是true的。所以只要线程闲置超过keepalive时长，会被回收。操作位于ThreadPoolExecutor的getTask()方法中，如下。  

```java
boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
Runnable r = timed ? 
            workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
            workQueue.take();
    if (r == null) timeout = true;//(这行改写的)
```


![image](https://raw.githubusercontent.com/leo1992/blog/master/_posts/blog_image/AsyncTask-class.jpg)

# 线程池  
AsyncTask中有两个线程池，能够执行并行任务的线程池，和一次执行一个的串行的线程池，AsyncTask默认的是串行执行的线程池。它的具体执行也是通过THREAD_POOP_EXECUTOR的执行的。代码逻辑如下。SerialExecutor内部维护一个Runable数组存放要执行的任务，每次调用execute的时候，将任务存放在mTasks中，并且在执行完毕后再去调度下一个任务。在最开始没有任务在执行的时候（mActive == null），就直接去调度下一个。scheduleNext的逻辑就是从mTasks中取一个任务，赋给mactive，然后让THREAD_POOP_EXECUTOR去执行当前任务。
  
```java
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```

## 线程切换  
IntentHandler继承handler，在构造函数中传入了Looper.getMainLooper()用主线程的looper，所以intenthandler是在主线程中执行的。另外覆写了handleMessage方法，接受两个消息：执行结果和状态更新，调用finish()回调和onProgressUpdate()回调，finish()方法中根据情况会调用onCancelled(）和 onPostExecuete方法，说明这两个方法也是在主线程中执行. 
## 返回运行结果
-- 通过 FutureTask, Callable 实现，具体见注释  
AsyncTask中的**worker(WorkRnnable)**继承自**Callable**，实现了call方法，在被调用时调用asyncTask的doInBackground方法, 将返回结果直接通过postResult返回。  
* 所以问题：为什么要设置两次结果？  
查找了一下futuretask中的结果设置的位置，即done方法的调用（finishCompletion方法中调用）发现，除了运行结果的时候会执行外，在出现很多异常的时候也会执行，所以认为，futureTask也就是在执行各种异常的情况下，能够设置运行结果，而且WorkRnnable的执行是futureTask去调用的，所以我认为：1. 用futuretask封装了整个生命周期的状态，并且要支持在调用前出现异常时的结果处理 ；2. 支持在任何时机去主动获取结果，即使没有执行结束 3. 在期间的任何异常都能够有对应的异常状态结果的返回。 
AsyncTask中的**future(FutureTask)**实现了done方法，在方法中通过futureTask的get方法取到结果，然后调用postResultIfNotInvoked来设置结果，即**向handler发送post_result消息**。   
FutureTask用于异步获取执行结果或取消执行任务的场景，它的主要功能：判断任务是否完成，获取任务执行结果（结果只可以在计算完成之后获取，get方法会阻塞当计算没有完成的时候，一旦计算已经完成， 那么计算就不能再次启动或是取消）, 中断任务。

## AsyncTask的执行  
asyncTask的执行最终都是通过THREAD_POOL_EXECUTOR.execute来执行的（原因见上面的线程解释），调用execute方法默认用串行执行的线程池，需要并行执行的时候，调用executeOnExecutor，可以传入THREAD_POOL_EXECUTOR。  
三个方法：  
- execute(Params...): 串行执行的方法  
- execiteOnExecutor: 支持并行执行  
- execute(runnable): 串行执行。省略了判断状态的逻辑，认为就是不关心运行结果，和过程，只是要实现后台操作的时候用，runnable就是在后台线程中执行的操作。simple版，TextView中就用到。   


```java

@MainThread
public static void execute(Runnable runnable) {
    sDefaultExecutor.execute(runnable);
}
    
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}

@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    if (mStatus != Status.PENDING) {// 重复执行会抛出异常？
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }

    mStatus = Status.RUNNING;

    onPreExecute();

    mWorker.mParams = params;
    exec.execute(mFuture);

    return this;
}
```
THREAD_POOL_EXECUTOR的执行过程- to_be_continue
1. 判断当前工作执行的线程少于核心线程数，就创建一个新的线程执行
2. 如果成功加入了工作队列，并且在执行了，

```java
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
	 int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```
## 注意
1. AsyncTask是抽象类，在调用处通常会成为内部匿名类，所以在task中用到view，和持有activity的时候注意防止内存泄漏。
2. AsyncTask类应该在主线程中加载（activitythread中有执行asyncTask.init()之所以这点暂时不用考虑）。是因为handler负责主线程的执行任务，所以必须在主线程中执行。书上的原因说因为为它为static变量，所以会在类加载的时候被创建赋值，所以应该在主线程中创建。当前源码（24+）版本的代码sHandler是推迟了创建对象赋值的操作，handler都是通过getHandler去获取，在创建时通过Looper.getMainLooper()保证在线程中。所以依然不用考虑这个问题。
3. AsyncTask对象必须在主线程中创建，AsyncTask的执行（executor）应该在主线程中。为什么？
4. AsyncTask不适合执行特别耗时的任务。网上提到是因为跟activity的生命周期（旋转）和容易内存泄露的考虑。
5. asyncTask取消之后（cancel方法），onPostExecute和onProgressUpdate不会再调用了，但doInBackground方法却会一直执行下去，对用户来说，在调用了cancle方法后，后台的任务就不会在影响到主线程的界面变化了。（https://blog.csdn.net/qq_25806863/article/details/72782050）

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

# HandlerThread

官方解释HandlerThread就是一个可以创建一个带有looper的线程，我们、、可以直接使用这个 Looper 创建 Handler。   
**应用场景**：需要创建线程间传递消息的线程时。看了一个例子是在网络连接管理的ConnectivityManager中使用到了HandlerThread来处理PacketKeepalive事件。可以看到，在handlerMessage中，可以获取到message的类型，错误信息，然后进行处理。发送消息是在NetWorkAgent（继承Handler）中的handler去发送的。因此，实现了线程之间的消息传递。

```java
//ConnectivityManager类中
HandlerThread thread = new HandlerThread(TAG);
thread.start();
mLooper = thread.getLooper();
mMessenger = new Messenger(new Handler(mLooper) {
    @Override
    public void handleMessage(Message message) {
        switch (message.what) {
            case NetworkAgent.EVENT_PACKET_KEEPALIVE:
                int error = message.arg2;
                try {
                    if (error == SUCCESS) {
                        if (mSlot == null) {
                            mSlot = message.arg1;
                            mCallback.onStarted();
                        } else {
                            mSlot = null;
                            stopLooper();
                            mCallback.onStopped();
                        }
                    } else {
                        stopLooper();
                        mCallback.onError(error);
                    }
                } catch (Exception e) {
                    Log.e(TAG, "Exception in keepalive callback(" + error + ")", e);
                }
                break;
            default:
                Log.e(TAG, "Unhandled message " + Integer.toHexString(message.what));
                break;
        }
    }
});
            
//调用
// NetWorkAgent类中
queueOrSendMessage(EVENT_PACKET_KEEPALIVE, slot, reason);
private void queueOrSendMessage(int what, int arg1, int arg2, Object obj{
    Message msg = Message.obtain();
    ...
    queueOrSendMessage(msg);
}
private void queueOrSendMessage(Message msg) {
    synchronized (mPreConnectedQueue) {
        if (mAsyncChannel != null) {
            mAsyncChannel.sendMessage(msg);
        } else {
            mPreConnectedQueue.add(msg);
        }
    }
}
```  
**为什么要使用？**攒了一下网上的解释  
1. 多个耗时任务串行执行时，防止频繁创建一般的创建new Thread(){…}.start()的方法可以处理单个耗时任务，但是如果有多个耗时任务串行时，频繁的创建很耗系统资源，容易存在性能问题。
2. 多线程通信时，handler的looper使用问题。当创建多个线程时，需要让多个线程之间能够方便通信（包括主线程），因此需要使用到该线程的handler，HandlerThread就是用来做这件事。Handler在执行时创建handler（因为在创建handler时要传入线程的looper，此时可能线程并未准备好，所以在执行中再创建handler），准备looper。handlerthread就是帮助我们做好了这些操作。

## 实现
HandlerThread继承了Thread，实现就是增加了looper，处理了线程的终止。

### looper的创建和获取
**创建：**run方法中，调用Looper.prepare 和 looper.loop准备好线程的looper并开启，(所说的run方法的死循环就是looper.loop是一个死循环)，在创建时如果出现并发执行的情况时，要考虑同步问题，因为加了对象锁。mTid是当前线程的标识，如果执行完成，tid会置回-1。  
**获取：**getLooper方法会返回looper，在这个方法中会判断mLooper是否为空，如果未空会调用wait等待，等到线程准备好looper之后再返回。wait方法注释是:让当前线程等待，直到调用了当前对象的notify或notifyall方法。在run方法中可以看到，mLooper赋值之后会调用notifyAll()方法，因为能够保证looper在获取时不会为空或线程未准备好。

```java
@Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
    
public Looper getLooper() {
	if (!isAlive()) {
	    return null;
	}
	    
	// If the thread has been started, wait until the looper has been created.
	synchronized (this) {
    while (isAlive() && mLooper == null) {
        try {
            wait(); 
        } catch (InterruptedException e) {
        }
    }
}
return mLooper;
}
```
### 执行

HandlerThread的执行不是传统的再thread的run方法中取执行操作，而是让handler去处理接受到的消息去执行操作，在handleMessage方法中执行异步任务。前面提到的串行执行耗时任务，应该就是用hander发送msg，然后通过looper循环，来保证串行处理。

### quit和 quitSafely  
对应looper的quit和quitSafely，是否执行完之后再停止。

### 使用过程
1. 创建 HandlerThread handlerThread = new HandlerThread(TAG）
2. 开启线程 handlerThread.start();
3. 创建消息处理机制，handler的实现有三种：CallBack会创建为Handler的私有变量Callback, handleMessage, handler.post(new Runnable)，三者的具体解释参考handler学习笔记。
4. 构建Handler handler = new Handler(handlerThread.getLooper);//具体的创建方式看使用场景。

## 注意
1. HandlerThread在不需要使用的时候需要手动的回收掉，因为其run方法是一个无限循环。