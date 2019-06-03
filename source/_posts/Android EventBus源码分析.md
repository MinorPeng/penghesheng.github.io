---
title: Android——EventBus源码分析
tag: Android第三方框架
date: 2019-04-30
---

[TOC]

# EventBus3.0 简单分析

## 获取单例

首先我们看一下使用

```java
EventBus.getDefault().register(this);
```

我们通过这样来注册一个订阅者

`getDefualt()`就是获取一个EventBus的单例

```java
/** Convenience singleton for apps using a process-wide EventBus instance. */
   public static EventBus getDefault() {
       if (defaultInstance == null) {
           synchronized (EventBus.class) {
               if (defaultInstance == null) {
                   defaultInstance = new EventBus();
               }
           }
       }
       return defaultInstance;
   }
```

很明显，就是通过DCL的形式来获取一个单例

然后接着看看EventBus实例化中做了什么事吧

```java
private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();

public EventBus() {
       this(DEFAULT_BUILDER);
   }

   EventBus(EventBusBuilder builder) {
       //EventBus特用的log
       logger = builder.getLogger();
       subscriptionsByEventType = new HashMap<>();
       typesBySubscriber = new HashMap<>();
       stickyEvents = new ConcurrentHashMap<>();
       mainThreadSupport = builder.getMainThreadSupport();
       mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
       backgroundPoster = new BackgroundPoster(this);
       asyncPoster = new AsyncPoster(this);
       indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
       subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
               builder.strictMethodVerification, builder.ignoreGeneratedIndex);
       logSubscriberExceptions = builder.logSubscriberExceptions;
       logNoSubscriberMessages = builder.logNoSubscriberMessages;
       sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
       sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
       throwSubscriberException = builder.throwSubscriberException;
       eventInheritance = builder.eventInheritance;
       executorService = builder.executorService;
   }
```

通过默认的EventBusBuilder来进行初始化，就是一个Builder模式，DEFAULT_BUILDER的实例化就是一个空实现

然后我们看到在具体初始化中，初始化了很多参数，这些参数需要好好看一下

- logger：就是EventBus用的log
- subscriptionsByEventType：一个Map集合，具体类型是`Map<Class<?>, CopyOnWriteArrayList<Subscription>>`，也就是这个Map中存放的是以Class为key，List为value，这里的key其实就是事件类型，后面会分析到，CopyOnWriteArrayList其实就是一个Subscription的list，而这个Subscription是一个包含订阅者和其对应的方法的一个类
- typesBySubscriber：也是一个Map集合，具体类型是`Map<Object, List<Class<?>>>`，以订阅者为key，订阅者订阅的所有事件为value，这个后面也会讲到
- stickyEvents：一个线程安全的Map集合，具体是`Map<Class<?>, Object>`，粘性事件的缓存
- mainThreadSupport：MainThreadSupport，根据主线程Looper创建的一个类，还包含了EventBus发射器Poster
- mainThreadPoster：处于主线程的发射器，通过MainThreadSupport直接创建的
- backgroundPoster：同步发射器
- asyncPoster：异步发射器
- subscriberMethodFinder：订阅者方法查找类，根据订阅者的类信息来查找里面订阅方法，注册的时候会讲到
- executorService：线程池

好，这里就大概知道这么多，后面注册的时候再仔细了解

## 注册

接下来看看register方法

```java
public void register(Object subscriber) {
       Class<?> subscriberClass = subscriber.getClass();
       List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
       synchronized (this) {
           for (SubscriberMethod subscriberMethod : subscriberMethods) {
               subscribe(subscriber, subscriberMethod);
           }
       }
   }
```

使用的时候，这个subscriber是我们传递的Activity，然后通过`getClass()`来获取Class，接着subscriberMethodFinder来查找当前subscriber的所有订阅方法，先看看这个查找吧

```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
       List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
       if (subscriberMethods != null) {
           return subscriberMethods;
       }

       if (ignoreGeneratedIndex) {
           subscriberMethods = findUsingReflection(subscriberClass);
       } else {
           subscriberMethods = findUsingInfo(subscriberClass);
       }
       if (subscriberMethods.isEmpty()) {
           throw new EventBusException("Subscriber " + subscriberClass
                   + " and its super classes have no public methods with the @Subscribe annotation");
       } else {
           METHOD_CACHE.put(subscriberClass, subscriberMethods);
           return subscriberMethods;
       }
   }
```

