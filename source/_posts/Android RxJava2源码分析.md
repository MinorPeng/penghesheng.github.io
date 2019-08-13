---
title: Android RxJava2源码浅析
tag: Android第三方框架
date: 2019-06-11 9:21

---

<meta name="referrer" content="no-referrer" />



[TOC]

# Android RxJava2源码浅析（2.2.8）

## 介绍

一些类介绍：

- Observable：abstract class，被观察者，被订阅者，数据源，上游
- Observer：interface，观察者，订阅者，下游，数据接收处理的地方
- ObservableSource：interface，抽象订阅的接口，每一个Observable都实现了这个接口，可以将它和Observable同样看待
- ObservableOnSubscribe：interface，描述订阅的过程，实现这个subscribe方法进行发送数据，生产数据
- ObservableEmitter：interface，抽象的被订阅者发射器
- Emitter：interface，公用的抽象发射器，共有的API
- ObservableCreate：class，Observable的实例，继承自Observable，也就是一个Observable对象，可以说是原始的Observable对象，持有ObservableOnSubscribe，一个普通的使用可以通过它走完流程（即上游数据到下游数据）
- CreateEmitter：class，ObservableCreate的一个内部类，继承自AtomicReference，实现了ObservableEmitter和Disposable，真正的一个数据发射器，持有外部的observer对象
- AtomicReference：class，对象引用的自动原子更新
- Disposable：interface，描述数据的是否处置
- QueueDisposable：interface，队列形式的Disposable
- ObservableMap：class，继承自AbstractObservableWithUpstream，也是一个Observable，map的Observable，使用map的时候，则是包装了这层Observable来进行数据转换处理
- AbstractObservableWithUpstream：abstract class，继承自Observable，持有了上一级的Observable对象（也就是ObservableSource）
- MapObserver：class，继承自BasicFuseableObserver，是一个Observer，map的Observer，在onNext中通过mapper进行数据转换
- Function：interface，map使用的一个抽象接口
- BasicFuseableObserver：abstract class，实现了Observer和QueueDisposable，持有了下一级的observer对象
- ObservableSubscribeOn：class，继承自AbstractObservableWithUpstream，持有上一级的Observable和Scheduler，进行线程调度，订阅的Observable
- SubscribeOnObserver：class，继承自AtomicReference，实现了Observer和Disposable，线程调度具体实现地方，订阅的Observer
- SubscribeTask：class，就是一个Runnable，通过SubscribeOnObserver中的Scheduler去执行，run方法中就是订阅，只有第一次订阅才有效
- ObservableObserveOn：class，继承自AbstractObservableWithUpstream，持有上一级的Observable和线程调度的Scheduler，订阅者的Observable，每一次订阅都有效
- ObserveOnObserver：class，继承自BasicIntQueueDisposable，实现了Observer和Runnable，订阅者的Observer，Scheduler执行的就是它

## 源码分析

**简单使用**：

```java
Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
        emitter.onNext(1);
        emitter.onNext(2);
        emitter.onNext(3);
        emitter.onComplete();
    }
})
    .map(new Function<Integer, String>() {
        @Override
        public String apply(Integer integer) throws Exception {
            if (integer.equals(1)) {
                return "Observable1 map";
            } else if (integer.equals(2)) {
                return "Observable2 map";
            } else {
                return "Observable3 map";
            }
        }
    })
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Observer<String>() {
        @Override
        public void onSubscribe(Disposable d) {

        }

        @Override
        public void onNext(String s) {

        }

        @Override
        public void onError(Throwable e) {

        }

        @Override
        public void onComplete() {

        }
    });
```

这是一个简单的使用例子，创建Observable，发送了三个Integer（1，2，3），通过Map将数据进行了一次转换，转为String，在IO线程发送，在UI线程消费，在Observer中接收事件

整体来说，整个操作流程大致分为以下几步

- create：创建Observable，在上游来发送数据
- map：不是必须的一步，转换数据，这个放在后面分析
- subscribeOn：订阅线程，上游发送数据所在的线程
- observeOn：消费线程，下游接收数据所在的线程
- subscribe：订阅，关联起来

### create

（**这里分析的是最原始的，没有map操作，也没有线程切换**）

