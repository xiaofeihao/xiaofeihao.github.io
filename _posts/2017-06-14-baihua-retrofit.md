---
layout: post
title: "超级白话Retrofit2源码"
date: 2017-06-14 20:30:00 +0800
catalog: true
image: /images/head.png
tags:
    - android
    - 超级白话系列
category: tech-blog
---

## **关于retrofit的简单使用**

```Java

Retrofit retrofit = new Retrofit.Builder()
                                .baseUrl("")
                                .build();
MyApi mApi = retrofit.create(MyApi.class);
mApi.getMethod();

```

## **源码分析**

首先创建Retrofit对象，使用了建造者（Build）模式。直接看代码里面干了什么------

### **代码1：**

```Java
/**
Retrofit.class
*/
public Retrofit build() {
    //首先检查baseUrl是不是null，如果是，直接抛出异常。所以这个必须有
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }
    //这个是真正用于请求的client，retrofit2是使用OkHttpClient(它实现了okhttp3.Call.Factory这个接口。
    //首先是将this.callFactory赋值，所以先看下这个是什么时候设置的。看--->代码2
    //如果不赋值，会new一个OkHttpClient.
    //所以自己赋值有什么好处呢？这就看对OkHttpClient的要求的，因为OkHttpClient也是可以自定义一些参数的，比如拦截器什么的。
      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }
    //这是执行器，用于执行回调的任务，可以控制线程。this.callbackExecutor赋值与callFactory赋值类似
    //如果没有赋值，会使用默认的，所以需要看下默认的是怎么生成的。用到了一个platform对象。那就先看platform，转代码3；
      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      // 这个Adapter很容易想起ListView的Adapter。这里也是使用了适配器模式，这个的作用也是用来指定执行任务的。具体的慢慢来看。
      //首先先把手动添加的this.adapterFactories全部加进来，然后又添加了一个默认的。生成默认的CallAdapter.Factory还是用了刚刚生成的callbackExecutor做了参数，那么看看怎么搞得。转代码4；
      //看完代码4，继续看这里，这里用了List来保存所有的CallAdapter.Factory，然后通过适配器模式，找到最终合适的那个，用来进行执行任务。
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

      // Make a defensive copy of the converters.
      //至于转换器，官网wiki给的有很多，比如：Gson,Jackson,Moshi,Protobuf,Wire,Simple XML等。具体去wiki看{@link http://square.github.io/retrofit/}
      List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);
    //终于完事，要创建Retrofit对象了。所以最后看下这里干了什么事。
    //确定了baseUrl，制定了client，也就是callFactory，如果没有指定，就new一个OkHttpClient。指定了转换器，可以将request请求或者response的结果进行转换。
      return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
    }
```

*看到这里是把Builder看完了，接下来看Create。转代码6；*

### **代码2：**

```Java
/**
Retrofit.class
*/
//这里是给this.callFactory赋值，当然可以不赋值，不赋值的话回看代码1
public Builder client(OkHttpClient client) {
    return callFactory(checkNotNull(client, "client == null"));
}

public Builder callFactory(okhttp3.Call.Factory factory) {
    this.callFactory = checkNotNull(factory, "factory == null");
    return this;
}
```

### **代码3：**

```Java
/**
Platform.class
*/
//platform使用了静态单例模式。get()方法是在Retrofit.Builder()中调用的。
//根据不同的系统生成了不同的平台(platform)，Android自然就是Android了。
//Android是Platform的一个静态内部类，转代码4；
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
    try {
      Class.forName("org.robovm.apple.foundation.NSObject");
      return new IOS();
    } catch (ClassNotFoundException ignored) {
    }
    return new Platform();
}
```
### **代码4：**

