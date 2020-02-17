### Android 消息机制详述

##### 概述

​		在Android中，Handler是Android消息机制的上层接口，这使得在开发过程只需要和Handler交互即可。通过Handler可以轻松的将一个任务切换到Handler所在的线程中去执行。一般我们在应用中认为Handler是用来更新UI，这的确没错，但更新UI只是Handler的一个特殊使用场景。实际上来说是因为Android中不允许在主线程中做耗时的任务，容易出现ANR，那么我们执行一些耗时任务的时候，比如I/O操作、读取文件、访问网络等，一般会把这些耗时的任务放在子线程中， 等耗时任务执行完毕后，我们需要做一些UI上的更新，但是Android规定不能在子线程中访问UI控件， 那我们这个时候就通过Handler发送消息来通知主线程更新UI。Handler的其他作用在源码级别都是有体现的，比如应用启动的时候通过Handler来发送相关Activity的生命周期相关消息等

​		Android的消息机制主要涉及到三个类 ：***Handler***  ***MessageQueue***  ***Looper*** 

​		Handler的运行需要底层MessageQueue和Looper的支撑，下面先简单介绍下这几个类的大致作用：

1. ​			**Handler** 用于发送消息到MessageQueue以及接收消息的回调；

2. ​            **MessageQueue**  翻译过来是消息队列，用来将接收到的消息放入到一个队列中去 并提供方法 供Looper轮询取消息

3. ​            **Looper** 是一个消息轮询器， 用于轮询消息队列中的消息，looper是和线程绑定的，存储在ThreadLocal当中， 主线程默认应用启动的时候已经帮我们初始化好了Looper 所以在主线程中我们默认能用Handler；

   

   ##### 详解

   ​	详细介绍Handler运行机制的时候，上面那几个类是必须的，当然还有与Looper相关的一个线程存储类ThreadLocal， 下面我们一一介绍：

   ##### 		ThreadLocal

   ​			ThreadLocal是一个线程内部的数据存储类， 通过它我们可以在指定线程中存储数据，数据存储后，我们就可以在指定的线程中获取到存储的数据了，其他线程则无法获取到数据；

   ​			ThreadLocal提供了set() 和get() 方法 一个是存储数据，一个是获取数据，其方法内部细节在此不作讨论，整体来说ThreadLocak实现了数据与线程的绑定，各个线程间获取数据互不干扰，存储和获取数据的时候都与当前线程有关

   ###### 	 	MessageQueue

   ​			消息队列主要提供两个操作：插入和读取，读取操作本身也伴随着删除操作，这两个操作分别对应着MessageQueue的两个方法：***enquequeMessage*** 和 ***next***  。MessageQueue虽然叫消息队列，但其内部实现适用的单链表结构，毕竟单链表在**插入**和**删除**比较有优势。

   ​			整体工作流程如下：Handler的post和send方法最终都会调用MessageQueue的***enquequeMessage*** 方法 将消息通过链表的插入操作放入到队列当中； MessageQueue的***next***方法是由Looper调用的， next的内部是一个无限循环方法，一直在轮询消息队列，如果有消息的话 就会返回消息交由Handler的handleMassage方法处理  如果没有消息就会一直阻塞在那里（这里类似于等待，并不会占用资源而导致线程卡死）；

   ​			具体的next源码如下：

   ```java
   //只包含核心源码
   
   //Looper的loop方法 实际是通过MessageQueue的next来获取 msg的 如果这个方法返回null 则意味着looper被停止了
   Message next() {
        	//这里返回null  意味着Looper被停止了 具体的方法实在 dispose() 方法中将mPtr置为0
           final long ptr = mPtr;
           if (ptr == 0) {
               return null;
           }
   
           int pendingIdleHandlerCount = -1; // -1 only during first iteration
           int nextPollTimeoutMillis = 0;
       	//一个无限循环
           for (;;) {
               if (nextPollTimeoutMillis != 0) {
                   Binder.flushPendingCommands();
               }
   			
               //native方法， 如果消息队列里无消息， 则会阻塞在这里；
               nativePollOnce(ptr, nextPollTimeoutMillis);
   
               synchronized (this) {
                   // Try to retrieve the next message.  Return if found.
                   final long now = SystemClock.uptimeMillis();
                   Message prevMsg = null;
                   Message msg = mMessages;
                   if (msg != null && msg.target == null) {
                       // Stalled by a barrier.  Find the next asynchronous message in the queue.
                       do {
                           prevMsg = msg;
                           msg = msg.next;
                       } while (msg != null && !msg.isAsynchronous());
                   }
                   if (msg != null) {
                       if (now < msg.when) {
                           // Next message is not ready.  Set a timeout to wake up when it is ready.
                           //如果msg是延时消息 而且现在还没到那个时间 那么我们就等它准备好的时候再唤醒它
                           nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                       } else {
                           // Got a message.
                           //获取到一个msg
                           mBlocked = false;
                           if (prevMsg != null) {
                               prevMsg.next = msg.next;
                           } else {
                               mMessages = msg.next;
                           }
                           msg.next = null;
                           if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                           msg.markInUse();
                           //返回我们拿到的msg
                           return msg;
                       }
                   } else {
                       // No more messages.
                       nextPollTimeoutMillis = -1;
                   }
   
                   // Process the quit message now that all pending messages have been handled.
                   //如果looper正在停止 则直接调用dispose方法 并返回null 在dispose方法中会将mPtr置为0
                   if (mQuitting) {
                       dispose();
                       return null;
                   }
   
                   ...
           }
       }
   
    	// Disposes of the underlying message queue.
       // Must only be called on the looper thread or the finalizer.
       private void dispose() {
           if (mPtr != 0) {
               nativeDestroy(mPtr);
               mPtr = 0;
           }
       }
```
   

   
##### 		Looper
   
