---
title: Android Glide源码分析
date: 2019-08-11
tag: Android
---

<meta name="referrer" content="no-referrer" />



# Android Glide源码分析（4.9.0）

通常使用Glide就是如下的一个链式使用

`Glide.with(context).load(url).into(imageView);`

这是最基本的一个用法，下面就主要看看这三个步骤主要干了什么

## with()

with可以重载以下的几个方法，然后返回一个RequestManager

```java
@NonNull
public static RequestManager with(@NonNull Context context) {
    return getRetriever(context).get(context);
}

@NonNull
public static RequestManager with(@NonNull Activity activity) {
    return getRetriever(activity).get(activity);
}

@NonNull
public static RequestManager with(@NonNull FragmentActivity activity) {
    return getRetriever(activity).get(activity);
}

@NonNull
public static RequestManager with(@NonNull Fragment fragment) {
    return getRetriever(fragment.getActivity()).get(fragment);
}

@SuppressWarnings("deprecation")
@Deprecated
@NonNull
public static RequestManager with(@NonNull android.app.Fragment fragment) {
    return getRetriever(fragment.getActivity()).get(fragment);
}

@NonNull
public static RequestManager with(@NonNull View view) {
    return getRetriever(view.getContext()).get(view);
}
```

这几个重载的方法无非就是根据Context的来源不用进行的一个重载，最后都是获取一个Context对象，调用`getRetriever()`来获取一个RequestManagerRetriever对象，然后获取对应的RequestManager

```java
private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    Preconditions.checkNotNull(context,
        "You cannot start a load on a not yet attached View or a Fragment where getActivity() "
            + "returns null (which usually occurs when getActivity() is called before the Fragment "
            + "is attached or after the Fragment is destroyed).");
    return Glide.get(context).getRequestManagerRetriever();
}
```

`getRetriever()`对Context进行检查，Context是不能为null的，然后就是通过Glide的get方法来获取了一个RequestManagerRetriever

```java
public static Glide get(@NonNull Context context) {
    if (glide == null) {
      synchronized (Glide.class) {
        if (glide == null) {
          checkAndInitializeGlide(context);
        }
      }
    }
    return glide;
}
```

Glide的get方法就主要是获取Glide的单例实例，Glide没有实例化就会触发一次实例化

```java
private static void checkAndInitializeGlide(@NonNull Context context) {
    ...
        initializeGlide(context);
    ...
}

private static void initializeGlide(@NonNull Context context) {
    initializeGlide(context, new GlideBuilder());
}

@SuppressWarnings("deprecation")
private static void initializeGlide(@NonNull Context context, @NonNull GlideBuilder builder) {
    Context applicationContext = context.getApplicationContext();
    //根据注解获取对应的Module
    GeneratedAppGlideModule annotationGeneratedModule = getAnnotationGeneratedGlideModules();
    //获取manifest中的Module
    List<com.bumptech.glide.module.GlideModule> manifestModules = Collections.emptyList();
    if (annotationGeneratedModule == null || annotationGeneratedModule.isManifestParsingEnabled()) {
        manifestModules = new ManifestParser(applicationContext).parse();
    }
    ...
    //设置RequestManagerFactory
    RequestManagerRetriever.RequestManagerFactory factory =
	annotationGeneratedModule != null    ? annotationGeneratedModule.getRequestManagerFactory() : null;
    builder.setRequestManagerFactory(factory);
    //自定义的GlideModule
    for (com.bumptech.glide.module.GlideModule module : manifestModules) {
        module.applyOptions(applicationContext, builder);
    }
    if (annotationGeneratedModule != null) {
        annotationGeneratedModule.applyOptions(applicationContext, builder);
    }
    //创建一个Glide
    Glide glide = builder.build(applicationContext);
    ...
        applicationContext.registerComponentCallbacks(glide);
    //保存到单例中
    Glide.glide = glide;
}
```

初始化Glide会调用`initializeGlide(context)`方法，然后会创建一个GlideBuilder，再重载一次`initializeGlide(context, glideBuilder)`方法，进行初始化；

GlideBuilder根据命名就大致可以看出是干嘛的，利用了Builder模式，来构造Glide，可以通过GlideBuilder来进行Glide的配置，如果不进行配置，就使用GlideBuilder中默认的配置咯；当然这里就是一次new，没有通过build来进行创建，但是在后面会进行build的

在调用`initializeGlide(context, glideBuilder)`中，会从注解和 Manifest 中获取 GlideModule，并根据各 GlideModule 中的方法对 Glide 进行自定义；这就意味着通过GlideBuilder我们基本上初始化了一个Glide，然后保存到了单例中 

看看GlideBuilder的build方法时，我们初始化了哪些默认参数

```java
Glide build(@NonNull Context context) {
    //请求的Executor
    if (sourceExecutor == null) {
        sourceExecutor = GlideExecutor.newSourceExecutor();
    }
    //磁盘缓存的Executor
    if (diskCacheExecutor == null) {
        diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();
    }

    if (animationExecutor == null) {
        animationExecutor = GlideExecutor.newAnimationExecutor();
    }
    //内存大小计算的
    if (memorySizeCalculator == null) {
        memorySizeCalculator = new MemorySizeCalculator.Builder(context).build();
    }
    //检查连接的
    if (connectivityMonitorFactory == null) {
        connectivityMonitorFactory = new DefaultConnectivityMonitorFactory();
    }
    //Bitmap池，缓存Bitmap的
    if (bitmapPool == null) {
        int size = memorySizeCalculator.getBitmapPoolSize();
        if (size > 0) {
            //
            bitmapPool = new LruBitmapPool(size);
        } else {
            bitmapPool = new BitmapPoolAdapter();
        }
    }

    if (arrayPool == null) {
        arrayPool = new LruArrayPool(memorySizeCalculator.getArrayPoolSizeInBytes());
    }
    //内存缓存
    if (memoryCache == null) {
        memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
    }
    //磁盘缓存
    if (diskCacheFactory == null) {
        diskCacheFactory = new InternalCacheDiskCacheFactory(context);
    }
    //网络请求的Engine
    if (engine == null) {
        engine = new Engine(memoryCache, diskCacheFactory, diskCacheExecutor, sourceExecutor, GlideExecutor.newUnlimitedSourceExecutor(), GlideExecutor.newAnimationExecutor(), isActiveResourceRetentionAllowed);
    }

    if (defaultRequestListeners == null) {
        defaultRequestListeners = Collections.emptyList();
    } else {
        defaultRequestListeners = Collections.unmodifiableList(defaultRequestListeners);
    }
    //初始化RequestManagerRetriever，requestManagerFactory是Glide通过setRequestManagerFactory方法设置进来的
    RequestManagerRetriever requestManagerRetriever = new RequestManagerRetriever(requestManagerFactory);

    return new Glide(context, engine, memoryCache, bitmapPool, arrayPool, requestManagerRetriever, connectivityMonitorFactory, logLevel, defaultRequestOptions.lock(), defaultTransitionOptions, defaultRequestListeners, isLoggingRequestOriginsEnabled);
}
```

