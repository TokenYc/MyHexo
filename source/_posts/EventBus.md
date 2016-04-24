title: 研究下EventBus
---
EventBus用法很简单，但是想想他是怎么实现各个组件之间的通信的呢？是如何在不同的线程中实现调用的呢？不是很明确呀，那就看看源码吧。

#### 涉及的东西

 - EventBus的作用
 - EventBus的使用方法
 - EventBus实现原理，结合源码解析。


为什么要写作用和使用方法？这些不是官方文档上都有么。我感觉过一遍可以帮助理解源码，并且可能get到不容易注意到的功能，so。


### EventBus的作用

- simplifies the communication between components
- decouples event senders and receivers
- performs well with Activities, Fragments, and background threads
- avoids complex and error-prone dependencies and life cycle issues
- 简化组件之间的通信。
- 解耦信息**发送者**和**接收者**。
- 在activity，fragment，异步线程中表型的阔以。
- 避免复杂容易出错的依赖和生命周期问题。

**总结一下**：只要是个对象就可以使用，使用起来方便，分成发送者和订阅者，事件作为信息的载体，可以灵活地在各个线程中使用。


### EventBus的使用方法
EventBus.getDefault()可以获取默认的EventBus单例。
也可以通过new EventBus（builder）后自己维护一个单例。
![盗下官方的图 (￣_,￣ )](http://7xpp4m.com1.z0.glb.clouddn.com/blog6454.tmp.png)
#### 定义Event

	public class MessageEvent {
	    public final String message;
	
	    public MessageEvent(String message) {
	        this.message = message;
	    }
	}

要求就是xxxxEvent这种格式。

#### 在订阅类里
以activity为例

 	@Override
    public void onStart() {
        super.onStart();
        EventBus.getDefault().register(this);
	//EventBus.getDefault().registerSticky(this);
    }

    @Override
    public void onStop() {
        EventBus.getDefault().unregister(this);
        super.onStop();
    }

    // This method will be called when a MessageEvent is posted
    public void onEvent(MessageEvent event){
        Toast.makeText(getActivity(), event.message, Toast.LENGTH_SHORT).show();
    }
	//下面4种回调线程选择一个，回调线程不同，方法名不同。	
    // 在当前线程回调 
    public void onEvent(YourEvent event){
        doSomethingWith(event);
    }
	
	//在主线程中回调
	public void onEventMainThread(YourEvent event){
		doSomething();
	}

	//在如果是在异步线程中post，直接在异步线程中接受事件。如果在MainThread中post，会在EventBus维护的一个线程中按顺序回调。总之就是不在MainThread中回调
	public void onEventBackgroundThread(YourEvent event){
		doSomething();
	}

	//总是在EventBus维护的线程池中回调
	public void onEventAsyncThread(YourEvent event){
		doSomething();
	}


#### 发送事件


	EventBus.getDefault().post(new MessageEvent("Hello everyone!"));
	//EventBus.getDefault().postSticky(new MessageEvent("Hello everyone!"));
	//在post之后注册的订阅类也可以接收到这个event。

### EventBus源码解析

看源码之前先自己猜一下是怎么实现的。订阅者，发送者，EventBus。
 
- 订阅时register(this)，把这个对象传了进去，那应该是要用反射找到里面的onEvent方法，然后保存起来。到时候别的地方post的时候再去调用。
- post(YourEvent event)，应该是要去找订阅了这个Evnet的类，然后调用它的方法，里面会有很多反射的操作
- unRegister(this)，那就是取消订阅，从保存它的地方去掉。

大致的思路应该是这样的，到底是怎么实现的，如何在不同的线程里回调，就要看源码了。

先去找register方法

	private synchronized void register(Object subscriber, boolean sticky, int priority) {
	        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriber.getClass());
	        for (SubscriberMethod subscriberMethod : subscriberMethods) {
	            subscribe(subscriber, subscriberMethod, sticky, priority);
	        }
	    }