`METHOD_CACHE`是`Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();`这样定义的，说明它是一个线程安全的集合，同时我们可以看到，这是SubscriberMethodFinder类内部维持的一个集合，就是一个subscriber和subscriberMethods的关系集合，说得明白点就是订阅者和其订阅方法的Map

在上面这个方法中，先是从缓存中看能不能获取到这个订阅者相应的订阅方法，有的话直接返回；没有的话，就接着看后面的代码

先是一个`ignoreGeneratedIndex`（是否忽略生成index）的一个判断，这个属性是前面实例化`SubscriberMethodFinder`时传入的，也是EventBus初始化时传入的，由于我们默认的EventBusBuilder是一个空实现，所以这里就是默认值false；这里就是来决定获取订阅者的方法是怎样来获取的，我们两个都看看吧；在获取到订阅方法后，会放入`METHOD_CACHE`中

### findUsingReflection（自定义注解器，索引）

当ignoreGeneratedIndex为true时，执行的是`findUsingReflection(subscriberClass)`，通过反射来获取

```java
private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
       //初始化FindState对象
       FindState findState = prepareFindState();
       //关联订阅者
       findState.initForSubscriber(subscriberClass);
       while (findState.clazz != null) {
           //在单个类中查找方法并放在findState中
           findUsingReflectionInSingleClass(findState);
           //查找父类
           findState.moveToSuperclass();
       }
       //返回订阅者及其父类的方法
       return getMethodsAndRelease(findState);
   }
```

首先会初始化一个FindState对象（通过`prepareFindState()`会先在缓存中查找，没有的话会直接new一个），这个类是SubscriberMethodFinder的一个内部类，就是用来辅助我们查找订阅方法的，先简单看看它的属性吧

```java
static class FindState {
       final List<SubscriberMethod> subscriberMethods = new ArrayList<>();
       final Map<Class, Object> anyMethodByEventType = new HashMap<>();
       final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();
       final StringBuilder methodKeyBuilder = new StringBuilder(128);

       Class<?> subscriberClass;
       Class<?> clazz;
       boolean skipSuperClasses;
       SubscriberInfo subscriberInfo;
}
```

可见，这个类里面的属性有很多，包含了订阅方法、根据事件类型来存储的方法等

回到之前，初始化FindState对象后，然后通过`findState.initForSubscriber(subscriberClass)`关联上订阅者SubscriberClass，接着通过判定`findState.clazz`是否为空来结束查找（有点链表的思想在里面），接着通过`findUsingReflectionInSingleClass(findState)`来进行具体的查找，查找结果就会存储在FindState中

```java
private void findUsingReflectionInSingleClass(FindState findState) {
       Method[] methods;
       try {
           //通过getDeclaredMethods获取方法，这会比getMethods快很多，特别是订阅者是Activity的时候
           methods = findState.clazz.getDeclaredMethods();
       } catch (Throwable th) {
           //当通过getDeclaredMethods找不到的时候再通过getMethods来获取
           methods = findState.clazz.getMethods();
           findState.skipSuperClasses = true;
       }
       //遍历所有的方法
       for (Method method : methods) {
           int modifiers = method.getModifiers();
           //忽略掉非public和static方法
           if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
               //获取参数
               Class<?>[] parameterTypes = method.getParameterTypes();
               //订阅者的方法参数只能是一个，就是传递的事件
               if (parameterTypes.length == 1) {
                   //获取注解
                   Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                   if (subscribeAnnotation != null) {
                       Class<?> eventType = parameterTypes[0];
                        // 检查eventType决定是否订阅，通常订阅者不能有多个eventType相同的订阅方法
                       if (findState.checkAdd(method, eventType)) {
                           //线程模式
                           ThreadMode threadMode = subscribeAnnotation.threadMode();
                           //添加到findState对象中的list中
                           findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                   subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                       }
                   }
               } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                   String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                   throw new EventBusException("@Subscribe method " + methodName +
                           "must have exactly 1 parameter but has " + parameterTypes.length);
               }
           } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
               String methodName = method.getDeclaringClass().getName() + "." + method.getName();
               throw new EventBusException(methodName +
                       " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
           }
       }
   }
```