就是一些配置的初始化，直接就可以看懂了



由此，我们通过`Glide.get()`就获取了Glide的单实例，接着通过`getRequestManagerRetriever()`获取了RequestManagerRetriever

也就是`getRetriever(context)`最后得到了RequestManagerRetriever对象，然后通过`get(context)`获取对应的RequestManger，这里重载了很多get方法

```java
public RequestManager get(@NonNull Context context) {
    if (context == null) {
        throw new IllegalArgumentException("You cannot start a load on a null Context");
    } else if (Util.isOnMainThread() && !(context instanceof Application)) {
        if (context instanceof FragmentActivity) {
            return get((FragmentActivity) context);
        } else if (context instanceof Activity) {
            return get((Activity) context);
        } else if (context instanceof ContextWrapper) {
            return get(((ContextWrapper) context).getBaseContext());
        }
    }
    return getApplicationManager(context);
}
```

也就是不管前面根据Activity、Fragment还是View、Context，都会调用对应的一个get方法

以Activity为例

```java
public RequestManager get(@NonNull Activity activity) {
    //在后台线程
    if (Util.isOnBackgroundThread()) {
      	return get(activity.getApplicationContext());
    } else {
        //否则会使用无 UI 的 Fragment 进行管理
        assertNotDestroyed(activity);
        android.app.FragmentManager fm = activity.getFragmentManager();
        return fragmentGet(
            activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
}
```

从这里就可以看出，Glide都是通过一个无UI的Fragment来进行管理生命周期的

```java
private RequestManager fragmentGet(@NonNull Context context,
                                   @NonNull android.app.FragmentManager fm,
                                   @Nullable android.app.Fragment parentHint,
                                   boolean isParentVisible) {
    RequestManagerFragment current = getRequestManagerFragment(fm, parentHint, isParentVisible);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
        // TODO(b/27524013): Factor out this Glide.get() call.
        Glide glide = Glide.get(context);
        requestManager =
            factory.build(
            glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
        current.setRequestManager(requestManager);
    }
    return requestManager;
}
```

这里先获取到了`RequestManagerFragment`和当前的`RequestManager`，接着获取Glide实例，从 `RequestManagerFragment` 中通过 `getGlideLifecycle()` 获取到了 `Lifecycle` 对象。`Lifecycle` 对象提供了一系列的、针对 Fragment 生命周期的方法。它们将会在 Fragment 的各个生命周期方法中被回调。然后，我们将该 `Lifecycle` 传入到 `RequestManager` 中，以 `RequestManager` 中的两个方法为例，`RequestManager` 会对 `Lifecycle` 进行监听，从而达到了对 Fragment 的生命周期进行监听的目的：

```java
public void onAttach(Activity activity) {
    super.onAttach(activity);
    ...
}

@Override
public void onDetach() {
    super.onDetach();
    unregisterFragmentWithRoot();
}

@Override
public void onStart() {
    super.onStart();
    lifecycle.onStart();
}

@Override
public void onStop() {
    super.onStop();
    lifecycle.onStop();
}

@Override
public void onDestroy() {
    super.onDestroy();
    lifecycle.onDestroy();
    unregisterFragmentWithRoot();
}
```



这样，最后通过`with()`我们就得到了一个RequestManager对象

## load()

前面通过`with()`拿到了RequestManager，接下来就是调用load方法了

在RequestManager中，因为实现了ModelTypes接口，所以重载了很多load方法，这就是为了方便我们使用，这样我们就是加载不同来源的图片了

```java
  T load(@Nullable Bitmap bitmap);

  T load(@Nullable Drawable drawable);

  T load(@Nullable String string);

  T load(@Nullable Uri uri);

  T load(@Nullable File file);

  T load(@RawRes @DrawableRes @Nullable Integer resourceId);

  T load(@Nullable URL url);

  T load(@Nullable byte[] model);

  T load(@Nullable Object model);
```

在RequestManager中，所有的load方法都是返回了`RequestBuilder<Drawable>`这个对象的，所以基本上处理的逻辑大致差不多

我们就选常用的string这个吧，因为常用的就是网络图片

```java
public RequestBuilder<Drawable> load(@Nullable String string) {
    return asDrawable().load(string);
}
```

`asDrawable()`就是获取到一个RequestBuilder（是一个Drawable的），这又是一个Builder模式，接着就调用了load方法

```java
//TranscodeType只是一个泛型表示
public RequestBuilder<TranscodeType> load(@Nullable String string) {
    return loadGeneric(string);
}

private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    this.model = model;
    isModelSet = true;
    return this;
}
```

其实，不论我们使用了 `load()` 的哪个重载方法，最终都会调用到下面的方法。它的逻辑也比较简单，就是将我们的图片资源信息赋值给 `RequestBuilder` 的局部变量就完事了。至于图片如何被加载和显示，则在 `into()` 方法中进行处理。

## into()

前面通过`with()`和`load()`分别先后获取了Glide的单例，然后将要加载的图片资源信息保存到了RequestBuilder中，接着就是最后一步也是最重要的一步，`into()`，这一步既包含了加载图片也包含了展示

这一步大致包含了以下几个过程

### 开启DecodeJob

`DecodeJob` 负责从缓存或者从原始的数据源中加载图片资源，对图片进行变换和转码，是 Glide 图片加载过程的核心。`DecodeJob` 继承了 `Runnable`，实际进行图片加载的时候会将其放置到线程池当中执行。这个阶段我们重点介绍的是从 `RequestBuilder` 构建一个 `DecodeJob` 并开启 `DecodeJob` 任务的过程。即构建一个 `DecodeJob` 并将其丢到线程池里的过程。

into方法也是有几个重载的，我们直接看ImageView这个就可以了

```java
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    //保证在UI线程执行
    Util.assertMainThread();
    Preconditions.checkNotNull(view);
	//获取RequestOptions
    BaseRequestOptions<?> requestOptions = this;
    ...
    return into(
        glideContext.buildImageViewTarget(view, transcodeClass),
        /*targetListener=*/ null,
        requestOptions,
        Executors.mainThreadExecutor());
}
```

先是保证当前是在UI线程，接着into方法接着调用了内部的into方法，其中`glideContext.buildImageViewTarget(view, transcodeClass)`是将ImageView封装成了一个BitmapImageViewTarget对象（transcodeClass是前面创建RequestBuilder时设置的Drawable，这是一个泛型），`Executors.mainThreadExecutor());`是获取主线程的线程池，方便最后回调用UI线程