这里看到了一个方法，看名字应该是去找订阅的方法的，SubscriberMethod应该是个Entity
看一下
	
	final class SubscriberMethod {
	    final Method method;
	    final ThreadMode threadMode;
	    final Class<?> eventType;
	    /** Used for efficient comparison */
	    String methodString;
	
	    SubscriberMethod(Method method, ThreadMode threadMode, Class<?> eventType) {
	        this.method = method;
	        this.threadMode = threadMode;
	        this.eventType = eventType;
	    }

果然是的。里面的method看样子就是要调用的那个onEvent(YourEvent event)方法。threadMode应该就是调用的线程。eventType就是里面那个参数的类吧。到底是不是这样继续往下看。看下那个findSubscriberMethods方法就行了。长的一笔。

	List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
	        String key = subscriberClass.getName();
	        List<SubscriberMethod> subscriberMethods;
	        synchronized (methodCache) {
	            subscriberMethods = methodCache.get(key);
	        }
	        if (subscriberMethods != null) {
	            return subscriberMethods;
	        }
	        subscriberMethods = new ArrayList<SubscriberMethod>();
	        Class<?> clazz = subscriberClass;
	        HashSet<String> eventTypesFound = new HashSet<String>();
	        StringBuilder methodKeyBuilder = new StringBuilder();
	        while (clazz != null) {
	            String name = clazz.getName();
	            if (name.startsWith("java.") || name.startsWith("javax.") || name.startsWith("android.")) {
	                // Skip system classes, this just degrades performance
	                break;
	            }
	
	            // Starting with EventBus 2.2 we enforced methods to be public (might change with annotations again)
	            Method[] methods = clazz.getDeclaredMethods();
	            for (Method method : methods) {
	                String methodName = method.getName();
	                if (methodName.startsWith(ON_EVENT_METHOD_NAME)) {
	                    int modifiers = method.getModifiers();
	                    if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
	                        Class<?>[] parameterTypes = method.getParameterTypes();
	                        if (parameterTypes.length == 1) {
	                            String modifierString = methodName.substring(ON_EVENT_METHOD_NAME.length());
	                            ThreadMode threadMode;
	                            if (modifierString.length() == 0) {
	                                threadMode = ThreadMode.PostThread;
	                            } else if (modifierString.equals("MainThread")) {
	                                threadMode = ThreadMode.MainThread;
	                            } else if (modifierString.equals("BackgroundThread")) {
	                                threadMode = ThreadMode.BackgroundThread;
	                            } else if (modifierString.equals("Async")) {
	                                threadMode = ThreadMode.Async;
	                            } else {
	                                if (skipMethodVerificationForClasses.containsKey(clazz)) {
	                                    continue;
	                                } else {
	                                    throw new EventBusException("Illegal onEvent method, check for typos: " + method);
	                                }
	                            }
	                            Class<?> eventType = parameterTypes[0];
	                            methodKeyBuilder.setLength(0);
	                            methodKeyBuilder.append(methodName);
	                            methodKeyBuilder.append('>').append(eventType.getName());
	                            String methodKey = methodKeyBuilder.toString();
	                            if (eventTypesFound.add(methodKey)) {
	                                // Only add if not already found in a sub class
	-------------------------------------------------------------------------
	                                subscriberMethods.add(new SubscriberMethod(method, threadMode, eventType));
 	-------------------------------------------------------------------------
	                            }
	                        }
	                    } else if (!skipMethodVerificationForClasses.containsKey(clazz)) {
	                        Log.d(EventBus.TAG, "Skipping method (not public, static or abstract): " + clazz + "."
	                                + methodName);
	                    }
	                }
	            }
	            clazz = clazz.getSuperclass();
	        }
			if (subscriberMethods.isEmpty()) {
	            throw new EventBusException("Subscriber " + subscriberClass + " has no public methods called "
	                    + ON_EVENT_METHOD_NAME);
	        } else {
	            synchronized (methodCache) {
	                methodCache.put(key, subscriberMethods);
	            }
	            return subscriberMethods;
	        }
	    }
暂时不去深究代码细节，反射就够写一大堆了。大致就是通过反射，对onEvent方法解析，得到需要的东西。是啥呢，就是上面那个SubscriberMethod的一个List。

然后看下register中调用的subscribe方法。

直接找里面我们想看到的东西。那就是怎么存这些对象和方法的。

	subscriptionsByEventType.put(eventType, subscriptions);

	typesBySubscriber.put(subscriber, subscribedEvents);
在里面找到了这个。分别是按EventType和Subscriber来存储。
这两个东西是啥，到上面找。

	private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
	private final Map<Object, List<Class<?>>> typesBySubscriber;