直接通过Class来获取所有的方法，然后对订阅者的所有方法进行一个过滤，将订阅方法筛选出来，保存到FindState对象中，具体的过滤就是根据订阅方法的一些特性：比如说只能是public修饰、只能有一个参数（事件）、注解等

接着回到`findUsingReflection()`方法，通过`findUsingReflectionInSingleClass(findState)`将SubscriberClass中所有的对象保存到FindState中后，通过`findState.moveToSuperclass()`查找订阅者的父类，以同样的方式去查找是否有订阅方法，最后通过`getMethodsAndRelease(findState)`返回所有的订阅方法，并添加到缓存中

```java
private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
       List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
       findState.recycle();
       synchronized (FIND_STATE_POOL) {
           for (int i = 0; i < POOL_SIZE; i++) {
               if (FIND_STATE_POOL[i] == null) {
                   FIND_STATE_POOL[i] = findState;
                   break;
               }
           }
       }
       return subscriberMethods;
   }
```

### findUsingInfo（没有自定义注解器，无索引）

接着分析当ignoreGeneratedIndex为false，执行的是`findUsingInfo(subscriberClass)`

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
       //同样会准备一个FindState对象，辅助查找
       FindState findState = prepareFindState();
       findState.initForSubscriber(subscriberClass);
       while (findState.clazz != null) {
           //获取对应的SubscriberInfo
           findState.subscriberInfo = getSubscriberInfo(findState);
           if (findState.subscriberInfo != null) {
               //通过SubscriberInfo获取订阅方法
               SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
               for (SubscriberMethod subscriberMethod : array) {
                   if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                       findState.subscriberMethods.add(subscriberMethod);
                   }
               }
           } else {
               //通过反射获取
               findUsingReflectionInSingleClass(findState);
           }
           //查找父类
           findState.moveToSuperclass();
       }
       return getMethodsAndRelease(findState);
   }
```

我们可以看到，这个获取订阅方法的方式是前面通过反射获取是类似的，都是借助了FindState这个对象；在具体查找的时候，先是通过SubscriberInfo是否为null来决定具体的查找方式，如果为null最后还是调用的反射去获取订阅方法，如果不为null，则是通过SubscriberInfo直接获取订阅方法

我们先看看SubscriberInfo是怎么来的吧

```java
private SubscriberInfo getSubscriberInfo(FindState findState) {
       if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
           SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
           if (findState.clazz == superclassInfo.getSubscriberClass()) {
               return superclassInfo;
           }
       }
       if (subscriberInfoIndexes != null) {
           for (SubscriberInfoIndex index : subscriberInfoIndexes) {
               SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
               if (info != null) {
                   return info;
               }
           }
       }
       return null;
   }
```

首先通过FindState中拿到SubscriberInfo，但是如果之前缓存中没有FindState，FindState就是通过new的，那么FindState中的SubscriberInfo就为null

那么接着就判断subscriberInfoIndexes，这个是我们在创建SubscriberMethodFinder时传进来的，也就是在获取EventBus单例时初始化的，具体就是EventBusBuilder中设置的

```java
/** Adds an index generated by EventBus' annotation preprocessor. */
public EventBusBuilder addIndex(SubscriberInfoIndex index) {
       if (subscriberInfoIndexes == null) {
           subscriberInfoIndexes = new ArrayList<>();
       }
       subscriberInfoIndexes.add(index);
       return this;
   }
```

这就是一个list，而且是通过EventBus的注解器生成的，具体自己就去了解一下吧，官方推荐使用这种方式哦

总之在`findUsingInfo()`中，如果有注解器生成的SubscriberInfoIndex，那么就可以直接通过SubscriberInfo获取到所有的订阅方法，默认是没有的，那么就会通过反射的方式去获取所有的订阅方法

## 订阅

回到最开始，register方法，获取到所有的订阅方法后，就会进行订阅

```java
public void register(Object subscriber) {
       Class<?> subscriberClass = subscriber.getClass();
       List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
       synchronized (this) {
           //通过遍历所有订阅方法进行订阅
           for (SubscriberMethod subscriberMethod : subscriberMethods) {
               subscribe(subscriber, subscriberMethod);
           }
       }
   }