​			介绍Looper的时候我们主要介绍Looper类的几个方法，等我们看完每个方法的源码后大致Looper的运行原理也就明白了：
   
##### 				构造方法
   
   ```java
   	private Looper(boolean quitAllowed) {
           //初始化对应的MeesageQueue
           mQueue = new MessageQueue(quitAllowed);
           //记录创建Looper的当前线程
           mThread = Thread.currentThread();
       }
```
   

   
##### 			prepare()
   
   ```java
   	public static void prepare() {
           //默认会调用prepare待参数的
           prepare(true);
       }
   	
   	//quitAllowed 指的是这个looper 是否可以手动暂停
   	//我们自己在子线程调用的prepare 是可以手动让looper停止并退出的
   	//ActivityThread中默认初始话的prepareMainLooper 是不需要手动停止的
       private static void prepare(boolean quitAllowed) {
           //表示单个线程只能有一个Looper对应来说也是只能有一个MessageQueue
           //如果我们在主线程中再次手动调用Looper.prepare()的话 会发生以下异常
           if (sThreadLocal.get() != null) {
               throw new RuntimeException("Only one Looper may be created per thread");
           }
           //将Looper存储在threadLocal当中
           sThreadLocal.set(new Looper(quitAllowed));
       }
   	
   	//应用启动的时候ActivityThread(主线程)中调用  用来初始化主线程的Looper 并让Looper开始loop 
       public static void prepareMainLooper() {
           	//此处标识 主线程的Looper不能手动停止（代码调用停止）
               prepare(false);
               synchronized (Looper.class) {
                   if (sMainLooper != null) {
                       throw new IllegalStateException("The main Looper has already been prepared.");
                   }
                   sMainLooper = myLooper();
               }
       }
```
   
##### 				loop()
   
   ```java
   public static void loop() {
       	//获取到当前线程的Looper
           final Looper me = myLooper();
       	//如果当前线程没有Looper 则抛出异常
           if (me == null) {
               throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
           }
           final MessageQueue queue = me.mQueue;
   
          	...
   
           for (;;) {
               //在这里可能会造成阻塞  因为MQ的next方法是一个无限循环方法去轮询消息，如果没有消息那么就可能一直在那边等待
               Message msg = queue.next(); // might block
               if (msg == null) {
                   // No message indicates that the message queue is quitting.
                   //如果msg为null  那么意味着looper  MQ正在停止
                   return;
               }
   
             	...
                   
               try {
                   //如果获取到msg  我们就会调用msg的target（这个target就是发送msg的那个handler）的dispatchMessage方法
                   msg.target.dispatchMessage(msg);
                   if (observer != null) {
                       observer.messageDispatched(token, msg);
                   }
                   dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
               } catch (Exception exception) {
                  ...
               } finally {
                  ...
               }
               ...
           }
       }
```
   
   

通过以上源码介绍，我们可以知道一下几个结论：

1. **每个线程只有一个Looper，也就是说只能调用一次prepare，多次调用会报错；**

2. **一个Looper对应一个MessageQueue， Looper的MessageQueue是在构造方法中初始化的；**

3. **Looper存储在ThreadLocal当中，并与当前线程绑定。**

