### EventBus源码解析：

​		**EventBus**是一个事件消息总线框架， 一般用于发送一些事件 通知到**Subscribers**，然后每个**Subscriber**去执行自己的Action（方法）；

​		EventBus的使用很简单， 总共就一下几个常用方法：

​		**EventBus.getDefault().register(this)**

​		**EventBus.getDefault().post(event)**

​		那我们现在就每个方法进去看一下源码（主要关注大概流程）：

### 		**EventBus.getDefault()**

```java
	static volatile EventBus defaultInstance;
	//单例模式（double check）返回EventBus对象
    public static EventBus getDefault() {
        EventBus instance = defaultInstance;
        if (instance == null) {
            synchronized (EventBus.class) {
                instance = EventBus.defaultInstance;
                if (instance == null) {
                    instance = EventBus.defaultInstance = new EventBus();
                }
            }
        }
        return instance;
    }
	
	//class对应的所有注解方法的包装对象（Subscription）
 	private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
	//EventBusBuilder是一个配置类 里面将一些 基础配置 如线程池 主线程包装类 主线程Handler等初始化好了
    private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();
    private static final Map<Class<?>, List<Class<?>>> eventTypesCache = new HashMap<>();

	//构造方法
	/**
     * Creates a new EventBus instance; each instance is a separate scope in which events are 				delivered. To use a
     * central bus, consider {@link #getDefault()}.
     */
	//源码的意思是  通过构造方法可以创建eventBus实例，每个实例都是一个event传递的集合， 如果想集中所有event， 可以考虑使用 getDefault() 方法获取到同一个单例对象
	 public EventBus() {
        this(DEFAULT_BUILDER);
    }

    EventBus(EventBusBuilder builder) {
        //主线程相关对象  里面持有主线程Looper对象 也是通过判断主线程的looper对象是否和当前线程Looper相等来确定是不是主线程
        mainThreadSupport = builder.getMainThreadSupport();
        //此处实际上是一个主线程handler
        mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
        //此处实际上是个runnable对象 可以放入到线程池中执行 用于切换到子线程
        backgroundPoster = new BackgroundPoster(this);
        //此处实际上是个runnable对象 可以放入到线程池中执行 用于切换到子线程
        asyncPoster = new AsyncPoster(this);
       	...
        //线程池    
        executorService = builder.executorService;    
    }
	
	
```

以上源码可以看出：EventBus.getDefault() 是通过单例模式返回一个EventBus对象，并在构造方法中做了一些初始化操作 ，  比如主线程包装类mainThreadSupport   builder对象  主线程handler  线程池 异步Runnable等

### **register(this)**

```java
 public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        //获取到该类所有 有Subscribe注解的方法
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);

        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                //将事件类型和注册有此事件的类 绑定起来  同时也将类和类中所有事件绑定起来
                subscribe(subscriber, subscriberMethod);
            }
        }
    }

 private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        Class<?> eventType = subscriberMethod.eventType;
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);

        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        //如果该事件的Subscription集合为空 则创建一个
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            //事件类型 和 subscriptions绑定起来
            //方便通过事件类型 来拿到Subscription
            subscriptionsByEventType.put(eventType, subscriptions);
        }

     	...
            
        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            //如果优先级别都一样 那么将newSubscription放到最后 如果优先级别不一致 按优先级别放置newSubscription的位置
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                //在这里添加Subscription
                subscriptions.add(i, newSubscription);
                break;
            }
        }

        
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            //事件集合
            subscribedEvents = new ArrayList<>();
            //每个class 所有的事件类型集合  和clazz 是绑定的
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);

       ...
    }
```

### **post(event)**

```java
 private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
        @Override
        protected PostingThreadState initialValue() {
            return new PostingThreadState();
        }
    };

public void post(Object event) {
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
    	//将事件添加到事件队列中（单链表结构）
        eventQueue.add(event);

        if (!postingState.isPosting) {
            //赋值当前是否是主线程
            postingState.isMainThread = isMainThread();
            postingState.isPosting = true;
           
            ...
            while (!eventQueue.isEmpty()) {
                //post Event
                postSingleEvent(eventQueue.remove(0), postingState);
            }
            
            ...
           
        }
    }

private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
       	...
            //继续往下走 
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
       ...
}

 private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            //拿到注册此事件的所有subscription（类）
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            //循环所有注册此事件的Subscription
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
            
                ...
            	//真正触发事件的方法
                postToSubscription(subscription, event, postingState.isMainThread);
           		...
        }
            ...
    }
     
      private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    //如果在主线程直接触发event
                    invokeSubscriber(subscription, event);
                } else {
                    //如果在子线程post(event) 则通过主线程的handler 将事件发送到主线程
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    //如果是在主线程  则通过将runnable放入到线程池中 以此来将事件切换到子线程去 并触发event 
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    //如果不是在主线程  直接触发event
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
          ...
        }
    }
    
     void invokeSubscriber(Subscription subscription, Object event) {
       	...
            //对应事件方法执行
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        ...
    }
     
    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            queue.enqueue(pendingPost);
           	...
            //线程池执行runnable对象    
            eventBus.getExecutorService().execute(this);
			
        }
    } 

     
```

总结：eventBus 更像是一个通过注解和反射 并通过运用handler  threadPool 来切换线程并通知消息的一个lib（仅个人理解，如有错误之处还请指出）；

#### EventBus中用到的基础知识：

#### 1.反射注解

#### 2.单例模式（double check）

#### 3.builder模式 （一个复杂对象一般都会用到  更像是配置类）

#### 4.handler 机制运用：

#### 1).通过线程中拿到的looper和主线looper对比 来判断是否是主线程 

#### 2).主线程handler在子线程发送消息到主线程

#### 3).ThreadLocal的运用

#### 5.线程池

#### 6.线程切换