先看看`Observable`的创建，在使用中，我们通常是通过匿名实现`ObservableOnSubscribe`来创建的，看看`Observable`中的create方法

```java
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
    ObjectHelper.requireNonNull(source, "source is null");
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```

我们new出的ObservableOnSubscribe就是我们的数据源，也就是上游，在create方法中先是对null进行了判断，不能为空；接着就直接返回了`RxJavaPlugins.onAssembly(new ObservableCreate<T>(source))`

RxJavaPlugins是RxJava中的一个工具类，用来注入一些RxJava操作，我们的Observable（被观察者）就是通过它来创建的，跟进去看看

```java
public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
    Function<? super Observable, ? extends Observable> f = onObservableAssembly;
    if (f != null) {
        return apply(f, source);
    }
    return source;
}
```

这是一个关于hook的方法，关于hook我们暂且不表，不影响主流程，我们默认使用中都没有hook，所以这里就是直接返回source，也就是我们前面创建的对象`new ObservableCreate<T>(source)`

先好好看看这个，从`create()`方法的返回值，我们需要的`Observable`对象，但这里返回的是根据`ObservableOnSubscribe`创建的`ObservableCreate`对象

```java
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }
	...
}
```

`ObservableCreate`就是`Observable`的一个实现类，持有了`ObservableOnSubscribe`这个`source`，关于`ObservableCreate`就先看这么多，后面还会再看这个的

下面再看看`ObservableOnSubscribe`这个是什么

```java
public interface ObservableOnSubscribe<T> {
    void subscribe(@NonNull ObservableEmitter<T> emitter) throws Exception;
}
```

这就是一个接口，内部定义了`subscribe()`方法，那再看看这个`ObservableEmitter`又是什么，从这里的注释可以知道它是一个发射器，用来抽象上关联`Observable`和`Observer`的，我们发射数据也是通过它来及进行的

```java
public interface ObservableEmitter<T> extends Emitter<T> {
    void setDisposable(@Nullable Disposable d);

    void setCancellable(@Nullable Cancellable c);

    boolean isDisposed();
    
    @NonNull
    ObservableEmitter<T> serialize();
    
    boolean tryOnError(@NonNull Throwable t);
}
```

`ObservableEmitter`又是继承自`Emitter`，同时自己也实现了一些API

```java
public interface Emitter<T> {

    void onNext(@NonNull T value);

    void onError(@NonNull Throwable error);

    void onComplete();
}
```

`Emitter`是一个公用的一个接口，定义了几个方法来标识一种状态（信号），这里面的几个API我们常用到，特别是在下游消费事件的时候

`create()`方法我们先就看这么多，通过create方法得到了`ObservableCreate`对象

### subsribe

接着我们看如何进行订阅接收数据的

通过`subscribe(Observer<T> observer)`方法来进行订阅，在Observer中接收数据

```java
public final void subscribe(Observer<? super T> observer) {
    ObjectHelper.requireNonNull(observer, "observer is null");
    try {
        observer = RxJavaPlugins.onSubscribe(this, observer);

        ObjectHelper.requireNonNull(observer, "The RxJavaPlugins.onSubscribe hook returned a null Observer. Please change the handler provided to RxJavaPlugins.setOnObservableSubscribe for invalid null returns. Further reading: https://github.com/ReactiveX/RxJava/wiki/Plugins");

        subscribeActual(observer);
    } catch (NullPointerException e) { // NOPMD
        throw e;
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        RxJavaPlugins.onError(e);
        NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
        npe.initCause(e);
        throw npe;
    }
}
```

同样，先是null检查，然后可能会抛出的一些异常捕获；

然后主要看看try语句块中的：`observer = RxJavaPlugins.onSubscribe(this, observer);`也是通过`RxJavaPlugins`来进行一个关于hook的处理，如果没有hook，其实返回的就是传入的observer；然后就是再一次null检查，接着就调用了`subscribeActual(observer);`方法

`subscribeActual()`方法是一个抽象方法，需要靠子类去实现，那么前面`Observable`的创建，得到的`Observable`对象实际是`ObservableCreate`

那就看看`ObservableCreate`中的`subscribeActual()`方法