```

通过遍历订阅方法，然后进行订阅

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
       //事件类型 
       Class<?> eventType = subscriberMethod.eventType;
       Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
       CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
       if (subscriptions == null) {
           subscriptions = new CopyOnWriteArrayList<>();
           subscriptionsByEventType.put(eventType, subscriptions);
       } else {
           if (subscriptions.contains(newSubscription)) {
               throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                       + eventType);
           }
       }

       int size = subscriptions.size();
       for (int i = 0; i <= size; i++) {
           if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
               subscriptions.add(i, newSubscription);
               break;
           }
       }

       List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
       if (subscribedEvents == null) {
           subscribedEvents = new ArrayList<>();
           typesBySubscriber.put(subscriber, subscribedEvents);
       }
       subscribedEvents.add(eventType);
	//粘性事件的处理
       if (subscriberMethod.sticky) {
           if (eventInheritance) {
               Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
               for (Map.Entry<Class<?>, Object> entry : entries) {
                   Class<?> candidateEventType = entry.getKey();
                   if (eventType.isAssignableFrom(candidateEventType)) {
                       Object stickyEvent = entry.getValue();
                       checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                   }
               }
           } else {
               Object stickyEvent = stickyEvents.get(eventType);
               checkPostStickyEventToSubscription(newSubscription, stickyEvent);
           }
       }
   }
```

首先获取到事件类型eventType，接着将subscriber和subscriberMethod组装成Subscription；接着从`subscriptionsByEventType`中根据事件类型获取到所有的Subscription，这个`subscriptionsByEventType`就是我们前面EventBus初始化讲到的根据事件类型为key和subscriptions为value的HashMap，这就是维护的一个总线缓存；先是根据事件类型获取对应的Subscription列表，没有就会new一个，然后将新的subscription存进去，如果`subscriptionsByEventType`中已经有了相同的Subscription，那么就会抛出异常（当你重复注册一个订阅者的时候就会抛出，反正我是遇到了）

如果相同的EventTyoe对应的Subscriptions没有包含这个新的Subscription，那么就会遍历该EventType下的所有Subscription，根据订阅方法的优先级插入到这个subscriptions中

接着`typesBySubscriber.get(subscriber)`根据订阅者获取了所有的订阅事件，前面我们EventBus初始化的时候，typesBySubscriber就是一个HashMap，从这里就可以知道它真的是以订阅者为key，订阅者订阅的所有事件为value；这里根据订阅者subscriber取出了所有的订阅事件，同样，如果这个list不存在就会new一个，并将这个新的事件类型放进去

到这里，一般的事件，就完成了订阅

后面就是粘性事件的处理了，eventInheritance就是是否继承的事件，也是在EventBus实例化的时候传进的，最后通过`checkPostStickyEventToSubscription()`来发送粘性事件，这里就暂且不分析了，我们回到主要的一般事件就ok了

## 发送

一般事件的发送

```java
EventBus.getDefault().post(new ObjectEvent);
```

还是先获取到EventBus的单例，这里开头分析过了，所以发送的时候就是直接获取到之前的单例了

直接看post方法吧

```java
/** Posts the given event to the event bus. */
   public void post(Object event) {
       //获取当前线程的posting状态
       PostingThreadState postingState = currentPostingThreadState.get();
       //获取事件队列
       List<Object> eventQueue = postingState.eventQueue;
       eventQueue.add(event);
       if (!postingState.isPosting) {
           postingState.isMainThread = isMainThread();
           postingState.isPosting = true;
           if (postingState.canceled) {
               throw new EventBusException("Internal error. Abort state was not reset");
           }
           try {
               while (!eventQueue.isEmpty()) {
                   postSingleEvent(eventQueue.remove(0), postingState);
               }
           } finally {
               postingState.isPosting = false;
               postingState.isMainThread = false;
           }
       }
   }
```