```Java
/**
Platform.class
*/
static class Android extends Platform {
    //找到了这个方法，default是new了一个MainThreadExecutor;
    //同样MainThreadExecutor是Android的一个静态内部类，继续看
    @Override public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }

    //直接new了一个ExecutorCallAdapterFactory。看下部分的ExecutorCallAdapterFactory代码，转代码5；
    @Override CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }
    //它实现了Executor接口，用于执行任务。里面的Handler通过getMainLooper生成，自然是将r运行在主线程。
    //所以看来，默认的Executor就是将任务在主线程执行。
    //回看代码1；
    static class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());

      @Override public void execute(Runnable r) {
        handler.post(r);
      }
    }
  }
```
### **代码5：**

```Java
/**
ExecutorCallAdapterFactory.class
*/
//总体来讲这个类的对象反正是new出来了，并且callbackExecutor也有了，就是刚刚创建的在主线程执行任务的执行器。回看代码1
final class ExecutorCallAdapterFactory extends CallAdapter.Factory {
  final Executor callbackExecutor;

  ExecutorCallAdapterFactory(Executor callbackExecutor) {
    this.callbackExecutor = callbackExecutor;
  }

//这是重写的CallAdapter.Factory里面的方法，具体实现与作用慢慢看
  @Override
  public CallAdapter<Call<?>> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    //.....具体实现转代码12，现在不用看
  }
//具体实现与作用慢慢看
  static final class ExecutorCallbackCall<T> implements Call<T> {
  }
}
```
****
*这里开始分析Retrofit对象的create*

### **代码6：**

```Java
/**
Retrofit.class
*/
//扫眼一看，是动态代理。一步一步看
public <T> T create(final Class<T> service) {
   //首先判断传进来的service是不是一个接口，不是接口直接抛出异常。可以自己点进去看
    Utils.validateServiceInterface(service);
    //其实刚刚使用builder的时候传进来一个参数，就是validateEagerly，所以看下eagerlyValidateMethods这个方法，是直接对service中的方法进行load，并缓存啥的。而动态代理中这些都是用反射做的，如果上来就直接load也不管用不用，对性能难免影响。所以这个还是暂时不要管。
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    //这才是重点！
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          //详细代码转代码7
        });
 }
```
### **代码7：**

```Java
/**
Retrofit.class
*/
//这是InvocationHandler里面的详细代码：
//首先还是先获取了一下是什么平台
private final Platform platform = Platform.get();
//invoke方法才是代理类要做的事情，在这里调用委托类的方法。至于动态代理就去google了。
@Override public Object invoke(Object proxy, Method method, Object... args) throws Throwable {
    // 如果是Object类的方法，那就直接默认调用就行了。代理类认为委托类不处理
    if (method.getDeclaringClass() == Object.class) {
        return method.invoke(this, args);
    }
    //如果是平台里默认的方法，代理类也是不处理，直接交给平台
    if (platform.isDefaultMethod(method)) {
        return platform.invokeDefaultMethod(method, service, proxy, args);
    }
    //终于是代理类需要处理的事情了。
    //代理的好处不就是在处理委托类的事情的时候可以自己先处理一下然后再去执行委托类的事务嘛
    //首先看第一步：将委托类的方法method进行了封装，封装成了ServiceMethod。这是很重要的！所以需要看一下如何封装的，也就是loadServiceMethod方法。转代码8；
    ServiceMethod serviceMethod = loadServiceMethod(method);
    //刚刚各种层次深入，终于看完了ServiceMethod这个对象。然后又创建了一个OkHttpCall对象。进去看下。转代码14
    OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
    //上面也终于看完到底什么情况，OkHttpCall就是对okhttp3.Call的一个封装，所以我们看看最终怎么执行的。看下callAdapter.adapt()的详细实现。这里就直接看retrofit包中默认的实现。转代码12
    //invoke方法的返回值时调用的委托类对象的对应方法的返回值
    return serviceMethod.callAdapter.adapt(okHttpCall);
}
//看完了，看看代码15之后的小总结吧
```

### **代码8：**

```Java
/**
Retrofit.class
*/
//这里用了简单的享元模式。毕竟使用反射还是耗性能的。serviceMethodCache是一个Map。
//ServiceMethod还是用的build。继续进入，转代码9；
ServiceMethod loadServiceMethod(Method method) {
    ServiceMethod result;
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```