是两个Map。这两个Map存储了我们所有的订阅信息。
好了，到这里为止，是整个订阅的过程和存储订阅的过程。反射的部分就自己看看吧 (￣_,￣ )

---

然后是post的过程。
这里再先理一下思路。已经知道EventBus是怎么存那些订阅信息了。现在post一个Event，那可以根据这个去存储的地方去找，然后调用他们。

	void invokeSubscriber(Subscription subscription, Object event) {
	        try {
	            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
	        } catch (InvocationTargetException e) {
	            handleSubscriberException(subscription, event, e.getCause());
	        } catch (IllegalAccessException e) {
	            throw new IllegalStateException("Unexpected exception", e);
	        }
	    }
问题是怎么在不同的线程中调用。但是不管在哪个线程调用，最后都要执行这个方法。还是反射。

接下来看如何在不同的线程中调用。

从post方法开始一步一步找，找到最后看到这个方法。

	private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case PostThread:
                invokeSubscriber(subscription, event);
                break;
            case MainThread:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case BackgroundThread:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case Async:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }

这里对应了4中线程模式。就是一开始说的那4中。
第一种，PostThread，即在当前线程调用。直接调用了上面那个invokeSubscriber()。
第二种，MainThread，如果在主线程，直接调用invokeSubscriber()。如果不在主线程，调用

	mainThreadPoster = new HandlerPoster(this, Looper.getMainLooper(), 10);
	final class HandlerPoster extends Handler{}
HandlerPoster继承了Handler，这里再构造函数里传入主线程的Looper，实际上就是构造了一个主线程的Handler。到这里就明白了，如何在主线程调用onEvent方法。如果不懂可以看下
[http://blog.csdn.net/lmj623565791/article/details/38377229](http://blog.csdn.net/lmj623565791/article/details/38377229 "鸿杨的Message，Looper，Handler解析")

第三种，backgroundThread，如果不在主线程，直接调用即可，如果在主线程，要放到异步线程里执行。
	
	private final BackgroundPoster backgroundPoster;
	backgroundPoster = new BackgroundPoster(this);

看下BackgroundPoster

	final class BackgroundPoster implements Runnable {

    private final PendingPostQueue queue;
    private final EventBus eventBus;

    private volatile boolean executorRunning;

    BackgroundPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            queue.enqueue(pendingPost);
            if (!executorRunning) {
                executorRunning = true;
	--------------------------------------------------------------
                eventBus.getExecutorService().execute(this);
	--------------------------------------------------------------
            }
        }
    }

    @Override
    public void run() {
        try {
            try {
                while (true) {
                    PendingPost pendingPost = queue.poll(1000);
                    if (pendingPost == null) {
                        synchronized (this) {
                            // Check again, this time in synchronized
                            pendingPost = queue.poll();
                            if (pendingPost == null) {
                                executorRunning = false;
                                return;
                            }
                        }
                    }
                    eventBus.invokeSubscriber(pendingPost);
                }
            } catch (InterruptedException e) {
                Log.w("Event", Thread.currentThread().getName() + " was interruppted", e);
            }
        } finally {
            executorRunning = false;
        }
    	}
	}	
这个类实现了runnnable接口，标出的那那一行比较关键。从线程池中取出一个线程，把任务放进去执行。这个线程池是BusEvent自己维护的一个线程池，并且，AsyncThread也是用的这个线程池。

	class AsyncPoster implements Runnable {

    private final PendingPostQueue queue;
    private final EventBus eventBus;

    AsyncPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        queue.enqueue(pendingPost);
        eventBus.getExecutorService().execute(this);
    }

    @Override
    public void run() {
        PendingPost pendingPost = queue.poll();
        if(pendingPost == null) {
            throw new IllegalStateException("No pending post available");
        }
	------------------------------------------------------------------
        eventBus.invokeSubscriber(pendingPost);
	------------------------------------------------------------------
    }
	}

和backgroundThread类似。他们最后都调用了invokeSubscriber方法。

到这就结束了。
sticky和priority都没涉及。因为感觉大思路有了，这些都比较好看懂了。
分析过程中没有涉及代码细节，而是带着目的去找应该存在的东西。之前有过漫无目的的读源码，可以说是没有任何收获。还是先思考一下整体思路为好。至于代码细节，每一处都值得另外推敲，可以另开一个blog。

---
OVER