先看看创建Target吧，不然后面结果回调不知道这个Target具体是什么

创建Target是根据GlideContext来创建的

```java
public <X> ViewTarget<ImageView, X> buildImageViewTarget(
    @NonNull ImageView imageView, @NonNull Class<X> transcodeClass) {
    return imageViewTargetFactory.buildTarget(imageView, transcodeClass);
}
```

接着调用了ImageViewTargetFactory来进行创建

```java
public <Z> ViewTarget<ImageView, Z> buildTarget(@NonNull ImageView view,
                                                @NonNull Class<Z> clazz) {
    if (Bitmap.class.equals(clazz)) {
        return (ViewTarget<ImageView, Z>) new BitmapImageViewTarget(view);
    } else if (Drawable.class.isAssignableFrom(clazz)) {
        return (ViewTarget<ImageView, Z>) new DrawableImageViewTarget(view);
    } else {
        throw new IllegalArgumentException(
            "Unhandled class: " + clazz + ", try .as*(Class).transcode(ResourceTranscoder)");
    }
}
```

这里就会根据clazz来判断创建什么样的Target，通过前面可以知道我们需要的是一个Drawable对象，所以这里就会创建一个DrawableImageViewTarget对象



好了，接着看重载的into吧

```java
private <Y extends Target<TranscodeType>> Y into(
    @NonNull Y target,
    @Nullable RequestListener<TranscodeType> targetListener,
    BaseRequestOptions<?> options,
    Executor callbackExecutor) {
    //检查封装的ViewTarget
    Preconditions.checkNotNull(target);
    if (!isModelSet) {
        throw new IllegalArgumentException("You must call #load() before calling #into()");
    }
    //创建Request
    Request request = buildRequest(target, targetListener, options, callbackExecutor);

    Request previous = target.getRequest();
    if (request.isEquivalentTo(previous)
        && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
        ...
    }

    requestManager.clear(target);
    target.setRequest(request);
    requestManager.track(target, request);

    return target;
}
```

先看看根据Target和主线程回调线程池这些来创建的Request

```java
private Request buildRequest(Target<TranscodeType> target, @Nullable RequestListener<TranscodeType> targetListener, BaseRequestOptions<?> requestOptions, Executor callbackExecutor) {
    return buildRequestRecursive(target, targetListener, /*parentCoordinator=*/ null, transitionOptions, requestOptions.getPriority(), requestOptions.getOverrideWidth(), requestOptions.getOverrideHeight(), requestOptions, callbackExecutor);
}

private Request buildRequestRecursive(
    Target<TranscodeType> target,
    @Nullable RequestListener<TranscodeType> targetListener,
    @Nullable RequestCoordinator parentCoordinator,
    TransitionOptions<?, ? super TranscodeType> transitionOptions,
    Priority priority,
    int overrideWidth,
    int overrideHeight,
    BaseRequestOptions<?> requestOptions,
    Executor callbackExecutor) {

    // Build the ErrorRequestCoordinator first if necessary so we can update parentCoordinator.
    ErrorRequestCoordinator errorRequestCoordinator = null;
    if (errorBuilder != null) {
        errorRequestCoordinator = new ErrorRequestCoordinator(parentCoordinator);
        parentCoordinator = errorRequestCoordinator;
    }
    //创建主要的Request，也就是我们需要加载的图片的Request 这里主要就是根据这些参数最后得到一个SingleRequest对象，同时将持有这个Target对象（DrawableImageViewTarget）
    Request mainRequest = buildThumbnailRequestRecursive(target, targetListener, parentCoordinator, transitionOptions, priority, overrideWidth, overrideHeight, requestOptions, callbackExecutor);

    if (errorRequestCoordinator == null) {
        return mainRequest;
    }

    int errorOverrideWidth = errorBuilder.getOverrideWidth();
    int errorOverrideHeight = errorBuilder.getOverrideHeight();
    if (Util.isValidDimensions(overrideWidth, overrideHeight)
        && !errorBuilder.isValidOverride()) {
        errorOverrideWidth = requestOptions.getOverrideWidth();
        errorOverrideHeight = requestOptions.getOverrideHeight();
    }
    //如果设置了error，就会创建一个erroe的Request，进行加载error的图片资源
    Request errorRequest = errorBuilder.buildRequestRecursive(target, targetListener, errorRequestCoordinator, errorBuilder.transitionOptions, errorBuilder.getPriority(), errorOverrideWidth, errorOverrideHeight, errorBuilder, callbackExecutor);
    errorRequestCoordinator.setRequests(mainRequest, errorRequest);
    return errorRequestCoordinator;
}
```

`buildRequest()`直接调用了内部的`buildRequestRecursive()`方法

会根据我们是否调用过 `RequestBuilder` 的 `error()` 方法设置过图片加载出错时候显示的图片来决定返回 `mainRequest` 还是 `errorRequestCoordinator`。因为我们没有设置该参数，所以会直接返回 `mainRequest`（最后是生成的SingleRequest对象，具体可以见源码的调用）



回到前面的into方法，得到了Request后，就会通过`requestManager.track(target, request);`来执行这个Request

```java
synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
    targetTracker.track(target);
    requestTracker.runRequest(request);
}
```

该方法的主要作用有两个：

1. 调用 `TargetTracker` 的 `track()` 方法对对当前 `Target` 的生命周期进行管理，就是DrawableImageViewTarget；
2. 调用 `RequestTracker` 的 `runRequest()` 方法对当前请求进行管理，当 Glide 未处于暂停状态的时候，会直接使用 `Request` 的 `begin()` 方法开启请求。

```java
public void runRequest(@NonNull Request request) {
    requests.add(request);
    if (!isPaused) {
        //开始请求
        request.begin();
    } else {
        request.clear();
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            Log.v(TAG, "Paused, delaying request");
        }
        //如果暂停的或，会加入到pendingRequests这个list中等待后面执行
        pendingRequests.add(request);
    }
}
```

这里的Request是一个SingleRequest，所以看它里面的begin方法