```java
protected void subscribeActual(Observer<? super T> observer) {
    //创建CreateEmitter实例
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);
    observer.onSubscribe(parent);
    try {
        //将ObservableOnSubscribe（上游）与CreateEmitter（Observer，下游）关联起来
        source.subscribe(parent);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        //错误回调
        parent.onError(ex);
    }
}
```

首先创建了`CreateEmitter`对象，这是`ObservableCreate`的一个静态内部类

```java
static final class CreateEmitter<T> extends AtomicReference<Disposable> implements ObservableEmitter<T>, Disposable {
    ...
        final Observer<? super T> observer;

    CreateEmitter(Observer<? super T> observer) {
        this.observer = observer;
    }
    ...
}
```

`CreateEmitter`继承自`AtomicReference`（一个可以通过原子方式来进行自动更新的类，**对象引用的自动原子更新**），实现了`ObservableEmitter`（这个在`Observable`的创建中我们提到过）和`Disposable`（用来描述资源是否被处置）

回到前面的`subscribeActual()`方法

接着通过`observer`回调了`onSubscribe(parent)`方法，这个参数就是刚new出的CreateEmitter（实现了Disposable）；然后通过`source`调用了`subscribe()`方法，这个`source`就是我们创建`ObservableCreate`时传入的`ObservableOnSubscribe`，也就是我们在代码里写的`ObservableOnSubscribe`，那么这里就是回调到了我们外面的`ObservableOnSubscribe`的`subscribe()`方法，这样就上游发射、下游接收就关联起来了

然后我们看看数据是怎么进行发射和接收的

在代码中，我们在`ObservableOnSubscribe`的`subscribe()`方法中进行数据的发射

```java
emitter.onNext(1);
emitter.onNext(2);
emitter.onNext(3);
emitter.onComplete();
```

比如我们通过发射器`ObservableEmitter`发射了1、2、3三个整型，这个`ObservableEmitter`就是前面创建的`CreateEmitter`

```java
public void onNext(T t) {
    if (t == null) {
        onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
        return;
    }
    if (!isDisposed()) {
        observer.onNext(t);
    }
}
```

对数据进行一个null检查，同时有`Disposable`来监控一个取消的状态，当还没有处理的时候，就直接通过`Observer`的`onNext()`进行回调了

其他的onError、onComplete等都是一样的

```java
public void onError(Throwable t) {
    if (!tryOnError(t)) {
        RxJavaPlugins.onError(t);
    }
}

public boolean tryOnError(Throwable t) {
    if (t == null) {
        t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
    }
    if (!isDisposed()) {
        try {
            observer.onError(t);
        } finally {
            dispose();
        }
        return true;
    }
    return false;
}

public void onComplete() {
    if (!isDisposed()) {
        try {
            observer.onComplete();
        } finally {
            dispose();
        }
    }
}

public void dispose() {
    DisposableHelper.dispose(this);
}

public boolean isDisposed() {
    return DisposableHelper.isDisposed(get());
}
```

在调用`onError()`或者`onComplete()`方法后，都会调用`dispose();`，来中断后续的操作了

基本上每个操作API都会用到`isDisposed()`来进行判断，我们就好好看看这个怎么来判断的

首先看看`DisposableHelper`这个类

```java
public enum DisposableHelper implements Disposable {
    /**
     * The singleton instance representing a terminal, disposed state, don't leak it.
     */
    DISPOSED
    ;
    ...
}
```

这是一个枚举类，拥有自身的单例，同时实现了Disposable

```java
public static boolean isDisposed(Disposable d) {
    return d == DISPOSED;
}
...
    public static boolean dispose(AtomicReference<Disposable> field) {
    //1 通过断点查看，默认情况下,field的值是"null"，并非引用是null哦！大坑大坑大坑
    //但是current是null引用
    Disposable current = field.get();
    Disposable d = DISPOSED;
    //2 null不等于DISPOSED
    if (current != d) {
        //3 field是DISPOSED了，current还是null
        current = field.getAndSet(d);
        if (current != d) {
            //4 默认情况下 走不到这里，这里是在设置了setCancellable()后会走到。
            if (current != null) {
                current.dispose();
            }
            return true;
        }
    }
    return false;
}
```