先是根据ThreadLocal获取到了当前线程的posting状态，接着获取当前线程的队列，将事件添加到队列中，当前线程没有正在发送事件时，设置发送状态，然后通过遍历队列，通过`postSingleEvent(eventQueue.remove(0), postingState);`来发送事件

```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
       Class<?> eventClass = event.getClass();
       boolean subscriptionFound = false;
       if (eventInheritance) {
           List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
           int countTypes = eventTypes.size();
           for (int h = 0; h < countTypes; h++) {
               Class<?> clazz = eventTypes.get(h);
               subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
           }
       } else {
           subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
       }
       //如果没有找到对应的Subscription，那么就是没有订阅者的事件，就会发送NoSubscriberEvent
       if (!subscriptionFound) {
           if (logNoSubscriberMessages) {
               logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
           }
           if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                   eventClass != SubscriberExceptionEvent.class) {
               //发送规定的NoSubscriberEvent，后面的过程跟现在的差不多
               post(new NoSubscriberEvent(this, event));
           }
       }
   }
```

首先看是否开启了事件继承，如果是（`eventInheritance == true`），则会找出发布事件的所有父类`lookupAllEventTypes(eventClass)`，然后遍历每个事件类型类型发布

```java
/** Looks up all Class objects including super classes and interfaces. Should also work for interfaces. */
   private static List<Class<?>> lookupAllEventTypes(Class<?> eventClass) {
       synchronized (eventTypesCache) {
           List<Class<?>> eventTypes = eventTypesCache.get(eventClass);
           if (eventTypes == null) {
               eventTypes = new ArrayList<>();
               Class<?> clazz = eventClass;
               while (clazz != null) {
                   eventTypes.add(clazz);
                   addInterfaces(eventTypes, clazz.getInterfaces());
                   clazz = clazz.getSuperclass();
               }
               eventTypesCache.put(eventClass, eventTypes);
           }
           return eventTypes;
       }
   }
```

查找父类所有的使劲按类型，并且放入缓存eventTypesCache中

如果不是（`eventInheritance == false`），则直接通过`postSingleEventForEventType()`发送事件

```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
      CopyOnWriteArrayList<Subscription> subscriptions;
      synchronized (this) {
          //取出Subscriptions
          subscriptions = subscriptionsByEventType.get(eventClass);
      }
      if (subscriptions != null && !subscriptions.isEmpty()) {
          for (Subscription subscription : subscriptions) {
              postingState.event = event;
              postingState.subscription = subscription;
              boolean aborted = false;
              try {
                  postToSubscription(subscription, event, postingState.isMainThread);
                  aborted = postingState.canceled;
              } finally {
                  postingState.event = null;
                  postingState.subscription = null;
                  postingState.canceled = false;
              }
              if (aborted) {
                  break;
              }
          }
          return true;
      }
      return false;
  }
```

根据事件类型取出Subscriptions，遍历所有的Subscription，将之添加到PostingThreadState中，通过`postToSubscription(subscription, event, postingState.isMainThread);`发送出去

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
      switch (subscription.subscriberMethod.threadMode) {
          case POSTING:
              invokeSubscriber(subscription, event);
              break;
          case MAIN:
              if (isMainThread) {
                  invokeSubscriber(subscription, event);
              } else {
                  mainThreadPoster.enqueue(subscription, event);
              }
              break;
          case MAIN_ORDERED:
              if (mainThreadPoster != null) {
                  mainThreadPoster.enqueue(subscription, event);
              } else {
                  // temporary: technically not correct as poster not decoupled from subscriber
                  invokeSubscriber(subscription, event);
              }
              break;
          case BACKGROUND:
              if (isMainThread) {
                  backgroundPoster.enqueue(subscription, event);
              } else {
                  invokeSubscriber(subscription, event);
              }
              break;
          case ASYNC:
              asyncPoster.enqueue(subscription, event);
              break;
          default:
              throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
      }
  }