```java
public synchronized void begin() {
    assertNotCallingCallbacks();
    stateVerifier.throwIfRecycled();
    startTime = LogTime.getLogTime();
    //这个model是在SingleRequest创建的时候设置的（最开始其实是load方法设置的），如果前面没有load()的参数是null就不会进行后面的操作
    if (model == null) {
        if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
            width = overrideWidth;
            height = overrideHeight;
        }
        int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
        onLoadFailed(new GlideException("Received null model"), logLevel);
        return;
    }

    if (status == Status.RUNNING) {
        throw new IllegalArgumentException("Cannot restart a running request");
    }
 	// 如果我们在完成之后重新启动（通常通过诸如 notifyDataSetChanged() 之类的方法，
    // 在相同的目标或视图中启动相同的请求），我们可以使用我们上次检索的资源和大小
    // 并跳过获取新的大小。所以，如果你因为 View 大小发生了变化而想要重新加载图片
    // 就需要在开始新加载之前清除视图 (View) 或目标 (Target)。
    if (status == Status.COMPLETE) {
        //已经有了图片数据源
        onResourceReady(resource, DataSource.MEMORY_CACHE);
        return;
    }
    status = Status.WAITING_FOR_SIZE;
    //保证size>0
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        //在这里进行请求
        onSizeReady(overrideWidth, overrideHeight);
    } else {
        //直接从target中获取
        target.getSize(this);
    }

    if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
        && canNotifyStatusChanged()) {
        //Target的生命周期
        target.onLoadStarted(getPlaceholderDrawable());
    }
    if (IS_VERBOSE_LOGGABLE) {
        logV("finished run method in " + LogTime.getElapsedMillis(startTime));
    }
}
```

这个方法几个步骤都在注释中了，当`status == Status.RUNNING`会调用`target.onLoadStarted(getPlaceholderDrawable());`触发Target的第一个生命周期

看看`onSizeReady()`方法

```java
public synchronized void onSizeReady(int width, int height) {
    //一些检验，status必须是Status.WAITING_FOR_SIZE
    ...
    //接着改变为RUNNING
    status = Status.RUNNING;

    float sizeMultiplier = requestOptions.getSizeMultiplier();
    this.width = maybeApplySizeMultiplier(width, sizeMultiplier);
    this.height = maybeApplySizeMultiplier(height, sizeMultiplier);

    if (IS_VERBOSE_LOGGABLE) {
        logV("finished setup for calling load in " + LogTime.getElapsedMillis(startTime));
    }
    loadStatus =
        engine.load(
        glideContext,
        model,
        requestOptions.getSignature(),
        this.width,
        this.height,
        requestOptions.getResourceClass(),
        transcodeClass,
        priority,
        requestOptions.getDiskCacheStrategy(),
        requestOptions.getTransformations(),
        requestOptions.isTransformationRequired(),
        requestOptions.isScaleOnlyOrNoTransform(),
        requestOptions.getOptions(),
        requestOptions.isMemoryCacheable(),
        requestOptions.getUseUnlimitedSourceGeneratorsPool(),
        requestOptions.getUseAnimationPool(),
        requestOptions.getOnlyRetrieveFromCache(),
        this,
        callbackExecutor);

    if (status != Status.RUNNING) {
        loadStatus = null;
    }
    if (IS_VERBOSE_LOGGABLE) {
        logV("finished onSizeReady in " + LogTime.getElapsedMillis(startTime));
    }
}
```

改变状态，然后通过Engine调用load方法进行请求

```java
public synchronized <R> LoadStatus load(/**这里是参数，太多了*/) {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;
	//生成一个EngineKey，标识这一次Request
    EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
                                        resourceClass, transcodeClass, options);
	//如果已经有了源数据，直接进行加载
    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
        //回调
        cb.onResourceReady(active, DataSource.MEMORY_CACHE);
        if (VERBOSE_IS_LOGGABLE) {
            logWithTimeAndKey("Loaded resource from active resources", startTime, key);
        }
        return null;
    }
	//如果有缓存，获取缓存，防止重复网络请求
    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
        cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
        if (VERBOSE_IS_LOGGABLE) {
            logWithTimeAndKey("Loaded resource from cache", startTime, key);
        }
        return null;
    }
	//从缓存的EngineJob获取
    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
    if (current != null) {
        current.addCallback(cb, callbackExecutor);
        if (VERBOSE_IS_LOGGABLE) {
            logWithTimeAndKey("Added to existing load", startTime, key);
        }
        return new LoadStatus(cb, current);
    }
	//管理资源加载以及当资源加载完毕的时候通知回调，内部维护了一个线程池
    EngineJob<R> engineJob =
        engineJobFactory.build(
        key,
        isMemoryCacheable,
        useUnlimitedSourceExecutorPool,
        useAnimationPool,
        onlyRetrieveFromCache);
	//否则开启一个任务进行请求
    DecodeJob<R> decodeJob =
        decodeJobFactory.build(
        glideContext,
        model,
        key,
        signature,
        width,
        height,
        resourceClass,
        transcodeClass,
        priority,
        diskCacheStrategy,
        transformations,
        isTransformationRequired,
        isScaleOnlyOrNoTransform,
        onlyRetrieveFromCache,
        options,
        engineJob);
	//存入Jobs缓存
    jobs.put(key, engineJob);
	//设置回调，后面要分析结果怎么返回的
    engineJob.addCallback(cb, callbackExecutor);
    //执行DecodeJob
    engineJob.start(decodeJob);

    if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Started new load", startTime, key);
    }
    return new LoadStatus(cb, engineJob);
}
```

`EngineJob` 内部维护了线程池，用来管理资源加载，以及当资源加载完毕的时候通知回调。 `DecodeJob` 继承了 `Runnable`，是线程池当中的一个任务。就像上面那样，我们通过调用 `engineJob.start(decodeJob)` 来开始资源加载。

- 从ActiveResuorces中获取，看能否直接获取到，否则进行下一步
- 从Cache中获取，没有则进行下一步
- 从缓存的EngineJob中获取，等待执行回调，没有则进行下一步
- 创建一个DecodeJob来进行请求

### 打开网络流

在上一段内容中，我们知道了如果不能直接获取到结果，就会创建一个EngineJob和DecodeJob，将DecodeJob这个Runnable放到EngineJob的线程池中执行

看看DecodeJob的run方法

```java
public void run() {
    ...
    DataFetcher<?> localFetcher = currentFetcher;
    try {
        if (isCancelled) {
            notifyFailed();
            return;
        }
        runWrapped();
    } catch (CallbackException e) {
       ...
    } catch (Throwable t) {
       ...
    } finally {
        ..
        if (localFetcher != null) {
            localFetcher.cleanup();
        }
        GlideTrace.endSection();
    }
}
```

就是调用了`runWrapped()`方法

```java
private void runWrapped() {
    switch (runReason) {
        case INITIALIZE:
            stage = getNextStage(Stage.INITIALIZE);
            currentGenerator = getNextGenerator();
            runGenerators();
            break;
        case SWITCH_TO_SOURCE_SERVICE:
            runGenerators();
            break;
        case DECODE_DATA:
            decodeFromRetrievedData();
            break;
        default:
            throw new IllegalStateException("Unrecognized run reason: " + runReason);
    }
}
```