### **代码9：**

```Java
/**
ServiceMethod.class
*/
public ServiceMethod build() {
    //首先创建了callAdapter 。等等，这个好熟悉。刚刚保存了一个List来着。快点看下这个createCallAdapter方法。转代码10；
      callAdapter = createCallAdapter();
      //...判断了一下responseType....代码省略
      //转换器的获取跟callAdpater差不多，可以自己看下实现源码。比较常用的是Gson，当然需要手动导入包，也是不在Retrofit包内。可以在Retrofit.Builder中add进去。也是添加了一个List，然后挨个找合适的。
      responseConverter = createResponseConverter();
      //历经千山万水终于到了真正解析注解的时刻。
      //具体代码还是去源码中看吧，挺长的，挺像的。反正就是把注解都解析了。赋值给了ServiceMethod.Builder中的字段。
      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }
    //...容错代码省略
    //等把所有必须的字段都赋值了，就可以创建ServiceMethod对象了，所以可以看出，这个对象包含了请求的所有信息，包括请求方法，请求参数，路径，请求头等等。看完这个可以继续看代码7了。
      return new ServiceMethod<>(this);
    }
```
### **代码10：**

```Java
/**
ServiceMethod.class
*/
private CallAdapter<?> createCallAdapter() {
    //首先获取method得返回值类型。
      Type returnType = method.getGenericReturnType();
      //然后判断这个返回类型能不能处理，如果不能处理直接抛出异常，至于怎么判断我也不知道，可以点进去看看。
      if (Utils.hasUnresolvableType(returnType)) {
        throw methodError(
            "Method return type must not include a type variable or wildcard: %s", returnType);
      }
      //然后返回类型不能是void的。retrofit1是通过callback，所以返回值是void。这是retrofit2,已经不允许void了。
      if (returnType == void.class) {
        throw methodError("Service methods cannot return void.");
      }
      //然后获取了method的注解。难道终于要解析注解了？不要着急，还没有。
      Annotation[] annotations = method.getAnnotations();
      try {
      //return的是调用的retrofit对象里面的方法？好吧，点进去看看。转代码11
        return retrofit.callAdapter(returnType, annotations);
      } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(e, "Unable to create call adapter for %s", returnType);
      }
    }
```
### **代码11：**

```Java
/**
Retrofit.class
*/
public CallAdapter<?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
}
public CallAdapter<?> nextCallAdapter(CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    //...判空代码省略
    //哇，终于看到了很久之前的那个List的用武之地
    int start = adapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
    //这里，可以看下里面item的get方法，也就是适配器模式。转代码12
      CallAdapter<?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      //看了代码12的策略，如果不合适get就返回Null。所以当不是null的时候就找到合适的了，返回这个adapter。
      //Adapter终于创建出来了，返回代码9
      if (adapter != null) {
        return adapter;
      }
    }
    //...后续处理代码省略
  }
```
### **代码12：**

```Java
/**
ExecutorCallAdapterFactory.class
*/
//适配在这里
//这是CallAdapter.Factory默认实现类，这个可以处理返回类型是Call.class的方法。所以判断返回类型是不是Call,如果不是，返回null。
//当然还有其他的实现类，比如非常流行的Rxjava中的。可以简单看一下。转代码13.
if (getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    //终于根据返回类型找到了合适的Adapter。所以这个Adapter中有个adapt（）方法，记住了。一会再看。先回到代码11
    return new CallAdapter<Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }
    //这里就是adapt的具体实现。是new了一个ExecutorCallbackCall.看下具体实现。转代码15
      @Override public <R> Call<R> adapt(Call<R> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
```
### **代码13：**

