---
title: Android Retrofit2源码分析
tag: Android第三方框架
date: 2019-06-02
---

# Android Retrofit2源码分析

- [介绍](##介绍)
- [源码分析](##源码分析)
    - [Retrofit的创建](###Retrofit的创建)
    - [动态代理得到接口对象](###创建接口对象)
    - [执行enqueue](###执行enqueue)
    - [OkHttpCall和数据解析](###OkHttpCall)
- [总结](##总结)

## 介绍

Retrofit2是Retrofit的一个升级版，底层基于OkHttp3的一个网络请求框架，更直白的说，Retrofit2就是对OkHttp3的一种封装，可以让使用者避免很多重复的网络请求代码，同时灵活性很高，可以自定义OkHttpClient、数据转换器、请求转换器等

在Retrofit2中的改进

- 使用了运行时注解，可以根据需要来将业务调用接口转换成Http请求接口
- 可以取消正在进行中的业务，Service的模式变成Call的形式
- Converter从Retrofit2中删除，通过addConverterFactory来添加转换器
- 需要OkHttp的支持，网络请求完全由OkHttp3进行

 **看一下一些重要的类**

- Call：interface，像服务器发送请求并返回响应的调用
- CallAdapter：interface，Call的适配器，用来包装Call
- CallBack：interface，Call执行时的回调
- Converter：interface，数据转换器，将一个对象转换为另一个对象
- CallAdapter.Factory：abstract class，数据转换器Converter的工厂，可以转换结果和请求
- RequestFactory：class，创建OkHttp请求的Request
- RequestFactoryParser ：class，解析GitHubService.listRepos()方法的注解和参数，生成RequestFactory。（会用到requestBodyConverter，stringConverter）
- OkHttpCall：class，实现Call接口，获取传入的Call（代理Call，通过Retrofit.callFactory生成的）执行请求，获取数据并使用responseConverter进行解析
- Retrofit：class，这个网络请求的配置和控制
- Retrofit.Builder：class，Builder模式创建Retrofit

## 源码分析

### 简单使用

业务接口

```java
public interface ApiService {
    public final static String BASE_URL = "http://";

    @GET("user")
    Call<User> getUser();
}
```

调用

```java
	//创建retrofit，通过Builder模式
	Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(ApiService.BASE_URL)
        		//设置OkClient 默认会自己new一个 可以不用配置
                .client(new OkHttpClient())
        		//对数据的转换 GSON 可以不用
                .addConverterFactory(GsonConverterFactory.create())
        		//对RxJava2的适配，可以不用
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .build();
	//创建接口对象ApiService
	ApiService apiService = retrofit.create(ApiService.class);
	Call<User> userCall = apiService.getUser();
    userCall.enqueue(new Callback<User>() {
		@Override
	    public void onResponse(Call<User> call, Response<User> response) {
	    
	    }

        @Override
		public void onFailure(Call<User> call, Throwable t) {

	    }
    });
```

这就是一个正常的请求流程了

### Retrofit的创建

首先我们就来看看Retrofit的创建

Retrofit的创建很明显就是通过Builder模式来构造的，通过Builder模式来设置需要的参数，最后通过build进行创建

先看看Builder

```java
public final class Retrofit {
	...
    public static final class Builder {
        //平台，可以获取很多默认参数
        private final Platform platform;
        //okhhtp3的
        private @Nullable okhttp3.Call.Factory callFactory;
        //请求地址
        private @Nullable HttpUrl baseUrl;
        //存储converters 转化数据
        private final List<Converter.Factory> converterFactories = new ArrayList<>();
        //存储callAdapters 对call进行转化
        private final List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>();
        //传递回调到ui线程
        private @Nullable Executor callbackExecutor;
        private boolean validateEagerly;

        Builder(Platform platform) {
          this.platform = platform;
        }

        public Builder() {
          this(Platform.get());
        }
        //通过Retrofit构建
        Builder(Retrofit retrofit) {
            ...
        }
		//配置OkClient
        public Builder client(OkHttpClient client) {
          return callFactory(checkNotNull(client, "client == null"));
        }
		//配置callFactory
        public Builder callFactory(okhttp3.Call.Factory factory) {
          this.callFactory = checkNotNull(factory, "factory == null");
          return this;
        }
		//请求地址 一个Retrofit一个url
        public Builder baseUrl(String baseUrl) {
          checkNotNull(baseUrl, "baseUrl == null");
          return baseUrl(HttpUrl.get(baseUrl));
        }

        public Builder baseUrl(HttpUrl baseUrl) {
          checkNotNull(baseUrl, "baseUrl == null");
          List<String> pathSegments = baseUrl.pathSegments();
          if (!"".equals(pathSegments.get(pathSegments.size() - 1))) {
            throw new IllegalArgumentException("baseUrl must end in /: " + baseUrl);
          }
          this.baseUrl = baseUrl;
          return this;
        }
		//转换器工厂
        public Builder addConverterFactory(Converter.Factory factory) {
          converterFactories.add(checkNotNull(factory, "factory == null"));
          return this;
        }
		//适配器工厂
        public Builder addCallAdapterFactory(CallAdapter.Factory factory) {
          callAdapterFactories.add(checkNotNull(factory, "factory == null"));
          return this;
        }

        public Builder callbackExecutor(Executor executor) {
          this.callbackExecutor = checkNotNull(executor, "executor == null");
          return this;
        }
		//创建Retrofit实例
        public Retrofit build() {
            if (baseUrl == null) {
            	throw new IllegalStateException("Base URL required.");
            }

            okhttp3.Call.Factory callFactory = this.callFactory;
            if (callFactory == null) {
            	callFactory = new OkHttpClient();
            }

          	Executor callbackExecutor = this.callbackExecutor;
          	if (callbackExecutor == null) {
              	//有一个默认实现 根据platform
              	callbackExecutor = platform.defaultCallbackExecutor();
          	}
          	List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories); 
            //默认实现的callAdapter
  		callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));
          	List<Converter.Factory> converterFactories = new ArrayList<>(
 1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

          	converterFactories.add(new BuiltInConverters());
          	converterFactories.addAll(this.converterFactories);
          	converterFactories.addAll(platform.defaultConverterFactories());

          	return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories), unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
        }
	}
}
```

Retrofit的Builder是一个静态内部类，通过Builder，我们可以配置以下

- converterFactory：针对数据，对数据转换

- callAdapterFactory：针对call，对call进行转化

- callbackExecutor：用来将回调传递到UI线程；

    利用platform对象，对平台进行判断，判断主要是利用Class.forName("")进行查找，如果是Android平台，会自定义一个Executor对象，并且利用Looper.getMainLooper()实例化一个handler对象，在Executor内部通过handler.post(runnable)

- okHttpClient：肯定需要啊，进行网络请求

- baseUrl：最重要的，请求地址

- validateEagerly：在调用时的验证配置的标志量，不设置为null

在配置Factories的时候，可以设置多个，然后是放在其对应的list中的

最后`build()`方法new出了一个Retrofit，并将参数通过Retrofit的构造方法传进去

**这里需要注意的是**如果baseUrl为null，会抛出异常；如果没有设置callFactory，则默认直接`new OkHttpClient()`；如果没有设置callbackExecutor，就会通过platform来获取一个默认的；然后将callAdapterFactories添加到一个新的list中（如果没有配置callAdapterFactory，就会根据platform获取默认的实现——ExecutorCallAdapterFactory）；同样将conerterFactories添加到新的list中，同时添加了默认的几个（如BuiltInConverters和从platform中获取的）

我们来看看这些默认值的获取，毕竟我们可以不设置这些参数

- 首先就是platform是啥

    platform是我们Retrofit运行的平台，在创建Builder的时候得到的

    ```java
    public Builder() {
    	this(Platform.get());
    }
    ```

    接着看看这个具体的get方法

    ```java
    class Platform {
      	private static final Platform PLATFORM = findPlatform();
    
      	static Platform get() {
        	return PLATFORM;
      	}
    
      	private static Platform findPlatform() {
        	try {
          		Class.forName("android.os.Build");
          		if (Build.VERSION.SDK_INT != 0) {
            		return new Android();
          		}
        	} catch (ClassNotFoundException ignored) {
        	}
        	try {
          		Class.forName("java.util.Optional");
          		return new Java8();
        	} catch (ClassNotFoundException ignored) {
        	}
        	return new Platform();
     	}
        ...
    }
    ```

    通过get方法获取的是Platform的单例，最后是通过`findPlatform()`来获取的

    在`findPlatform()`中，通过Class.forName("")进行查找，如果是Android平台，返回了一个Android平台的Platform（就是继承自Platform的内部类，Java也是同样的），如果是Java平台，返回的是Java8的，否则返回一个什么都没实现的Platform（通过查看源码可以知道，很多方法返回的都是null）

    主要看看Android平台的

    ```java
    static class Android extends Platform {
        ...
    }
    ```

    Platform.Android这个内部类继承自Platform，根据特性，进行自己的方法实现，具体我们就看后面的吧

- `callbackExecutor = platform.defaultCallbackExecutor()`这个时候就应该是Platform.Android中的`defaultCallbackExecutor()`方法

    ```java
    @Nullable Executor defaultCallbackExecutor() {
        return new MainThreadExecutor();
    }
    
    static class MainThreadExecutor implements Executor {
    	private final Handler handler = new Handler(Looper.getMainLooper());
          	@Override public void execute(Runnable r) {
            	handler.post(r);
    	}
    }
    ```

    `defaultCallbackExecutor()`方法new了一个MainThreadExecutor，这个类继承子Executor，同时持有了主线程的Looper，然后通过post方法，将回调传递到UI线程

- `platform.defaultCallAdapterFactories(callbackExecutor)`方法，同上

    ```java
    @Override List<? extends CallAdapter.Factory> defaultCallAdapterFactories(
            @Nullable Executor callbackExecutor) {
    	if (callbackExecutor == null) throw new AssertionError();
    		ExecutorCallAdapterFactory executorFactory = new ExecutorCallAdapterFactory(callbackExecutor);
    	return Build.VERSION.SDK_INT >= 24
            ? asList(CompletableFutureCallAdapterFactory.INSTANCE, executorFactory)
            : singletonList(executorFactory);
        }
    ```

    由于前面的callbackExecutor 是MainThreadExecutor，所以不会抛出异常，同时创建了ExecutorCallAdapterFactory，根据Android的版本做了一些调整，大于24的则多了一个CompletableFutureCallAdapterFactory，这里就不继续看这两个CallAdapter的实现了，后面会看到，到时候我们再继续看

- `platform.defaultConverterFactories()`，同上

    ```java
    @Override List<? extends Converter.Factory> defaultConverterFactories() {
          return Build.VERSION.SDK_INT >= 24
              ? singletonList(OptionalConverterFactory.INSTANCE)
              : Collections.<Converter.Factory>emptyList();
    }
    ```

    根据Android API是否大于等于24返回，24及以上版本的，返回了OptionalConverterFactory。以下的，返回了一个空的

接着看看Retrofit的的一些成员属性和构造方法

```java
public final class Retrofit {
    //ServiceMethod缓存
    private final Map<Method, ServiceMethod<?>> serviceMethodCache = new ConcurrentHashMap<>();

    final okhttp3.Call.Factory callFactory;
    final HttpUrl baseUrl;
    final List<Converter.Factory> converterFactories;
    final List<CallAdapter.Factory> callAdapterFactories;
    final @Nullable Executor callbackExecutor;
    final boolean validateEagerly;

    Retrofit(okhttp3.Call.Factory callFactory, HttpUrl baseUrl, 
             List<Converter.Factory> converterFactories, 
             List<CallAdapter.Factory> callAdapterFactories,
      		@Nullable Executor callbackExecutor, boolean validateEagerly) {
    	this.callFactory = callFactory;
    	this.baseUrl = baseUrl;
    	this.converterFactories = converterFactories; // Copy+unmodifiable at call site.
    	this.callAdapterFactories = callAdapterFactories; // Copy+unmodifiable at call site.
    	this.callbackExecutor = callbackExecutor;
    	this.validateEagerly = validateEagerly;
  	}
    ...
}
```

Retrofit的创建我们就看这么多

### 创建接口对象

获取到Retrofit后，通过Retrofit的`create()`方法，我们得到了ApiService

我们就看看这个create方法做了什么

```java
public final class Retrofit {
    ...
	public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    //创建Retrofit时配置的，默认null
    if (validateEagerly) {
        eagerlyValidateMethods(service);
    }
    //返回动态代理对象
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();
          private final Object[] emptyArgs = new Object[0];

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
          }
        });
    }
}
```

这里就比较关键了，`create()`方法根据传进来的接口ApiService直接返回了一个动态代理对象，这是Java中的动态代理；当我们调用ApiSerivce时就会调用代理对象的`invoke()`方法

这后面就跟2.4.0版本有所不同了，但大致的思想是没变了得

在invoke方法中，最后返回了`loadServiceMethod(method).invoke(args != null ? args : emptyArgs);`它，来看看是个啥吧

`loadServiceMethod(method)`就是Retrofit中的一个方法

```java
	ServiceMethod<?> loadServiceMethod(Method method) {
        ServiceMethod<?> result = serviceMethodCache.get(method);
        if (result != null) return result;

        synchronized (serviceMethodCache) {
          result = serviceMethodCache.get(method);
          if (result == null) {
            result = ServiceMethod.parseAnnotations(this, method);
            serviceMethodCache.put(method, result);
          }
        }
        return result;
  	}
```

从代码中可以看出，通过Method，最后返回了一个ServiceMethod对象，这个就和之前2.0版本的ServiceMethod对应了

在这个方法中，先是通过method在serviceMethodCache去获取ServiceMethod；serviceMethodCache就是我们前面看Retrofit中的一个Map成员属性，是一个ConcurrentHashMap，Method为key，ServiceMethod为value的缓存；如果result（ServiceMethod）不为null就直接返回，否则就要根据这个Method进行创建一个ServiceMethod

首先，synchronized再一次对serviceMethodCache加锁，保证安全，再一次确定缓存中没有对应的ServiceMethod才会去通过`ServiceMethod.parseAnnotations(this, method)`创建，然后存入缓存

```java
abstract class ServiceMethod<T> {
    static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
        //根据注解解析出
    	RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

    	Type returnType = method.getGenericReturnType();
    	if (Utils.hasUnresolvableType(returnType)) {
      		throw methodError(method, "Method return type must not include a type variable or wildcard: %s", returnType);
    	}
    	if (returnType == void.class) {
      		throw methodError(method, "Service methods cannot return void.");
    	}

    	return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  	}

  	abstract T invoke(Object[] args);
}
```

ServiceMethod现在是一个抽象类（记得2.4.0不是的），`parseAnnotations()`方法中，先是获取了requestFactory，先来看看这个`RequestFactory.parseAnnotations(retrofit, method);`方法

```java
static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
    return new Builder(retrofit, method).build();
}
...
static final class Builder {
	Builder(Retrofit retrofit, Method method) {
      	this.retrofit = retrofit;
      	this.method = method;
      	this.methodAnnotations = method.getAnnotations();
      	this.parameterTypes = method.getGenericParameterTypes();
      	this.parameterAnnotationsArray = method.getParameterAnnotations();
    }
    ...
}
```

这个方法就是通过builder模式创建了RequestFactory，通过Method解析了注解、参数类型、参数注解等

最后通过`HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);`创建了一个ServiceMethod并返回

ServiceMethod中的invoke方法是一个抽象方法，稍后会讲到

接着看一下`HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);`

```java
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
    CallAdapter<ResponseT, ReturnT> callAdapter = createCallAdapter(retrofit, method);
    Type responseType = callAdapter.responseType();
    if (responseType == Response.class || responseType == okhttp3.Response.class) {
      throw methodError(method, "'"
          + Utils.getRawType(responseType).getName()
          + "' is not a valid response body type. Did you mean ResponseBody?");
    }
    if (requestFactory.httpMethod.equals("HEAD") && !Void.class.equals(responseType)) {
      throw methodError(method, "HEAD method must use Void as response type.");
    }

    Converter<ResponseBody, ResponseT> responseConverter =
        createResponseConverter(retrofit, method, responseType);

    okhttp3.Call.Factory callFactory = retrofit.callFactory;
    return new HttpServiceMethod<>(requestFactory, callFactory, callAdapter, responseConverter);
}
```

在HttpServiceMethod的parseAnnotations方法中（这还是个static方法），根据retrofit和method先后获取了callAdapter、responseType、responseConverter、okClient

- createCallAdapter

    ```java
    private static <ResponseT, ReturnT> CallAdapter<ResponseT, ReturnT> createCallAdapter(
          Retrofit retrofit, Method method) {
        Type returnType = method.getGenericReturnType();
        Annotation[] annotations = method.getAnnotations();
        try {
          //noinspection unchecked
          return (CallAdapter<ResponseT, ReturnT>) retrofit.callAdapter(returnType, annotations);
        } catch (RuntimeException e) { // Wide exception range because factories are user code.
          throw methodError(method, e, "Unable to create call adapter for %s", returnType);
        }
    }
    ```

    根据Method的返回类型和注解，从retrofit中取出了对应的callAdapter

    ```java
    public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
        return nextCallAdapter(null, returnType, annotations);
    }
      
    public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
          Annotation[] annotations) {
        checkNotNull(returnType, "returnType == null");
        checkNotNull(annotations, "annotations == null");
    
        int start = callAdapterFactories.indexOf(skipPast) + 1;
        for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
          CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
          if (adapter != null) {
            return adapter;
          }
        }
        ...
        throw new IllegalArgumentException(builder.toString());
    }
    ```

    在retrofit中通过nextCallAdapter来获取CallAdapter，遍历callAdapterFactories中的所有CallAdapter，返回符合条件的CallAdapter，从前面Retrofit的创建知道，如果没有设置，则会获取默认的ExecutorCallAdapterFactory

- createResponseConverter

    ```java
    private static <ResponseT> Converter<ResponseBody, ResponseT> createResponseConverter(
          Retrofit retrofit, Method method, Type responseType) {
        Annotation[] annotations = method.getAnnotations();
        try {
          return retrofit.responseBodyConverter(responseType, annotations);
        } catch (RuntimeException e) { // Wide exception range because factories are user code.
          throw methodError(method, e, "Unable to create converter for %s", responseType);
        }
    }
    ```

    根据Method的注解和response类型，从retrofit中获取了responseConverter结果转换器

    ```java
    public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[] annotations) {
        return nextResponseBodyConverter(null, type, annotations);
    }
    
    public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
          @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
        checkNotNull(type, "type == null");
        checkNotNull(annotations, "annotations == null");
    
        int start = converterFactories.indexOf(skipPast) + 1;
        for (int i = start, count = converterFactories.size(); i < count; i++) {
          Converter<ResponseBody, ?> converter =
              converterFactories.get(i).responseBodyConverter(type, annotations, this);
          if (converter != null) {
            //noinspection unchecked
            return (Converter<ResponseBody, T>) converter;
          }
        }
    	...
        throw new IllegalArgumentException(builder.toString());
    }
    ```

    responseConverter也是同样的方式，遍历所有的converter，从converterFactories返回符合的responseConverter，如果没有设置，则会获取默认的null（API>=24是一个OptionalConverterFactory），这可以从前面Retrofit的创建知道

在HttpServiceMethod的parseAnnotations方法最后，直接new了HttpServiceMethod，这就是现在关键的大BOSS了

```java
final class HttpServiceMethod<ResponseT, ReturnT> extends ServiceMethod<ReturnT> {
    ...
    private HttpServiceMethod(RequestFactory requestFactory, okhttp3.Call.Factory callFactory,
          CallAdapter<ResponseT, ReturnT> callAdapter,
          Converter<ResponseBody, ResponseT> responseConverter) {
        this.requestFactory = requestFactory;
        this.callFactory = callFactory;
        this.callAdapter = callAdapter;
        this.responseConverter = responseConverter;
    }
    ...
}
```

这还是私有的构造方法，也就意味着我们只能通过前面的方式来构建HttpServiceMethod，然后保留了这些参数requestFacotry、callFactory、callAdapter、responseConverter等

从HttpServiceMethod的继承关系来看，继承自ServiecMethod，所以前面一连串的调用下来，最后返回了HttpServiceMethod

回到前面Retrofit的`create()`方法，现在我们知道了`loadServiceMethod(method)`方法就是返回了一个ServiceMethod（具体实现是HttPServiceMethod），在Retrofit的`create()`方法中，通过动态代理得到动态代理对象，会调用invoke方法，invoke方法返回的是`loadServiceMethod(method).invoke(args != null ? args : emptyArgs)`，现在我们就知道后面调用的invoke方法是ServiceMethod中的invoke方法了

```java
abstract class ServiceMethod<T> {
    ...
	abstract T invoke(Object[] args);
}
```

前面创建ServiceMethod的时候，我们知道真正实现是HttpServiceMethod，那就接着看HttpServiceMethod的invoke方法

```java
@Override ReturnT invoke(Object[] args) {
    return callAdapter.adapt(
        new OkHttpCall<>(requestFactory, args, callFactory, responseConverter));
}
```

在HttpServiceMethod的invoke方法中，创建了一个OkHttpCall（网络请求需要）到callAdapter中，将OkHttpCall通过callAdapter转化为需要的Call并返回（例如`getUser()`方法。需要的Call是`Call<User>`，那就创建ServiceMethod的时候就是User这个泛型，最后返回的就是`Call<User>`），如果我们前面Retrofit通过Builder设置了`addCallAdapterFactory(RxJava2CallAdapterFactory.create())`，那么返回的就是RxJava2对应的Call，就可以很好的配合RxJava2使用，这就不得不感叹适配器模式的伟大

在这里我们知道是new了一个OkHttpCall的，接着就create这个方法就完了。通过`ApiService.getUser()`来调用`getUser()`方法，会生成对应的动态代理对象，调用接口中对应的方法，通过注解等获取`getUser()`的参数、返回类型等信息，最终得到一个需要的Call对象

从前面分析知道，callAdapter在Retrofit创建时没有配置的话，也会有一个默认的CallAdapterFactory——ExecutorCallAdapterFactory

```java
final class ExecutorCallAdapterFactory extends CallAdapter.Factory {
  	final Executor callbackExecutor;

  	ExecutorCallAdapterFactory(Executor callbackExecutor) {
    	this.callbackExecutor = callbackExecutor;
  	}

  	@Override public @Nullable CallAdapter<?, ?> get(
      Type returnType, Annotation[] annotations, Retrofit retrofit) {
    	if (getRawType(returnType) != Call.class) {
      		return null;
    	}
    	final Type responseType = Utils.getCallResponseType(returnType);
    	return new CallAdapter<Object, Call<?>>() {
      		@Override public Type responseType() {
       			return responseType;
     		}

      		@Override public Call<Object> adapt(Call<Object> call) {
        		return new ExecutorCallbackCall<>(callbackExecutor, call);
      		}
		};
	}
	...
}
```

那么在ExecutorCallAdapterFactory的adapt方法，返回了ExecutorCallbackCall，这个类是ExecutorCallAdapterFactory的内部类，一个Call的实现类

然后我们关心一下ExecutorCallbackCall的几个参数：callbackExecutor，这个是在创建Retrofit时通过Platform得到的，并借此实例化了ExecutorCallAdapterFactory；call，就是前面的OkHttpCall了，里面拥有了配置的requestFactory，callFactory,，responseConverter和方法参数等

### 执行enqueue

拿到了Call对象，就是进行网络请求了

```java
		userCall.enqueue(new Callback<User>() {
            @Override
            public void onResponse(Call<User> call, Response<User> response) {

            }

            @Override
            public void onFailure(Call<User> call, Throwable t) {

            }
        });
```

调用enqueue方法，同时设置回调Callback

Call是一个接口，我们要找到具体的实现类，通过前面的分析，我们最后的Call是ExecutorCallAdapterFactory的内部类ExecutorCallbackCall，它持有了new出的OkHttpCall以及传递回调的callbackExecutor（拥有主线程的Looper）

接着就看看ExecutorCallbackCall吧

```java
static final class ExecutorCallbackCall<T> implements Call<T> {
    final Executor callbackExecutor;
    final Call<T> delegate;

    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
      	this.callbackExecutor = callbackExecutor;
      	this.delegate = delegate;
    }

    @Override public void enqueue(final Callback<T> callback) {
      	checkNotNull(callback, "callback == null");

      	delegate.enqueue(new Callback<T>() {
        	@Override public void onResponse(Call<T> call, final Response<T> response) {
          		callbackExecutor.execute(new Runnable() {
            		@Override public void run() {
              			if (delegate.isCanceled()) {
                			// Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                			callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
              			} else {
                			callback.onResponse(ExecutorCallbackCall.this, response);
                  		}
                	}
              	});
        	}

        	@Override public void onFailure(Call<T> call, final Throwable t) {
          		callbackExecutor.execute(new Runnable() {
            		@Override public void run() {
              			callback.onFailure(ExecutorCallbackCall.this, t);
            		}
          		});
        	}
      	});
    }

    @Override public boolean isExecuted() {
      	return delegate.isExecuted();
    }

    @Override public Response<T> execute() throws IOException {
 	    return delegate.execute();
    }

    @Override public void cancel() {
      	delegate.cancel();
    }

    @Override public boolean isCanceled() {
      	return delegate.isCanceled();
    }

    @SuppressWarnings("CloneDoesntCallSuperClone") // Performing deep clone.
    @Override public Call<T> clone() {
      	return new ExecutorCallbackCall<>(callbackExecutor, delegate.clone());
    }

    @Override public Request request() {
      	return delegate.request();
    }
}
```

 重点关注我们的enqueue方法，又调用了`delegate.enqueue()`方法，同时通过匿名实现方式，创建了Callback的回调，再回调到我们传进来的Callback中；那这个delegate又是什么，就是一个Call对象？

别忘了ExecutorCallbackCall怎么实例化的，前面提到，这个delegate其实就是我们的OhHttpCall

这就好理解了，通过OkHttpCall去进行网络请求

### OkHttpCall

看看OkHttpCall中的enqueue方法

```java
@Override public void enqueue(final Callback<T> callback) {
    checkNotNull(callback, "callback == null");

    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
        if (executed) throw new IllegalStateException("Already executed.");
      		executed = true;
		//拿到OkHttp3.Call
      	call = rawCall;
      	failure = creationFailure;
      	if (call == null && failure == null) {
            try {
                //第一次需要创建call
                call = rawCall = createRawCall();
            } catch (Throwable t) {
                throwIfFatal(t);
                failure = creationFailure = t;
            }
      	}
    }
	//失败回调
    if (failure != null) {
      	callback.onFailure(this, failure);
      	return;
    }
	//取消
    if (canceled) {
      	call.cancel();
    }
	//通过OkHttp3.Call执行
    call.enqueue(new okhttp3.Callback() {
      	@Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
        	Response<T> response;
        	try {
                //解析返回结果
          		response = parseResponse(rawResponse);
        	} catch (Throwable e) {
          		throwIfFatal(e);
          		callFailure(e);
          	return;
        	}

        	try {
          		callback.onResponse(OkHttpCall.this, response);
        	} catch (Throwable t) {
          		t.printStackTrace();
        	}
      	}

      	@Override public void onFailure(okhttp3.Call call, IOException e) {
        	callFailure(e);
      	}

      	private void callFailure(Throwable e) {
        	try {
          		callback.onFailure(OkHttpCall.this, e);
        	} catch (Throwable t) {
          		t.printStackTrace();
        	}
      	}
    });
}
```

#### 创建OkHttp3.Call

在这个方法中，首先是通过synchronized代码块来获取到OkHttp3.Call，如果为null，就通过`createRawCall()`来创建

```java
private okhttp3.Call createRawCall() throws IOException {
    okhttp3.Call call = callFactory.newCall(requestFactory.create(args));
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
}
```

callFactory是在OkHttpCall实例化的时候，Retrofit配置的时候设置的OkClient，通过newCall方法创建；requestFactory.create(args)得到了okhttp3.Request对象；requestFactory又是我们在生成HttpServiceMethod（具体应该在ServiceMethod中）时，创建出来的，args就是我们Method中的方法参数了（动态代理对象调用方法的参数，invoke传入）

```java
okhttp3.Request create(Object[] args) throws IOException {
    @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

    int argumentCount = args.length;
    if (argumentCount != handlers.length) {
      	throw new IllegalArgumentException("Argument count (" + argumentCount
          + ") doesn't match expected count (" + handlers.length + ")");
    }
	//创建了RequestBuilder
    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl,
        headers, contentType, hasBody, isFormEncoded, isMultipart);

    List<Object> argumentList = new ArrayList<>(argumentCount);
    for (int p = 0; p < argumentCount; p++) {
      	argumentList.add(args[p]);
      	handlers[p].apply(requestBuilder, args[p]);
    }
	//通过RequestBuilder创建Request
    return requestBuilder.get()
        .tag(Invocation.class, new Invocation(method, argumentList))
        .build();
}
```

接着回到OkHttpCall中的enqueue方法，看第二步解析结果

#### 解析返回结果

```java
call.enqueue(new okhttp3.Callback() {
    @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
		Response<T> response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          throwIfFatal(e);
          callFailure(e);
          return;
        }

        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
	}

    @Override public void onFailure(okhttp3.Call call, IOException e) {
        callFailure(e);
    }

    private void callFailure(Throwable e) {
        try {
          	callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          	t.printStackTrace();
		}
    }
});
```

通过OkHttp3的Call执行真正的网络请求，最后结果会通过okhttp3.Callback回调，接着调用的是`response = parseResponse(rawResponse);`方法

```java
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();

    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();
	//获取code码，判定错误，抛出
    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
      	try {
        	// Buffer the entire body to avoid future I/O.
        	ResponseBody bufferedBody = Utils.buffer(rawBody);
        	return Response.error(bufferedBody, rawResponse);
      	} finally {
        	rawBody.close();
      	}
    }
	//204和205关闭流，但是访问是成功的
    if (code == 204 || code == 205) {
      	rawBody.close();
      	return Response.success(null, rawResponse);
    }
	//包装一下Response
    ExceptionCatchingResponseBody catchingBody = new ExceptionCatchingResponseBody(rawBody);
    try {
        //通过我们在HttpServiceMethod中设置的responseConverter再进行解析，进行数据转换
      	T body = responseConverter.convert(catchingBody);
      	return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      	catchingBody.throwIfCaught();
      	throw e;
    }
}
```

在解析结果的时候，会先进行一些状态码的判断，然后通过包装一层ResponseBody，再交给我们HttpServiceMethod中的responseConverter解析成对应的数据类型，然后返回，通过前面的callback回调到

###  2.5.0和之前版本的区别

- 抽离了ServiceMethod，使用了具体实现HttpServiceMethod，减轻了ServiceMethod的负重，更多的放到了HttpServiceMethod中，以及通过层层参数的传递，进行了解耦
- 不管是CallAdapter还是ResponseConverter都是再创建HttpServiceMethod时都从Retrofit中找出，存放到自己HttpServiceMethod中

## 总结

- Retrofit的创建
- 动态代理得到接口对象
- 执行enqueue
- 真正通过OkHttpCall进行网络请求
- 数据解析和转换

## 特别鸣谢

- [Retrofit2源码解析——网络调用流程(上)](<https://juejin.im/post/5b97d08df265da0ac138fd5f>)
- [Retrofit2源码解析——网络调用流程(下)](<https://juejin.im/post/5b992acfe51d451a3f4bde83>)
- [Retrofit2 完全解析 探索与okhttp之间的关系](<https://blog.csdn.net/lmj623565791/article/details/51304204>)
- [Retrofit2 源码解析](<http://www.qinglinyi.com/posts/retrofit/>)