这里的 `runReason` 是一个枚举类型，它包含的枚举值即为上面的三种类型。当我们在一个过程执行完毕之后会回调 `DecodeJob` 中的方法修改 `runReason`，然后根据新的状态值执行新的逻辑。

除了 `runReason`，`DecodeJob` 中还有一个变量 `stage` 也是用来决定 `DecodeJob` 状态的变量。同样，它也是一个枚举，用来表示将要加载数据的数据源以及数据的加载状态。它主要在加载数据的时候在 `runGenerators()`、`runWrapped()` 和 `getNextStage()` 三个方法中被修改。通常它的逻辑是，先从（大小、尺寸等）转换之后的缓存中拿数据，如果没有的话再从没有转换过的缓存中拿数据，最后还是拿不到的话就从原始的数据源中加载数据。

对于一个新的任务，会在 `DecodeJob` 的 `init()` 方法中将 `runReason` 置为 `INITIALIZE`，所以，我们首先会进入到上述 `switch` 中的 `INITIALIZE` 中执行。然后，因为我们没有设置过磁盘缓存的策略，因此会使用默认的 `AUTOMATIC` 缓存方式。于是，我们将会按照上面所说的依次从各个缓存中拿数据。由于我们是第一次加载，并且暂时我们不考虑缓存的问题，所以，最终数据的加载会交给 `SourceGenerator` 进行。

先看看`getNextGenerator();`

```java
private DataFetcherGenerator getNextGenerator() {
    switch (stage) {
        case RESOURCE_CACHE:
            return new ResourceCacheGenerator(decodeHelper, this);
        case DATA_CACHE:
            return new DataCacheGenerator(decodeHelper, this);
        case SOURCE:
            return new SourceGenerator(decodeHelper, this);
        case FINISHED:
            return null;
        default:
            throw new IllegalStateException("Unrecognized stage: " + stage);
    }
}
```

这个就是前面所说的顺序

- 先从转换之后的缓存中获取，如果没有则进行下一步
- 从没有转换过的缓存中获取，如果没有则进行下一步
- 从原始的数据源中获取数据

stage值会在`runGenerators();`进行修改，这样一次一次向后调用

```java
private void runGenerators() {
    currentThread = Thread.currentThread();
    startFetchTime = LogTime.getLogTime();
    boolean isStarted = false;
    while (!isCancelled && currentGenerator != null
           && !(isStarted = currentGenerator.startNext())) {
        //这里就会根据stage去获取对应的数据
        stage = getNextStage(stage);
        currentGenerator = getNextGenerator();
        if (stage == Stage.SOURCE) {
            reschedule();
            return;
        }
    }
    if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
        notifyFailed();
    }
}
```

`currentGenerator.startNext()`进行获取，`currentGenerator`就是对应的几个ResourceCacheGenerator、DataCacheGenerator、SourceGenerator

接着看看`getnextStage()`方法

```java
private Stage getNextStage(Stage current) {
    switch (current) {
        case INITIALIZE:
            return diskCacheStrategy.decodeCachedResource()
                ? Stage.RESOURCE_CACHE : getNextStage(Stage.RESOURCE_CACHE);
        case RESOURCE_CACHE:
            return diskCacheStrategy.decodeCachedData()
                ? Stage.DATA_CACHE : getNextStage(Stage.DATA_CACHE);
        case DATA_CACHE:
            // Skip loading from source if the user opted to only retrieve the resource from cache.
            return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
        case SOURCE:
        case FINISHED:
            return Stage.FINISHED;
        default:
            throw new IllegalArgumentException("Unrecognized stage: " + current);
    }
}
```

就是判断有没有数据的

#### 从转换过的缓存中获取ResourceCacheGenerator



#### 从没有转换过的缓存中获取DataCacheGenerator



#### 从原始数据获取SourceGenerator

如果是第一次的话，最后是通过SourceGenerator来获取数据的，看看它的`startNext()`

```java
public boolean startNext() {
    if (dataToCache != null) {
        Object data = dataToCache;
        dataToCache = null;
        cacheData(data);
    }

    if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
        return true;
    }
    sourceCacheGenerator = null;

    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
        //获取对应的ModelLoader
        loadData = helper.getLoadData().get(loadDataListIndex++);
        if (loadData != null
            && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
                || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
            started = true;
            //获取ModelLoader的fetcher进行loadData
            loadData.fetcher.loadData(helper.getPriority(), this);
        }
    }
    return started;
}
```

1. 先使用 `DecodeHelper` 的 `getLoadData()` 方法从注册的映射表中找出当前的图片类型对应的 `ModelLoader`；
2. 然后使用它的 `DataFetcher` 的 `loadData()` 方法从原始的数据源中加载数据。

由于我们的图片时网络中的资源，在默认情况下会使用 Glide 内部的 `HttpUrlFetcher` 从网络中加载数据。其 `loadData()` 方法定义如下：

```java
public void loadData(@NonNull Priority priority,
                     @NonNull DataCallback<? super InputStream> callback) {
    long startTime = LogTime.getLogTime();
    try {
        //获取到了InputStram流了
        InputStream result = loadDataWithRedirects(glideUrl.toURL(), 0, null, glideUrl.getHeaders());
        //回调结果
        callback.onDataReady(result);
    } catch (IOException e) {
        ...
    } finally {
        ...
    }
}
```

通过`loadDataWithRedirects()`获取数据流

```java
private InputStream loadDataWithRedirects(URL url, int redirects, URL lastUrl,
                                          Map<String, String> headers) throws IOException {
    ...
	//创建UrlConnection
    urlConnection = connectionFactory.build(url);
    for (Map.Entry<String, String> headerEntry : headers.entrySet()) {
        urlConnection.addRequestProperty(headerEntry.getKey(), headerEntry.getValue());
    }
    urlConnection.setConnectTimeout(timeout);
    urlConnection.setReadTimeout(timeout);
    urlConnection.setUseCaches(false);
    urlConnection.setDoInput(true);
    urlConnection.setInstanceFollowRedirects(false);
    urlConnection.connect();
    stream = urlConnection.getInputStream();
    if (isCancelled) {
        return null;
    }
    final int statusCode = urlConnection.getResponseCode();
    if (isHttpOk(statusCode)) {
        return getStreamForSuccessfulRequest(urlConnection);
    } else if (isHttpRedirect(statusCode)) {
        String redirectUrlString = urlConnection.getHeaderField("Location");
        ...
        URL redirectUrl = new URL(url, redirectUrlString);
        cleanup();
        return loadDataWithRedirects(redirectUrl, redirects + 1, url, headers);
    } else if (statusCode == INVALID_STATUS_CODE) {
        throw new HttpException(statusCode);
    } else {
        throw new HttpException(urlConnection.getResponseMessage(), statusCode);
    }
}
```