`isDisposed()`就是简单的比较引用是否相同

`dispose()`方法，这个通过把当前的`CreateEmitter`（继承自`AtomicReference`）传入，进行dispose（这里就直接看[张旭童](https://blog.csdn.net/zxt0601)的[RxJava2 源码解析（一）](<https://blog.csdn.net/zxt0601/article/details/61614799>)中引用的部分了）

又出现了`AtomicReference`，简单说一下相关的API吧

```java
//返回当前的引用。
V get()
//如果当前值与给定的expect引用相等，（注意是引用相等而不是equals()相等），更新为指定的update值。
boolean compareAndSet(V expect, V update)
//原子地设为给定值并返回旧值。
V getAndSet(V newValue)
```

好了，自此我们就在下游获取到了数据

### subscribeOn

接着我们加上线程切换来看看是怎么回事

```java
.subscribeOn(Schedulers.io())
```

这是订阅的线程，跟进去看看

```java
public final Observable<T> subscribeOn(Scheduler scheduler) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}
```

检查null，然后同样是`RxJavaPlugins`就不再多说，返回的就是`ObservableSubscribeOn`对象，看看类的继承结构

```java
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
	...
}

abstract class AbstractObservableWithUpstream<T, U> extends Observable<U> implements HasUpstreamObservableSource<T> {
	...
}
```

从这里可以知道，`ObservableSubscribeOn`也是一个`Observable`对象

接着看看它的构造方法

```java
public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
    super(source);
    this.scheduler = scheduler;
}
```

调用super方法，传入了source，在其父类`AbstractObservableWithUpstream`中只是简单的持有了source这个对象，也就是说`ObservableSubscribeOn`持有了上一次的`Observable`对象（也是`ObservableSource`，前面说过它们的关系）和线程调度器`Scheduler`

然后我们就接着看是怎么订阅上的，最后subscribe的时候，同普通的时候一样，会调用`subscribeActual(observer);`方法，但这里就不是`ObservableCreate`中的了，而是`ObservableSubscribeOn`中

```java
public void subscribeActual(final Observer<? super T> observer) {
    final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);
    //回调方法，传入上一级的observer onSubscribe()方法执行在 订阅处所在的线程
    observer.onSubscribe(parent);
    //线程的处理 setDisposable()是为了将子线程的操作加入Disposable管理中
    parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
}
```

在这个方法中，创建了`SubscribeOnObserver`（也是一个`Observer`对象），这个类是对传入的`observer`进行一个持有

然后创建了`SubscribeTask`

```java
final class SubscribeTask implements Runnable {
    private final SubscribeOnObserver<T> parent;

    SubscribeTask(SubscribeOnObserver<T> parent) {
        this.parent = parent;
    }

    @Override
    public void run() {
        //这里就进行了真正的订阅
        source.subscribe(parent);
    }
}
```

这就是一个Runnable，持有`SubscribeOnObserver`对象，进行了真正的订阅

然后将`SubscribeTask`传到了`scheduler`的`scheduleDirect()`方法，就是去执行`SubscribeOnObserver`中的代码，这样就做到了线程的调度

如此，我们的订阅的流程都是在我们所指定的`scheduler`中执行了

然后我们看看`SubscribeOnObserver`中的`onNext()`方法

```java
public void onNext(T t) {
    downstream.onNext(t);
}

public void onError(Throwable t) {
    downstream.onError(t);
}

public void onComplete() {
    downstream.onComplete();
}
```

可以看到，不管是`onNext()`、`onError()`还是`onComplete()`，都是直接通过downstream（这里就是我们设置的observer）直接回调了

### observeOn

`observeOn()`大致思想都差不多，只不过这里是消费的线程了（订阅者）

```java
public final Observable<T> observeOn(Scheduler scheduler) {
    return observeOn(scheduler, false, bufferSize());
}
...
    public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    ObjectHelper.verifyPositive(bufferSize, "bufferSize");
    return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
}
```

直接返回了`ObservableObserveOn`对象

```java
public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;
    final boolean delayError;
    final int bufferSize;
    public ObservableObserveOn(ObservableSource<T> source, Scheduler scheduler, boolean delayError, int bufferSize) {
        //同样保留上一次的Observable——ObservableSource
        super(source);
        this.scheduler = scheduler;
        this.delayError = delayError;
        this.bufferSize = bufferSize;
    }
    ...
}
```

订阅的时候

```java
protected void subscribeActual(Observer<? super T> observer) {
    if (scheduler instanceof TrampolineScheduler) {
        source.subscribe(observer);
    } else {
        //1 创建出一个 主线程的Worker	
        Scheduler.Worker w = scheduler.createWorker();
        //2 订阅上游数据源
        source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
    }
}
```

同样也是创建了一个`Observer`，不过这里的`Observer`是`ObserveOnObserver`

```java
static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T> implements Observer<T>, Runnable {
    //下游的观察者
    final Observer<? super T> downstream;
    //对应Scheduler里的Worker
    final Scheduler.Worker worker;
    final boolean delayError;
    final int bufferSize;
    //上游被观察者 push 过来的数据都存在这里
    SimpleQueue<T> queue;
    Disposable upstream;
    //如果onError了，保存对应的异常
    Throwable error;
    //是否完成
    volatile boolean done;
    //是否取消
    volatile boolean disposed;
    // 代表同步发送 异步发送 
    int sourceMode;
    boolean outputFused;
    ...
}        
```

接下来简单分析几个API

```java
public void onSubscribe(Disposable d) {
    if (DisposableHelper.validate(this.upstream, d)) {
        this.upstream = d;
        if (d instanceof QueueDisposable) {
            ...
                //同步
                if (m == QueueDisposable.SYNC) {
                    ...
                }
            //异步
            if (m == QueueDisposable.ASYNC) {
                ...
            }
        }
        //创建一个queue 用于保存上游 onNext() push的数据
        queue = new SpscLinkedArrayQueue<T>(bufferSize);
        //回调下游观察者onSubscribe方法
        downstream.onSubscribe(this);
    }
}
```

在`onSubscribe()`多了同步异步的判断和处理，同时创建了queue来保存上游的`onNext()`发送的数据

再看看`onNext()`等

```java
public void onNext(T t) {
    if (done) {
        return;
    }
	//如果数据源类型不是异步的， 默认不是
    if (sourceMode != QueueDisposable.ASYNC) {
        //将上游push过来的数据 加入 queue里
        queue.offer(t);
    }
    //开始进入对应Workder线程，在线程里 将queue里的t 取出 发送给下游Observer
    schedule();
}

public void onError(Throwable t) {
    if (done) {
        RxJavaPlugins.onError(t);
        return;
    }
    error = t;
    done = true;
    schedule();
}

public void onComplete() {
    if (done) {
        return;
    }
    done = true;
    schedule();
}
```

都会判断是否完成，调用`onError()`和`onComplete()`都会让done置为true，然后都会调用`schedule()`方法

```java
void schedule() {
    if (getAndIncrement() == 0) {
        //该方法需要传入一个线程， 注意看本类实现了Runnable的接口，所以查看对应的run()方法
        worker.schedule(this);
    }
}

public void run() {
    //默认false
    if (outputFused) {
        drainFused();
    } else {
        //取出queue里的数据 发送
        drainNormal();
    }
}
```

通过`drainNormal()`来进行数据发送

```java
void drainNormal() {
    int missed = 1;
    final SimpleQueue<T> q = queue;
    final Observer<? super T> a = downstream;
    for (;;) {
        //如果已经 终止 或者queue空，则跳出函数
        if (checkTerminated(done, q.isEmpty(), a)) {
            return;
        }

        for (;;) {
            boolean d = done;
            T v;
            try {
                //从queue里取出一个值
                v = q.poll();
            } catch (Throwable ex) {
                ...
                return;
            }
            boolean empty = v == null;
            //再次检查 是否 终止  如果满足条件 跳出函数
            if (checkTerminated(d, empty, a)) {
                return;
            }
            //上游还没结束数据发送，但是这边处理的队列已经是空的，不会push给下游 Observer
            if (empty) {
                //仅仅是结束这次循环，不发送这个数据而已，并不会跳出函数
                break;
            }
            //发送给下游
            a.onNext(v);
        }
		...
    }
}
```

这个方法主要就是死循环，不断从队列中取，然后进行null检查已经检查是否种植，发送给下游，回调

### map操作符

得到了`ObservableCreate`对象后，在其`subscribe()`方法发送数据

```java
emitter.onNext(1);
emitter.onNext(2);
emitter.onNext(3);
emitter.onComplete();
```

接着，我们可以选择使用map操作符，来对数据进行一个转换操作（这是RxJava非常强大的地方之一了）

使用的时候，主要是创建了Function来做这个数据如何变换，在`apply()`方法中会接收到前面发送的数据，这个时候我们就可以进行对应的数据转换操作了

```java
.map(new Function<Integer, String>() {
    @Override
    public String apply(Integer integer) throws Exception {
        if (integer.equals(1)) {
            return "Observable1 map";
        } else if (integer.equals(2)) {
            return "Observable2 map";
        } else {
            return "Observable3 map";
        }
    }
})
```

然后将这个Function设置给`Observable`对象，也就是前面的`ObservableCreate`

```java
public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
    ObjectHelper.requireNonNull(mapper, "mapper is null");
    return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
}
```

这里又出现了`RxJavaPlugins`这个类，其实进去看源码知道，如果没有hook的情况下，都会返回你传进去的参数，传进去是什么，返回的就是什么，如果有hook就有对应的一些处理；默认是没有hook的，所以我们就不再看有hook的情况了

那么这里就是这里new出了`ObservableMap`，并返回了它

```java
public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
	...
}

abstract class AbstractObservableWithUpstream<T, U> extends Observable<U> implements HasUpstreamObservableSource<T> {
    ...
}        
```

通过这个类的继承关系，就知道，`ObservableMap`也是一个`Observable`对象（`ObservableCreate`），所以我们前面能够直接返回

看看这个`ObservableMap`对象的构造方法，因为参数是`(this, mapper)`这样的

```java
public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
    super(source);
    this.function = function;
}
```

`ObservableSource`是什么？`ObservableSource`是一个接口，从前面传参的时候，说明`Observable`是实现了这个接口的

```java
public abstract class Observable<T> implements ObservableSource<T> {
	...
}

public interface ObservableSource<T> {
    void subscribe(@NonNull Observer<? super T> observer);
}
```

它代表了一个标准的无背压的 源数据接口

那么现在知道了这个source是什么了（从另一方面来说，`ObservableSource`其实是和`Observable`（`ObservableCreate`）等价的，可能只是为了从`Observable`众多的操作剥离出这个抽象来），同时我们得到了source是为了在`AbstractObservableWithUpstream`保存这个上游的`ObservableSource`，也就是我们一开始创建的`ObservableCreate`对象

那么现在再看看`ObservableMap`，它首先持有了进行数据转换的Function，它还继承自`AbstractObservableWithUpstream`，这个类里也保存了转换之前的`Observable`（上游）

也就是说，如果我们使用了map操作，最后得到的`Observable`对象其实是`ObservableMap`对象，那么在接下来调用订阅的时候`subscribe()`，会调用到`ObservableMap`中的`subscribeActual()`方法

```java
public void subscribeActual(Observer<? super U> t) {
    //这里的t就是外面的observer，function就是之前通过map定义的Function进行数据转换的，source就是之前的Observable，建立了和MapObserver的关联
    source.subscribe(new MapObserver<T, U>(t, function));
}
```

在这里直接创建了一个`MapObserver`，这是`ObservableMap`的静态内部类

```java
static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
    ...
        MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
        super(actual);
        this.mapper = mapper;
    }
    ...
}            
```

`MapObserver`继承自`BasicFuseableObserver`，从类名就应该能猜出这就是一个Observer

```java
public abstract class BasicFuseableObserver<T, R> implements Observer<T>, QueueDisposable<R> {
    //下游的订阅者 observer
    protected final Observer<? super R> downstream;
    ...
    public BasicFuseableObserver(Observer<? super R> downstream) {
        this.downstream = downstream;
    }
	...
}
```

从`BasicFuseableObserver`的继承结构看到，它实现了`Observer`，所以它也是一个`Observer`，同时保存了我们传入的`Observer`，它还实现了队列的Disposable`QueueDisposable`，这就说明了我们的map操作是一个一个执行的，按队列执行的

所以`MapObserver`和`BasicFuseableObserver`都是对`Observer`的再一层包装；`ObservableMap`也是对之前的`Observable`的一层包装，保留了之前的`Observable`（source就是上一次的`ObservableSource`，最开始创建的`ObservableCreate`）

和没有map的时候一样，前面`Observable`发送数据，通过`onNext()`发送，`onComplete()`结束，那么看看`MapObserver`中的`onNext()`方法

```java
public void onNext(T t) {
    //调用onComplete和onError方法，会在BasicFuseableObserver中置为true
    if (done) {
        return;
    }

    if (sourceMode != NONE) {
        downstream.onNext(null);
        return;
    }

    U v;

    try {
        v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
    } catch (Throwable ex) {
        fail(ex);
        return;
    }
    downstream.onNext(v);
}
```

T是转换前的数据类型，U是转换后的数据类型，在这个方法中，很直观的看到，通过调用`mapper.apply(t)`来进行了数据类型的转换（这个mapper就是我们通过map设置的数据转换Function），将T转换为U，然后通过downstream（就是我们代码设置的Observer）回调，这样就进行了我们的数据转换

梳理一下map的流程

订阅的过程，是从下游到上游依次订阅的：

1. 即终点 `Observer` 订阅了 map 返回的`ObservableMap`。
2. 然后map的`Observable`(`ObservableMap`)在被订阅时，会订阅其内部保存上游`Observable`（创建的`ObservableCreate`），用于订阅上游的`Observer`是`MapObserver`，内部保存了下游（本例是终点）`Observer`，以便上游发送数据过来时，能传递给下游。
3. 以此类推，直到源头`Observable`被订阅，根据前面的内容，它开始向`Observer`发送数据。

数据传递的过程，当然是从上游push到下游的：

1. 源头`Observable`（`ObservableCreate`）传递数据给下游`Observer`（这里就是`MapObserver`）
2. 然后`MapObserver`接收到数据，对其变换操作后(实际的function在这一步执行)，再调用内部保存的下游`Observer`的`onNext()`发送数据给下游
3. 以此类推，直到终点`Observer`

## 总结

- 没有其他操作时

    ![RxJava2没有其他操作时](https://img-blog.csdnimg.cn/20190611174554298.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWR1XzM2OTU5ODg2,size_16,color_FFFFFF,t_70)

- 加上map

    ![RxJava2有map](https://img-blog.csdnimg.cn/20190611180410578.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWR1XzM2OTU5ODg2,size_16,color_FFFFFF,t_70)

    1. 内部对上游Observable进行订阅
    2. 内部订阅者接收到数据后，将数据转换，发送给下游Observer.
    3. 操作符返回的Observable和其内部订阅者、是装饰者模式的体现。
    4. 操作符数据变换的操作，也是发生在订阅后。

- 加上线程调度

    ![加上线程调度](https://img-blog.csdnimg.cn/20190611182202538.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JhaWR1XzM2OTU5ODg2,size_16,color_FFFFFF,t_70)

    - 线程调度subscribeOn()：

    1. 内部先切换线程，在切换后的线程中对上游Observable进行订阅，这样上游发送数据时就是处于被切换后的线程里了。
    2. 也因此多次切换线程，最后一次切换（离源数据最近）的生效。
    3. 内部订阅者接收到数据后，直接发送给下游Observer.
    4. 引入内部订阅者是为了控制线程（dispose）
    5. 线程切换发生在Observable中。

    - 线程调度observeOn():

    1. 使用装饰的Observer对上游Observable进行订阅
    2. 在Observer中onXXX()方法里，将待发送数据存入队列，同时请求切换线程处理真正push数据给下游。
    3. 多次切换线程，都会对下游生效。

## 特别鸣谢

- [RxJava2 源码解析（一）](<https://blog.csdn.net/zxt0601/article/details/61614799>)
- [RxJava2 源码解析（二）](<https://blog.csdn.net/zxt0601/article/details/61637439>)