```Java
/**
注意，这个不是在Retrofit的包里，这是在adapter-rxjava的包里的类。可以对比Retrofit中的默认的适配器的实现。
*/
public CallAdapter<?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    Class<?> rawType = getRawType(returnType);
    String canonicalName = rawType.getCanonicalName();
    boolean isSingle = "rx.Single".equals(canonicalName);
    boolean isCompletable = "rx.Completable".equals(canonicalName);
    //这里判断，返回类型是不是rxjava能够处理的Observable等。看完回到代码12
    if (rawType != Observable.class && !isSingle && !isCompletable) {
      return null;
    }
}
```
### **代码14：**

```Java
/**
OkHttpCall.class
*/
//这里省略了很多代码
final class OkHttpCall<T> implements Call<T> {
  private final ServiceMethod<T> serviceMethod;
    //我擦，我看到了什么？？里面封装了一个okhttp3.Call。
    //原来这个类就是对okhttp3.Call的一个封装，所有的调用，其实都是okhttp3.Call的执行
  // All guarded by this.
  private okhttp3.Call rawCall;
  OkHttpCall(ServiceMethod<T> serviceMethod, Object[] args) {
    this.serviceMethod = serviceMethod;
    this.args = args;
  }
  @Override public OkHttpCall<T> clone() {}
  @Override public synchronized Request request() {}
  @Override public void enqueue(final Callback<T> callback) {}
  @Override public synchronized boolean isExecuted() {}
  @Override public Response<T> execute() throws IOException {}
  //原来serviceMethod的作用在这里，使用serviceMethod创建okhttp3的Request。然后创建Call。这就是okHttp3的知识点了。先不管。反正终于找到使用的地方了。回去看代码7
  private okhttp3.Call createRawCall() throws IOException {
    Request request = serviceMethod.toRequest(args);
    okhttp3.Call call = serviceMethod.callFactory.newCall(request);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }
  
  Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {}
  public void cancel() {}
}
```
### **代码15：**

```Java
/**
ExecutorCallAdapterFactory.class
*/
//这个类实现了Call接口。
//因为如果只用Retrofit2的话，我们声明的请求方法都是返回Call<T>。然后用这个的实例去调用execute(同步访问）或者enqueue(异步访问)。刚刚也是最终返回给retrofit的create的invoke。也就说的通了。
static final class ExecutorCallbackCall<T> implements Call<T> {
    final Executor callbackExecutor;
    final Call<T> delegate;

    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
    //这个callbackExecutor就是一开始传进来的MainCallbackExecutor啊。
      this.callbackExecutor = callbackExecutor;
      //这个delegate就是之前传过来的OkHttpCall啊。它封装了okhttp3.Call。一切都通了。
      this.delegate = delegate;
    }

    @Override public void enqueue(final Callback<T> callback) {
      if (callback == null) throw new NullPointerException("callback == null");
        //所以这里调用enqueue其实是OkHttpCall在调用，而OkHttpCall封装了okhttp3.Call，所以其实是okhttp3.Call在调用。那么通过回调，将返回结果执行在callbackExecutor的线程里，也就是主线程里。终于理顺了。回到代码7
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
}
```
## **总结**
使用动态代理，生成了MyApi的实例，当实例去访问getMethod这样的网络请求方法的时候，会调用代理的invoke，而invoke的返回值就是getMethod的返回值。在调用委托类也就是MyApi的方法时，首先对这个方法进行了一次封装，封装到了ServiceMethod这个对象中，这个对象里面封装了该请求需要的所有信息，包括请求地址，请求头，参数，方法类型等等等等。然后封装了OkHttpCall，每个serviceMethod对应一个call。最后返回委托类的返回值。其实委托类调用getMethod这个方法，并没有访问网络，真正访问网络还得需要Call的execute或者enqueue方法。它只是生成了Call这个实例。至于跟Rxjava结合起来，只是CallAdapter的不同。这个是通过适配器来找的。而条件就是能否处理方法的返回值类型(getType方法)。总体来讲就是这样的。可以看到，什么callAdapter啊，converter啊，都像是Retrofit的插件。可以自由选择，非常之强大。