这里就是通过UrlConnection去·打开网络，获取数据流，下面具体的就不再看了

### 将输入流转换为Drawable

在获取到数据流后，会将数据流进行回调，先是回调到SourceGenerator中，最后回调到DecodeJob中

```java
public void onDataFetcherReady(Key sourceKey, Object data, DataFetcher<?> fetcher,
                               DataSource dataSource, Key attemptedKey) {
    //赋值保存
    this.currentSourceKey = sourceKey;
    this.currentData = data;
    this.currentFetcher = fetcher;
    this.currentDataSource = dataSource;
    this.currentAttemptingKey = attemptedKey;
    if (Thread.currentThread() != currentThread) {
        runReason = RunReason.DECODE_DATA;
        callback.reschedule(this);
    } else {
        GlideTrace.beginSection("DecodeJob.decodeFromRetrievedData");
        try {
            //对数据进行decode得到期望得到的类型
            decodeFromRetrievedData();
        } finally {
            GlideTrace.endSection();
        }
    }
}
```

通过接口回调，然后将数据流及相应的一些参数赋值保存，然后调用`decodeFromRetrievedData()`进行数据转换

```java
private void decodeFromRetrievedData() {
    ...
    Resource<R> resource = null;
    try {
        //进行具体的数据转换
        resource = decodeFromData(currentFetcher, currentData, currentDataSource);
    } catch (GlideException e) {
        ...
    }
    if (resource != null) {
        notifyEncodeAndRelease(resource, currentDataSource);
    } else {
        runGenerators();
    }
}
```

接着调用内部的`decodeFromData()`

```java
private <Data> Resource<R> decodeFromData(DataFetcher<?> fetcher, Data data,
                                          DataSource dataSource) throws GlideException {
    try {
        if (data == null) {
            return null;
        }
        long startTime = LogTime.getLogTime();
        //进行decode
        Resource<R> result = decodeFromFetcher(data, dataSource);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Decoded result " + result, startTime);
        }
        return result;
    } finally {
        fetcher.cleanup();
    }
}
```

检查data后然后调用`decodeFromFetcher()`

```java
private <Data> Resource<R> decodeFromFetcher(Data data, DataSource dataSource)
    throws GlideException {
    //注意这里的R泛型是在创建Request时（也就是SingleRequest）指定的，也就是Drawable
    //获取LoadPath
    LoadPath<Data, ?, R> path = decodeHelper.getLoadPath((Class<Data>) data.getClass());
    return runLoadPath(data, dataSource, path);
}
```

接着调用`runLoadPath()`

```java
private <Data, ResourceType> Resource<R> runLoadPath(Data data, DataSource dataSource,
                                                     LoadPath<Data, ResourceType, R> path) throws GlideException {
    Options options = getOptionsWithHardwareConfig(dataSource);
    //将数据流封装到DataRewinder中
    DataRewinder<Data> rewinder = glideContext.getRegistry().getRewinder(data);
    try {
        // 调用LoadPath中的load方法
        return path.load(
            rewinder, options, width, height, new DecodeCallback<ResourceType>(dataSource));
    } finally {
        rewinder.cleanup();
    }
}
```

接着调用LoadPath中的load方法

```java
public Resource<Transcode> load(DataRewinder<Data> rewinder, @NonNull Options options, int width,
                                int height, DecodePath.DecodeCallback<ResourceType> decodeCallback) throws GlideException {
    //异常检查
    List<Throwable> throwables = Preconditions.checkNotNull(listPool.acquire());
    try {
        return loadWithExceptionList(rewinder, options, width, height, decodeCallback, throwables);
    } finally {
        listPool.release(throwables);
    }
}
```

接着调用`loadWithExceptionList()`

```java
private Resource<Transcode> loadWithExceptionList(DataRewinder<Data> rewinder,
                                                  @NonNull Options options,
                                                  int width, int height, DecodePath.DecodeCallback<ResourceType> decodeCallback,
                                                  List<Throwable> exceptions) throws GlideException {
    Resource<Transcode> result = null;
    //noinspection ForLoopReplaceableByForEach to improve perf
    for (int i = 0, size = decodePaths.size(); i < size; i++) {
        DecodePath<Data, ResourceType, Transcode> path = decodePaths.get(i);
        try {
            //进行decode
            result = path.decode(rewinder, width, height, options, decodeCallback);
        } catch (GlideException e) {
            exceptions.add(e);
        }
        ...
    }
	...
    return result;
}
```

接着通过DecodePath中的load方法进行decode

```java
public Resource<Transcode> decode(DataRewinder<DataType> rewinder, int width, int height,
                                  @NonNull Options options, DecodeCallback<ResourceType> callback) throws GlideException {
    //将数据流转换成我们需要的类型
    Resource<ResourceType> decoded = decodeResource(rewinder, width, height, options);
    //继续处理并显示
    Resource<ResourceType> transformed = callback.onResourceDecoded(decoded);
    return transcoder.transcode(transformed, options);
}
```

先看看`decodeResource()`方法

```java
private Resource<ResourceType> decodeResource(DataRewinder<DataType> rewinder, int width,
                                              int height, @NonNull Options options) throws GlideException {
    List<Throwable> exceptions = Preconditions.checkNotNull(listPool.acquire());
    try {
        return decodeResourceWithList(rewinder, width, height, options, exceptions);
    } finally {
        listPool.release(exceptions);
    }
}

private Resource<ResourceType> decodeResourceWithList(DataRewinder<DataType> rewinder, int width,
                                                      int height, @NonNull Options options, List<Throwable> exceptions) throws GlideException {
    Resource<ResourceType> result = null;
    //noinspection ForLoopReplaceableByForEach to improve perf
    for (int i = 0, size = decoders.size(); i < size; i++) {
        ResourceDecoder<DataType, ResourceType> decoder = decoders.get(i);
        try {
            DataType data = rewinder.rewindAndGet();
            if (decoder.handles(data, options)) {
                data = rewinder.rewindAndGet();
                //根据ResourceDecoder进行具体的转换
                result = decoder.decode(data, width, height, options);
            }
            ...
        } catch (IOException | RuntimeException | OutOfMemoryError e) {
            ...
        }
		...
    }
	...
    return result;
}
```

`decodeResource()`方法会调用内部的`decodeResourceWithList()`进行数据转换，在`decodeResourceWithList()`方法中，会获取所有的ResourceDecoder，然后调用decode方法进行具体的转换

`ResourceDecoder` 具有多个实现类，比如 `BitmapDrawableDecoder`、`ByteBufferBitmapDecoder`等。从名字也可以看出来是用来将一个类型转换成另一个类型的。

