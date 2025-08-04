# Retrofit源码剖析

   Retrofit使用步骤

1. 添加Retrofit库的依赖，添加网络权限
2. 创建 接受服务器返回数据的类
3. 创建用于描述网络请求的接口
4. 创建Retrofit实例
5. 创建网络请求接口实例
6. 发送网络请求（同步、异步）
7. 处理服务器返回的数据

Retrofit代理模式

代理模式：为其他对象提供一种代理， 用以控制对这个对象的访问

+ 静态代理

+ 动态代理：代理类在程序运行时创建的代理方式

  无侵入

  通俗：增强方法

网络通信八步

1. 创建Retrofit实例
2. 定义一个网络请求接口并为接口中的方法添加注解
3. 通过动态代理生成网络请求对象
4. 通过网络请求适配器将网络请求对象进行平台适配
5. 通过网络请求执行器发送网络请求
6. 通过数据转换器解析数据
7. 通过回调执行器切换线程
8. 用户在主线程处理返回结果

Retrofit成员列表

```java
private final Map<Method, ServiceMethod> serviceMethodCache = new LinkedHashMap<>();
private final okhttp3.Call.Factory callFactory;
private final HttpUrl baseUrl;
private final List<Converter.Factory> converterFactories;
private final List<CallAdapter.Factory> adapterFactories;
private final Executor callbackExecutor;
private final boolean validateEagerly;
```

Bulilder方法

```java
// 默认构造
public Builder() {
  this(Platform.get());
}

// 通过反射校验平台
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
  
  // 以Android为例
    static class Android extends Platform {
    @Override public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }

    @Override CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }

	// 通过mainHandler回调给主线程
    static class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());

      @Override public void execute(Runnable r) {
        handler.post(r);
      }
    }
  }

// 传参构造，converterFactories传入默认BuiltInConverters
    Builder(Platform platform) {
      this.platform = platform;
      // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
      converterFactories.add(new BuiltInConverters());
    }
// Retrofit 构造
public Retrofit build() {
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }

      okhttp3.Call.Factory callFactory = this.callFactory;
    // 内部请求通过OkHttpClient
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

    // 获取默认回调Executor 
      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      // Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

      return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
    }
  }
```

 RxJavaCallAdapterFactory 内部构造与工作原理

callAdapter

主要将负责Call（Retrofit对OkHttpCall的封装）转换为Java对象， 具体转换流程

实现Factory -> 注册CallAdapter -> Factory.get -> adapt 方法

创建网络接口实例

```
public <T> T create(final Class<T> service) {
    this.validateServiceInterface(service);
    return Proxy.newProxyInstance(service.getClassLoader(), new Class[]{service}, new InvocationHandler() {
        private final Object[] emptyArgs = new Object[0];

        @Nullable
        public Object invoke(Object proxy, Method method, @Nullable Object[] args) throws Throwable {
            if (method.getDeclaringClass() == Object.class) {
                return method.invoke(this, args);
            } else {
                args = args != null ? args : this.emptyArgs;
                Reflection reflection = Platform.reflection;
                return reflection.isDefaultMethod(method) ? reflection.invokeDefaultMethod(method, service, proxy, args) : Retrofit.this.loadServiceMethod(service, method).invoke(proxy, args);
            }
        }
    });
}
```

通过动态代理返回接口实例

 Retrofit请求方式

+ 同步：OkhttpCall.execute
+ 异步：OkhttpCall.enqueue

Retrofit设计模式：

构建者模式.

工厂模式

外观模式

策略模式

适配器模式

动态代理模式

观察者模式