4. **Looper通过loop方法来轮询消息，实际是通过MQ的next方法获取消息，得到消息后会通过*msg.target*拿到对应得handler 并交给handler去处理； 如果没有消息，则会一直阻塞等待。**

   

   ##### Handler

   Handler的post()和send()一系列方法 最终调的都是sendMessageDelayed方法，其源码如下：

   ```java
   //handler 默认构造方法
   public Handler(@Nullable Callback callback, boolean async) {
           
   	    ...
           //默认无参handler 对应的 looper 从当前线程对应的threadLocal中取looper
           mLooper = Looper.myLooper();
       	//如果当前没有looper 则直接报异常  这个也是默认handler初始化不能在子线程初始化
       	//因为子线程默认没有looper
           if (mLooper == null) {
               throw new RuntimeException(
                   "Can't create handler inside thread " + Thread.currentThread()
                           + " that has not called Looper.prepare()");
           }
           mQueue = mLooper.mQueue;
           mCallback = callback;
           mAsynchronous = async;
        	
        	
   } 
   public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
           if (delayMillis < 0) {
               delayMillis = 0;
           }
           return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
   }
   
   public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
           MessageQueue queue = mQueue;
           if (queue == null) {
               RuntimeException e = new RuntimeException(
                       this + " sendMessageAtTime() called with no mQueue");
               Log.w("Looper", e.getMessage(), e);
               return false;
           }
           return enqueueMessage(queue, msg, uptimeMillis);
   }
   
    private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
               long uptimeMillis) {
        	//赋值msg.target为当前Handler 供Looper中那到消息的时候调用handler的callback
           msg.target = this;
           msg.workSourceUid = ThreadLocalWorkSource.getUid();
   
           if (mAsynchronous) {
               msg.setAsynchronous(true);
           }
        	//调用MQ的enqueueMessage 将msg添加到消息队列中，并将msg.when属性赋值
           return queue.enqueueMessage(msg, uptimeMillis);
   }
   
   //MQ的enqueueMessage 方法 细节可以不用挖掘 大致就是将msg放入到MQ当中 或者如果是无延迟消息则直接触发
    boolean enqueueMessage(Message msg, long when) {
        
           ...
               
           synchronized (this) {
              ...
   
               msg.markInUse();
               msg.when = when;
               Message p = mMessages;
               boolean needWake;
               if (p == null || when == 0 || when < p.when) {
                   // New head, wake up the event queue if blocked.
                   msg.next = p;
                   mMessages = msg;
                   needWake = mBlocked;
               } else {
                   needWake = mBlocked && p.target == null && msg.isAsynchronous();
                   Message prev;
                   for (;;) {
                       prev = p;
                       p = p.next;
                    if (p == null || when < p.when) {
                           break;
                    }
                       if (needWake && p.isAsynchronous()) {
                           needWake = false;
                       }
                   }
                   msg.next = p; // invariant: p == prev.next
                   prev.next = msg;
               }
   
               // We can assume mPtr != 0 because mQuitting is false.
               if (needWake) {
                   nativeWake(mPtr);
               }
           }
           return true;
       }
   ```
   
   通过上面可以了解到Handler发送消息只是向MessageQueue当中插入一条消息，然后MessageQueue的next方法会将msg返回给Looper， Looper拿到消息后交给handler去处理，即调用handler的dispatchMessage方法，源码如下：
   
   ```java
    public void dispatchMessage(@NonNull Message msg) {
     	//如果msg的callback（实际上callback是一个Runnable对象就是我们post的runnable）不为null  则调用runnable的run方法；
           if (msg.callback != null) {
            handleCallback(msg);
           } else {
            //如果我们handler传递的callback参数不为null则调用mCallback.handleMessage
               if (mCallback != null) {
                   if (mCallback.handleMessage(msg)) {
                       return;
                   }
               }
               handleMessage(msg);
           }
   }
   
   private static void handleCallback(Message message) {
           //msg的callback对象是runnable类型
           message.callback.run();
   }
   ```
   
   从上面代码我们可以看出handler处理msg的流程为：先判断msg.callback是否为null 如果不为null则直接交给callback处理 否则判断mCallback是否为null 如果不为null 直接交给mCallback的handleMessage方法，最后如果都没调用则调用自己的handleMessage方法。
   
   到此基本Android的消息机制分析完毕，看完上面的一些源码分析，我们可以对每个类的职责一目了然。最后额外简单介绍一下主线程消息循环源码：
   
   ```java
   public static void main(String[] args) {
           Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
   
           // Install selective syscall interception
           AndroidOs.install();
   
           ...
               
           Process.setArgV0("<pre-initialized>");
   		//准备主线程的Looper
           Looper.prepareMainLooper();
   
           //主线程的创建
        ActivityThread thread = new ActivityThread();
           thread.attach(false, startSeq);
   		
       	//主线程的Handler即activityThread.H类 里面定义了一组消息常量 主要包含了四大组件的生命周期回调
           if (sMainThreadHandler == null) {
               sMainThreadHandler = thread.getHandler();
           }
   
           if (false) {
               Looper.myLooper().setMessageLogging(new
                       LogPrinter(Log.DEBUG, "ActivityThread"));
           }
   
          	//Looper开始循环
           Looper.loop();
   
           ...
       }
   ```
   
   **最后要感谢任玉刚老师的《Android开发艺术探索》指导！**