在我们的程序中会使用 `ByteBufferBitmapDecoder` 来将 `ByteBuffer` 转成 `Bitmap`。它最终会在 `Downsampler` 的 `decodeStream()` 方法中调用 `BitmapFactory` 的 `decodeStream()` 方法来从输入流中得到 Bitmap。（我们的 `ByteBuffer` 在  `ByteBufferBitmapDecoder` 中先被转换成了输入流。）

```java
public class ByteBufferBitmapDecoder implements ResourceDecoder<ByteBuffer, Bitmap> {
    private final Downsampler downsampler;
    public ByteBufferBitmapDecoder(Downsampler downsampler) {
        this.downsampler = downsampler;
    }

    @Override
    public boolean handles(@NonNull ByteBuffer source, @NonNull Options options) {
        return downsampler.handles(source);
    }

    @Override
    public Resource<Bitmap> decode(@NonNull ByteBuffer source, int width, int height,
                                   @NonNull Options options)
        throws IOException {
        //将数据流还原成InputStream
        InputStream is = ByteBufferUtil.toStream(source);
        //转换并返回Resource<Bitmap>
        return downsampler.decode(is, width, height, options);
    }
}
```

在ByteBufferBitmapDecoder的decode方法中，会先将数据流还原成InputStream类型，接着调用了`downsampler.decode(is, width, height, options);`

```java
public Resource<Bitmap> decode(InputStream is, int outWidth, int outHeight,
                               Options options) throws IOException {
    return decode(is, outWidth, outHeight, options, EMPTY_CALLBACKS);
}

public Resource<Bitmap> decode(InputStream is, int requestedWidth, int requestedHeight,
                               Options options, DecodeCallbacks callbacks) throws IOException {
    ...
    try {
        //转换成一个Bitmap对象
        Bitmap result = decodeFromWrappedStreams(is, bitmapFactoryOptions, downsampleStrategy, decodeFormat, isHardwareConfigAllowed, requestedWidth, requestedHeight, fixBitmapToRequestedDimensions, callbacks);
        return BitmapResource.obtain(result, bitmapPool);
    } finally {
       ...
    }
}

private Bitmap decodeFromWrappedStreams(InputStream is,
                                        BitmapFactory.Options options, DownsampleStrategy downsampleStrategy,
                                        DecodeFormat decodeFormat, boolean isHardwareConfigAllowed, int requestedWidth,
                                        int requestedHeight, boolean fixBitmapToRequestedDimensions,
                                        DecodeCallbacks callbacks) throws IOException {
    ...
    Bitmap downsampled = decodeStream(is, options, callbacks, bitmapPool);
    callbacks.onDecodeComplete(bitmapPool, downsampled);
	...
}

private static Bitmap decodeStream(InputStream is, BitmapFactory.Options options,
                                   DecodeCallbacks callbacks, BitmapPool bitmapPool) throws IOException {
    ...
    try {
        result = BitmapFactory.decodeStream(is, null, options);
    } catch (IllegalArgumentException e) {
        ...
    } finally {
        TransformationUtils.getBitmapDrawableLock().unlock();
    }
	...
    return result;
}
```

在Downsampler中的decode方法会调用重载的另一个decode方法，然后通过`decodeFromWrappedStreams()`转换为Bitmap对象，在`decodeFromWrappedStreams()`中又会调用`decodeStream()`方法，最后通过`BitmapFactory.decodeStream()`将InputSteam转换为Bitmap，然后一层一层返回



回到之前的DecodePath的decode方法

```java
public Resource<Transcode> decode(DataRewinder<DataType> rewinder, int width, int height,
                                  Options options, DecodeCallback<ResourceType> callback) throws GlideException {
    //返回到这里来
    Resource<ResourceType> decoded = decodeResource(rewinder, width, height, options); // 1
    //进行图片处理，如圆角等
    Resource<ResourceType> transformed = callback.onResourceDecoded(decoded); // 2
    //这里会将处理后的Bitmap转为Drawable
    return transcoder.transcode(transformed, options); // 3
}
```

这里会调用 `callback` 的方法进行回调，它最终会回调到 `DecodeJob` 的 `onResourceDecoded()` 方法。其主要的逻辑是根据我们设置的参数进行变化，也就是说，如果我们使用了 `centerCrop` 等参数，那么这里将会对其进行处理。这里的 `Transformation` 是一个接口，它的一系列的实现都是对应于 `scaleType` 等参数的

```java
<Z> Resource<Z> onResourceDecoded(DataSource dataSource,
                                  @NonNull Resource<Z> decoded) {
    @SuppressWarnings("unchecked")
    Class<Z> resourceSubClass = (Class<Z>) decoded.get().getClass();
    Transformation<Z> appliedTransformation = null;
    Resource<Z> transformed = decoded;
    //对图片进行变换
    if (dataSource != DataSource.RESOURCE_DISK_CACHE) {
        appliedTransformation = decodeHelper.getTransformation(resourceSubClass);
        transformed = appliedTransformation.transform(glideContext, decoded, width, height);
    }
    ...
    Resource<Z> result = transformed;
    boolean isFromAlternateCacheKey = !decodeHelper.isSourceKey(currentSourceKey);
    //进行数据缓存
    if (diskCacheStrategy.isResourceCacheable(isFromAlternateCacheKey, dataSource,
                                              encodeStrategy)) {
        if (encoder == null) {
            throw new Registry.NoResultEncoderAvailableException(transformed.get().getClass());
        }
        final Key key;
        switch (encodeStrategy) {
            case SOURCE:
                key = new DataCacheKey(currentSourceKey, signature);
                break;
            case TRANSFORMED:
                key =
                    new ResourceCacheKey(
                    decodeHelper.getArrayPool(),
                    currentSourceKey,
                    signature,
                    width,
                    height,
                    appliedTransformation,
                    resourceSubClass,
                    options);
                break;
            default:
                throw new IllegalArgumentException("Unknown strategy: " + encodeStrategy);
        }

        LockedResource<Z> lockedResult = LockedResource.obtain(transformed);
        deferredEncodeManager.init(key, encoder, lockedResult);
        result = lockedResult;
    }
    return result;
}
```

在上面的方法中对图形进行变换之后还会根据图片的缓存策略决定对图片进行缓存。然后这个方法就直接返回了我们变换之后的图象

接着回到上一个的return：`return transcoder.transcode(transformed, options)`会返回`Resouces<BitmapDrawable>`对象，transcoder是`ResourceTranscoder<ResourceType, Transcode>`，那么我们这里就是一个BitmapDrawableTranscoder

```java
public Resource<BitmapDrawable> transcode(@NonNull Resource<Bitmap> toTranscode,
                                          @NonNull Options options) {
    //进行转换
    return LazyBitmapDrawableResource.obtain(resources, toTranscode);
}
```