```

根据不同的线程模式，进行对应的转发

- POSTING：事件在哪种线程发送，就在哪种线程订阅
- MAIN：无论事件在什么线程发送，都在主线程接收
- BACKGROUND：如果发送的线程不是主线程，则在该线程订阅，如果是主线程发送，则使用一个单独的后台线程订阅
- ASYNC：在非主线程和发布线程种订阅

不管是哪种线程模式，都会调用`invokeSubscriber(subscription, event);`发送到对应的订阅者那里

## 接收

接着就看看接收吧

```java
void invokeSubscriber(Subscription subscription, Object event) {
       try {
           subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
       } catch (InvocationTargetException e) {
           handleSubscriberException(subscription, event, e.getCause());
       } catch (IllegalAccessException e) {
           throw new IllegalStateException("Unexpected exception", e);
       }
   }
```

在这里，在Subscription种取出订阅方法SubscriberMethod，然后进行订阅（通过Java API直接调用），事件就发送到我们的订阅方法中了

## 取消注册

通常我们通过如下来取消注册

```java
EventBus.getDefault().unregister(this);
```

直接看unregister方法吧

```java
/** Unregisters the given subscriber from all event classes. */
   public synchronized void unregister(Object subscriber) {
       List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
       if (subscribedTypes != null) {
           for (Class<?> eventType : subscribedTypes) {
               unsubscribeByEventType(subscriber, eventType);
           }
           typesBySubscriber.remove(subscriber);
       } else {
           logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
       }
   }
```

通过订阅者（待取消订阅的订阅者）从缓存中获取所有的订阅的事件类型，遍历事件类型，进行取消订阅，并从缓存中移除该订阅者的相关订阅事件类型

接着看看`unsubscribeByEventType(subscriber, eventType);`取消订阅吧

```java
/** Only updates subscriptionsByEventType, not typesBySubscriber! Caller must update typesBySubscriber. */
   private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
       List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
       if (subscriptions != null) {
           int size = subscriptions.size();
           for (int i = 0; i < size; i++) {
               Subscription subscription = subscriptions.get(i);
               if (subscription.subscriber == subscriber) {
                   subscription.active = false;
                   subscriptions.remove(i);
                   i--;
                   size--;
               }
           }
       }
   }
```

根据事件类型，从subscriptionsByEventType缓存中获取所有的Subscription，然后遍历移除，自此就取消订阅了

## 总结

- 获取EventBus单例，进行EventBus的初始化，初始化几个缓存subscriptionsByEventType、typesBySubscriber、stickyEvents和相应的线程池、注解器等参数
- subscriptionsByEventType对应着事件类型和其对应的Subscription的list，Subscription是订阅者和订阅方法组装的一个类，可根据事件类型获取所有的Subscription；typesBySubscriber对应着订阅者和事件类型list，可根据订阅者获取它订阅的所有事件；stickyEvents就是粘性事件的缓存
- 注册时，根据订阅者获取到它所有的订阅方法（SubscriberMethodFinder和FindState）；有注解器就通过注解器获取，没有最后都会通过反射和注解以及一些订阅方法的条件进行过滤出所有的订阅方法（有可能要查找父类中的订阅方法）
- 获取到所有的订阅方法后，对每个方法进行订阅；将订阅者和订阅方法组装成一个Subscription根据优先级放入EventBus初始化时的一个HashMap缓存，防止重复订阅；通过订阅方法获取订阅事件类型，通过Subscriber订阅者获取缓存中它所有相关的订阅事件，把这个新的添加进去
- 发送：获取当前线程的状态，获取当前的事件发送队列，死循环队列，然后进行发送；发送前会根据事件类型查找所有对应的Subscription（如果是由继承的事件，则要查找父类），然后遍历所有的Subscription，通过线程池根据对应的线程模式，进行对应的发送
- 接收：最后通过Subscription获取到对应的订阅方法SubscriberMethod直接调用对应的方法进行事件的接收
- 最后取消注册，从EventBus的两个缓存中移除相关的信息

## 特别鸣谢

[Android开源库——EventBus源码解析](https://juejin.im/post/5a3e19c26fb9a0452207b6b5)

[EventBus源码分析](https://segmentfault.com/a/1190000014758421#articleHeader6)