通过BitmapDrawableTranscoder会返回一个`Resource<BitmapDrawable>`，也就是说这里就将Bitmap转换为了BitmapDrawable，然后一层一层返回

### 将Drawable显示到ImageView上

前面说到不断向上 `return` 进行返回。所以，我们又回到了 `DecodeJob` 的 `decodeFromRetrievedData()` 方法

```java
private void decodeFromRetrievedData() {
    ...
    Resource<R> resource = null;
    try {
        //这是前面分析的
        resource = decodeFromData(currentFetcher, currentData, currentDataSource);
    } catch (GlideException e) {
        e.setLoggingDetails(currentAttemptingKey, currentDataSource);
        throwables.add(e);
    }
    if (resource != null) {
        //接下来执行到这里
        notifyEncodeAndRelease(resource, currentDataSource);
    } else {
        runGenerators();
    }
}
```



```java
private void notifyEncodeAndRelease(Resource<R> resource, DataSource dataSource) {
    ...
    Resource<R> result = resource;
    ...
    notifyComplete(result, dataSource);
    ...
}
```

接着会调用`notifyComplete()`方法

```java
private void notifyComplete(Resource<R> resource, DataSource dataSource) {
    setNotifiedOrThrow();
    callback.onResourceReady(resource, dataSource);
}
```

最后通过callback进行回调，callback是在创建DecodeJob时传进来的，前面有，传进来的就是EngineJob（实现了DecodeJob.Callback），回调到EngineJob的`onResourceReady()`

```java
public void onResourceReady(Resource<R> resource, DataSource dataSource) {
    synchronized (this) {
        this.resource = resource;
        this.dataSource = dataSource;
    }
    notifyCallbacksOfResult();
}
```

将结果保存在EngineJob中，同时调用`notifyCallbacksOfResult();`通知

```java
void notifyCallbacksOfResult() {
    ResourceCallbacksAndExecutors copy;
    Key localKey;
    EngineResource<?> localResource;
    synchronized (this) {
      ...
      copy = cbs.copy();
      ...
    }
    for (final ResourceCallbackAndExecutor entry : copy) {
        //进行结果回调
        entry.executor.execute(new CallResourceReady(entry.cb));
    }
    ...
}
```

最后遍历所有的ResourceCallbackAndExecutor，这个里面包含了Engine的cb，也就是SingleRequest实现的回调接口（在SingleRequest调用engine的load方法时，会将自己实现的callback传入，然后在Engine中创建DecodeJob后，在执行DecodeJob之前会先将这个回调设置到EngineJob中`addCallback()`，同时将主线程的executor也传进来了，在这个方法就会添加到cbs中），也就是我们这里遍历的copy就是cbs的一个复制（cbs就是一个Callback和Executor的集合）

所以在这里执行的`entry.executor.execute(new CallResourceReady(entry.cb));`其实是执行的SingleRequest传进来的Executor（这个Executor是RequestBuilder的load方法时获取的主线程Executor），执行的回调接口就是SingleRequest本身

看看CallResourceReady

```java
private class CallResourceReady implements Runnable {

    private final ResourceCallback cb;

    CallResourceReady(ResourceCallback cb) {
        this.cb = cb;
    }

    @Override
    public void run() {
        synchronized (EngineJob.this) {
            if (cbs.contains(cb)) {
                // Acquire for this particular callback.
                engineResource.acquire();
                callCallbackOnResourceReady(cb);
                removeCallback(cb);
            }
            decrementPendingCallbacks();
        }
    }
}
```

就是一个线程，通过主线程的线程池执行，主要就是`callCallbackOnResourceReady()`来进行回调

```java
synchronized void callCallbackOnResourceReady(ResourceCallback cb) {
    try {
        //进行回调
        cb.onResourceReady(engineResource, dataSource);
    } catch (Throwable t) {
        throw new CallbackException(t);
    }
}
```

就这样，通过主线程的线程池，将结果回调到SingleRequest中了

接着看回调`onResourceReady()`

```java
public synchronized void onResourceReady(Resource<?> resource, DataSource dataSource) {
    ...
        onResourceReady((Resource<R>) resource, (R) received, dataSource);
}

private synchronized void onResourceReady(Resource<R> resource, R result, DataSource dataSource) {
    ...
    try {
        ...
        if (!anyListenerHandledUpdatingTarget) {
            Transition<? super R> animation =
                animationFactory.build(dataSource, isFirstResource);
            //将结果传给了target，这个前面说过，是DrawableImageViewTarget
            target.onResourceReady(result, animation);
        }
    } finally {
        isCallingCallbacks = false;
    }
    notifyLoadSuccess();
}
```

`onResourceReady()`调用了内部的`onResourceReady()`方法，在这个方法我们主要看到`target.onResourceReady(result, animation);`这一行，这个target就是我们在最后一步into方法传入ImageView时，会去构建一个Target对象（实际上DrawableImageViewTarget对象），这个target就持有了我们的ImageView，接下来看看DrawableImageViewTarget

通过看源码知道，DrawableImageViewTarget没有`onResourceReady()`方法的实现，但是它是继承自ImageViewTarget的，所以看看ImageViewTarget

```java
public void onResourceReady(@NonNull Z resource, @Nullable Transition<? super Z> transition) {
    //没有动画的时候
    if (transition == null || !transition.transition(resource, this)) {
        setResourceInternal(resource);
    } else {
        maybeUpdateAnimatable(resource);
    }
}
```

这里就可以知道了，我们会通过`setResourceInternal()`这加载显示图片

```java
private void setResourceInternal(@Nullable Z resource) {
    setResource(resource);
    maybeUpdateAnimatable(resource);
}
```

会去调用`setResource()`这个抽象方法，那么针对我们前面的Target是DrawableImageViewTarget，就看DrawableImageViewTarget中的实现吧

```java
protected void setResource(@Nullable Drawable resource) {
    view.setImageDrawable(resource);
}
```

view就是我们设置的ImageView，这里就直接调用View的`setImageDrawable()`就完了，这样，图片就展示出来了



## 总结

- with

    ![](https://user-gold-cdn.xitu.io/2019/1/6/16823422d928fb82?imageslim)

- load

- into



## 特别感谢

- [Glide 系列-2：主流程源码分析（4.8.0）](<https://juejin.im/post/5c31fbdff265da610e803d4e>)
- [Glide 源码分析](<https://juejin.im/entry/586766331b69e60063d889ea>)
- [Android源码分析：这是一份详细的图片加载库Glide源码讲解攻略](<https://blog.csdn.net/carson_ho/article/details/79212841>)
- [Glide 源码分析解读-基于最新版Glide 4.9.0](<https://www.jianshu.com/p/9bb50924d42a>)