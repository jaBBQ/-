# OkHttp 笔记

OkHttp同步方法总结：

1. 创建OkHttpClient和Request对象
2. 将Request分装成Call对象
3. 调用Call的execute发送同步请求

同步注意：发送请求后，就会进入阻塞状态，直到收到响应



OkHttp异步方法总结：

1. 创建OkHttpClient和Request对象
2. 将Request分装成Call对象
3. 调用Call的enqueue方法进行异步请求



异步和同步：

1. 发起请求的方法调用
2. 阻塞线程与否



enqueue方法总结：

1. 判断当前call
2. 封装成了一个AsyncCall对象
3. client.dispatcher().enqueue() 



Q1 OkHttp如何实现同步异步请求？

A：发送的同步 / 异步请求都会在dispatcher 中管理其状态



Q2 到底什么是dispatcher

A：dispatcher的作用未维护请求的状态， 并维护一个线程池， 用于执行请求 



Q3 异步请求为什么需要两个队列

A：生产者 - 消费者模型

1. dispatcher 生产者
2. ExecutorService 消费者池
3. Deque <readyAsyncCalls>  缓存
4. Deque<runningAsyncCalls> 正在运行的任务



OkHttp的任务调度

1. Call执行完肯定需要在runningasyncCalls 队列中移除这个线程

Q：那么readyAsyncCalls队列中的线程在上面时候才会被执行呢？

A：dispatcher的finished中， 调用promoteCalls， 从ready队列取出任务放入running队列中



OkHttp拦截器

官网概念：拦截器是OkHttp中提供的一种强大机制， 它可以实现网络监听、请求以及响应重写、请求失败重试等功能 



OkHttp拦截器 - 总结1

1. 创建一系列拦截器， 并将其放入一个拦截器list中
2. 创建一个拦截器链RealInterceptorChain， 并执行拦截器链的proceed方法



OkHttp拦截器 - 总结2

1. 在发送请求前对request进行处理
2. 调用下一个拦截器， 获取response
3. 对response进行处理， 返回给上一个拦截器



  RetryandFollowUpInterceptor

1. 创建StreamAllocation 对象
2. 调用RetryandFollowUpInterceptor.proceed() 进行网络请求
3. 根据异常结果或者响应结果判断是否要进行重新请求
4. 调用下一个拦截器， 对response 进行处理， 返回给上一个拦截器



 BridgeInterceptor

1. 是负责将用户创建的一个Request请求转化为能够进行网络访问的请求
2. 将这个符合网络请求的Request进行网络请求
3. 将网络请求回来的响应Response转化为用户可用的Response



Connectlnterceptor

1. ConnectInterceptor 获取Interceptor传过来的StreamAllocation, streamallocation.newStream()
2. 将刚才创建的用于网络IO的RealConnection 对象， 以及对于与服务器交互最为关键的Httpcodec等对象传递给后面的拦截器
2. 弄一个Realconnection对象
2. 选择不同的链接方式
2. CallServerInterceptor



connectionPool 总结

1. 产生一个StreamAllocation 对象

2. StreamAllocation 对象的弱引用添加到RealConnection对象的allocation集合
3. 从链接池中获取



总结

1. OkHttp使用GC回收算法
2. StreamAllocation 的数量会渐渐变为0
3. 被线程池检测到并回收， 这样就可以保持多个健康的keep-alive链接



OkHttp中一次网络请求的大致过程

1. Call对象对请求的封装
2. dispatcher对请求的分发
3. getResponseWithInterceptors()方法





# OkHttp 入门探索（一）



>初学Android， 想要了解一些Android的开源库， 恰好最近用到OkHttp网络框架， 借此机会也深入学习一下，写这边文章作为笔记，希望帮助自己， 也能帮助其他新手开发进行入门，如有不对， 敬请斧正

PS：本文使用OkHttp 4.9.3， 使用全Kotlin开发

## OkHttp的使用

首先看一段请求代码

```kotlin
//创建OkHttpClient
val client = OkHttpClient.Builder().build();

//创建请求
val request = Request.Builder()
           .url("https://wanandroid.com/wxarticle/list/408/1/json")
           .build()

//同步任务开启新线程执行
Thread {
    //发起网络请求
    val response = client.newCall(request).execute()
    if (!response.isSuccessful) throw IOException("Unexpected code $response")
    Log.d("okhttp_test", "response:  ${response.body?.string()}")
}.start()
```

这段代码是OkHttp的同步请求，会阻塞当前线程。首先创建了一个OkHttpClient对象， 然后构造了request， 使用OkHttpClient的newCall创建Call对象， 并调用execute执行；

Call其实是一个接口，本身实现是RealCall

使用execute表示发起同步请求， 相对应还有enqueue表示发起异步请求

然后查看execute在RealCall的实现：

```kotlin
  override fun execute(): Response {
    check(executed.compareAndSet(false, true)) { "Already Executed" }

    timeout.enter()
    callStart()
    try {
      client.dispatcher.executed(this)
      return getResponseWithInterceptorChain()
    } finally {
      client.dispatcher.finished(this)
    }
  }
```

+ 首先check该请求是否被执行， 如果执行抛出"Already Executed"错误
+ 然后调用timeout.enter()， timeout是一个AsyncTimeout对象，内部重写了timedOut方法，当发送超时时会执行cancel操作（TODO：timeout时间什么时候设定， 默认为多少）
+ 然后调用dispatcher执行executed， dispatcher本质是一个网络请求分发器，在下文会进行详细介绍
+ 最后返回getResponseWithInterceptorChain结果，总的看来，真正执行请求的地方就是该方法，其实这也是OkHttp的核心，通过责任链模式实现请求封装，进行各层解耦， 后文我们会详细介绍

异步请求和同步相似， 不同是通过enqueue执行，最后通过CallBack返回

## dispatcher

dispatcher的作用是维护请求的状态， 并维护一个线程池， 用于执行请求 

内部含有三个队列：

```kotlin
/** Ready async calls in the order they'll be run. */
private val readyAsyncCalls = ArrayDeque<AsyncCall>()

/** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
private val runningAsyncCalls = ArrayDeque<AsyncCall>()

/** Running synchronous calls. Includes canceled calls that haven't finished yet. */
private val runningSyncCalls = ArrayDeque<RealCall>()
```

上述三个队列维护请求执行



PS：队列为什么使用ArrayDeque而不是LinkedList？

 A：因为readyAsyncCalls会向runningAsyncCalls进行转换，LinkedList由于链表数据结构使元素分布在内存各个位置， 遍历时开销相比ArrayDeque较大， 另外在垃圾回收时， 使用数组结构的效率也优于链表。



然后看异步请求enqueue：

```kotlin
internal fun enqueue(call: AsyncCall) {
  synchronized(this) {
    readyAsyncCalls.add(call)

    // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
    // the same host.
    if (!call.call.forWebSocket) {
      val existingCall = findExistingCallWithHost(call.host)
      if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
    }
  }
  promoteAndExecute()
}
```

上述代码，进行以下几个操作

+ 首先将call存储readyAsyncCalls队列中
+ 然后判断当前请求是否为WebSocket（WebSocket 是一种不同于常规 HTTP 的协议，用于双向通信。如果请求是 WebSocket 请求，它通常不需要与 HTTP 请求使用相同的连接复用逻辑，因为 WebSocket 通常会保持长时间的连接。）， 如果不是调用findExistingCallWithHost在running和ready队列查找与当前请求host相同的请求，如果找到，让相同host请求的callsPerHost共享一个对象（猜测后续判断对同一host最大请求限制时使用）
+ 然后调用promoteAndExecute， 这是分发器分发异步请求的关键， 主要负责将ready中符合条件的可执行请求存入running和executableCalls队列中

下面来看promoteAndExecute实现：

```kotlin
private fun promoteAndExecute(): Boolean {
  this.assertThreadDoesntHoldLock()

  val executableCalls = mutableListOf<AsyncCall>()
  val isRunning: Boolean
  synchronized(this) {
    val i = readyAsyncCalls.iterator()
    while (i.hasNext()) {
      val asyncCall = i.next()

      if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
      if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue // Host max capacity.

      i.remove()
      asyncCall.callsPerHost.incrementAndGet()
      executableCalls.add(asyncCall)
      runningAsyncCalls.add(asyncCall)
    }
    isRunning = runningCallsCount() > 0
  }

  for (i in 0 until executableCalls.size) {
    val asyncCall = executableCalls[i]
    asyncCall.executeOn(executorService)
  }

  return isRunning
}
```

 上述代码做了这几件事：

+ 首先遍历readyAsyncCalls队列；校验runningAsyncCalls的数目是否大于最大请求数；遍历当前请求的host是否大于该host最大请求数
+ 如果满足上述条件，从readyAsyncCalls移除，当前call的callsPerHost增加， 存入executableCalls和runningAsyncCalls中
+ 遍历executableCalls，调用executeOn执行
+ 返回是否有执行的任务

流程图参考：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4d3698dbe67421ba2a60ef8556c2349~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1563&h=714&s=92436&e=jpg&b=f4f4f4)

下面看下executeOn代码：

```kotlin
fun executeOn(executorService: ExecutorService) {
  client.dispatcher.assertThreadDoesntHoldLock()

  var success = false
  try {
    executorService.execute(this)
    success = true
  } catch (e: RejectedExecutionException) {
    val ioException = InterruptedIOException("executor rejected")
    ioException.initCause(e)
    noMoreExchanges(ioException)
    responseCallback.onFailure(this@RealCall, ioException)
  } finally {
    if (!success) {
      client.dispatcher.finished(this) // This call is no longer running!
    }
  }
}
```

代码流程很简单，就是通过executorService执行该AsyncCall， 该AsyncCall继承自Runnable

executorService就是我们内部的线程池， 看下创建的地方：

```kotlin
@get:JvmName("executorService") val executorService: ExecutorService
  get() {
    if (executorServiceOrNull == null) {
      executorServiceOrNull = ThreadPoolExecutor(0, Int.MAX_VALUE, 60, TimeUnit.SECONDS,
          SynchronousQueue(), threadFactory("$okHttpName Dispatcher", false))
    }
    return executorServiceOrNull!!
  }
```

为了提升请求效率，则不能让新任务被添加到等待队列，而是要新建线程执行新任务，因此将 `corePoolSize` 设置为0，并传入无界队列 SynchronousQueue 使得每次添加新任务到队列都失败，则会新建线程去执行新任务。

下面看下AsyncCall的run代码：

```kotlin
  override fun run() {
    threadName("OkHttp ${redactedUrl()}") {
      var signalledCallback = false
      timeout.enter()
      try {
        val response = getResponseWithInterceptorChain()
        signalledCallback = true
        responseCallback.onResponse(this@RealCall, response)
      } catch (e: IOException) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log("Callback failure for ${toLoggableString()}", Platform.INFO, e)
        } else {
          responseCallback.onFailure(this@RealCall, e)
        }
      } catch (t: Throwable) {
        cancel()
        if (!signalledCallback) {
          val canceledException = IOException("canceled due to $t")
          canceledException.addSuppressed(t)
          responseCallback.onFailure(this@RealCall, canceledException)
        }
        throw t
      } finally {
        client.dispatcher.finished(this)
      }
    }
  }
}
```

可以看到和同步一样，最终还是调用getResponseWithInterceptorChain获取到结果， 最后回调callback返回结果。

不论是同步还是异步，在finally方法总能看到：

```kotlin
client.dispatcher.finished(this)
```

看下内部实现：

```kotlin
internal fun finished(call: AsyncCall) {
  call.callsPerHost.decrementAndGet()
  finished(runningAsyncCalls, call)
}
```

可以看到首先将该Call的host计数减一，然后调用内部finished

```kotlin
private fun <T> finished(calls: Deque<T>, call: T) {
  val idleCallback: Runnable?
  synchronized(this) {
    if (!calls.remove(call)) throw AssertionError("Call wasn't in-flight!")
    idleCallback = this.idleCallback
  }

  val isRunning = promoteAndExecute()

  if (!isRunning && idleCallback != null) {
    idleCallback.run()
  }
}
```

+ 首先从runningAsyncCalls队列中移除（异步情况， 同步移除runningSyncCalls）
+ 然后执行promoteAndExecute，这个方法我们之前看过，就是将ready队列中符合条件的Call移入running中
+ 下面的idleCallback.run()，是Dispatcher在空闲时的回调， 如果当前队列没有等待执行的任务， 并且idleCallback不为空就执行idleCallback



## 总结

+ OkHttp包含两种请求方式， 分别为同步execute和异步enqueue
+ 不论同步还是异步，都是通过dispatcher进行调度
+  dispatcher维护三个任务队列， 实现方式都为双端队列， 分别为同步请求执行队列，异步请求等待队列，异步请求执行队列。
+ 当发送一个新的请求时，会将Call存入执行队列中，然后调用getResponseWithInterceptorChain得到结果；同步在当前线程， 异步抛入线程池执行
+ 在执行完成后，会从running队列中移除该Call

附上同步、异步请求时序图：

同步：

![同步](https://gitee.com/lin-chujie/blog-pics/raw/master/Android/OkHttp/%E6%8E%A2%E7%B4%A2OkHttp%E7%B3%BB%E5%88%97%20(%E4%B8%80)%20%E8%AF%B7%E6%B1%82%E7%9A%84%E5%8F%91%E8%B5%B7%E4%B8%8E%E5%93%8D%E5%BA%94/image-20211113011234986.png)

异步：

![异步](https://gitee.com/lin-chujie/blog-pics/raw/master/Android/OkHttp/%E6%8E%A2%E7%B4%A2OkHttp%E7%B3%BB%E5%88%97%20(%E4%B8%80)%20%E8%AF%B7%E6%B1%82%E7%9A%84%E5%8F%91%E8%B5%B7%E4%B8%8E%E5%93%8D%E5%BA%94/image-20211113011738025.png)



总的来说，一切请求最终都走到getResponseWithInterceptorChain， 该方法是OkHttp的核心方法，通过责任链模式，一层一层实现请求发送的工作，类比七层网络模型，后续我们会对该方法展开描述。



# OkHttp 入门探索（二）

> 前文我们提到了OkHttp的请求的分发流程，通过dispatcher进行同步和异步的处理，但发现最后的核心处理流程都走到getResponseWithInterceptorChain中，这也是OkHttp的核心方法，通过责任链一步步处理请求和响应，最终返回给上层，下面我们就展开看看这个方法

## getResponseWithInterceptorChain

首先看getResponseWithInterceptorChain代码



```kotlin
 internal fun getResponseWithInterceptorChain(): Response {
    // Build a full stack of interceptors.
    val interceptors = mutableListOf<Interceptor>()
    interceptors += client.interceptors
    interceptors += RetryAndFollowUpInterceptor(client)
    interceptors += BridgeInterceptor(client.cookieJar)
    interceptors += CacheInterceptor(client.cache)
    interceptors += ConnectInterceptor
    if (!forWebSocket) {
      interceptors += client.networkInterceptors
    }
    interceptors += CallServerInterceptor(forWebSocket)

    val chain = RealInterceptorChain(
        call = this,
        interceptors = interceptors,
        index = 0,
        exchange = null,
        request = originalRequest,
        connectTimeoutMillis = client.connectTimeoutMillis,
        readTimeoutMillis = client.readTimeoutMillis,
        writeTimeoutMillis = client.writeTimeoutMillis
    )

    var calledNoMoreExchanges = false
    try {
      val response = chain.proceed(originalRequest)
      if (isCanceled()) {
        response.closeQuietly()
        throw IOException("Canceled")
      }
      return response
    } catch (e: IOException) {
      calledNoMoreExchanges = true
      throw noMoreExchanges(e) as Throwable
    } finally {
      if (!calledNoMoreExchanges) {
        noMoreExchanges(null)
      }
    }
  }
```

代码有点长，我们首先逐步进行分析：

```kotlin
val interceptors = mutableListOf<Interceptor>()
interceptors += client.interceptors
interceptors += RetryAndFollowUpInterceptor(client)
interceptors += BridgeInterceptor(client.cookieJar)
interceptors += CacheInterceptor(client.cache)
interceptors += ConnectInterceptor
if (!forWebSocket) {
  interceptors += client.networkInterceptors
}
interceptors += CallServerInterceptor(forWebSocket)
```

这段代码，我们可以看出通过list来保存Interceptor，按照顺序分别为：

1. client.interceptors 用户自定义的Interceptor
2. RetryAndFollowUpInterceptor
3. BridgeInterceptor
4. CacheInterceptor
5. ConnectInterceptor
6. CallServerInterceptor

上述各个拦截器，我们后面会逐一分析

### **Application Interceptors vs. Network Interceptors**

| 特性                   | Application Interceptors | Network Interceptors      |
| ---------------------- | ------------------------ | ------------------------- |
| **添加方式**           | `addInterceptor()`       | `addNetworkInterceptor()` |
| **调用次数**           | 只调用一次，不受重试影响 | 对每次网络请求都会调用    |
| **是否操作缓存**       | 无法操作缓存             | 可访问缓存后的请求        |
| **是否操作重定向请求** | 不拦截内部的重定向请求   | 可拦截重定向请求          |
| **拦截位置**           | 在应用逻辑层处理         | 在网络 I/O 逻辑层处理     |

如果forWebSocket为真，也就是使用WebSocket通信，就不需要networkInterceptors了

在这里我们获得了一个list<Interceptor>, 保存了我们所需要的各样拦截器，那么后续如何使用呢，我们接着往下看

```kotlin
val chain = RealInterceptorChain(
    call = this,
    interceptors = interceptors,
    index = 0,
    exchange = null,
    request = originalRequest,
    connectTimeoutMillis = client.connectTimeoutMillis,
    readTimeoutMillis = client.readTimeoutMillis,
    writeTimeoutMillis = client.writeTimeoutMillis
)
```

可以看到，我们使用interceptors构造了一个RealInterceptorChain

具体参数解释：

1. **call**: 当前的网络调用对象，代表这个请求的上下文，包含有关请求的信息。
2. **interceptors**: 一组拦截器，这些拦截器将在请求处理过程中依次被调用。
3. **index**: 当前拦截器的索引，通常从 0 开始，表示正在处理的拦截器在拦截器列表中的位置。
4. **exchange**: 代表网络交换的对象，通常在建立连接后使用，WebSocket 时可能为空。
5. **request**: 原始请求对象，包含要发送的 HTTP 请求的详细信息，例如 URL、方法、头信息等。
6. **connectTimeoutMillis**: 连接超时时间（毫秒），指的是建立连接时的最大等待时间。
7. **readTimeoutMillis**: 读取超时时间（毫秒），指的是从服务器读取响应时的最大等待时间。
8. **writeTimeoutMillis**: 写入超时时间（毫秒），指的是向服务器发送请求时的最大等待时间。

```kotlin
var calledNoMoreExchanges = false
try {
  val response = chain.proceed(originalRequest)
  if (isCanceled()) {
    response.closeQuietly()
    throw IOException("Canceled")
  }
  return response
} catch (e: IOException) {
  calledNoMoreExchanges = true
  throw noMoreExchanges(e) as Throwable
} finally {
  if (!calledNoMoreExchanges) {
    noMoreExchanges(null)
  }
}
```

这段代码流程也相对简单：

+ 首先调用 chain.proceed(originalRequest)获取到response， 可以看出来具体的请求流程就在proceed里了
+ 得到返回结果后当前请求是否被取消，如果被取消就closeQuietly关闭响应流，然后抛出异常
+ 拦截异常后给calledNoMoreExchanges赋值为true，执行noMoreExchanges了
+ noMoreExchanges表示不再进行网络请求交换，帮助连接池复用

之后看下

```kotlin
  @Throws(IOException::class)
  override fun proceed(request: Request): Response {
    check(index < interceptors.size)

    calls++

    if (exchange != null) {
      check(exchange.finder.sameHostAndPort(request.url)) {
        "network interceptor ${interceptors[index - 1]} must retain the same host and port"
      }
      check(calls == 1) {
        "network interceptor ${interceptors[index - 1]} must call proceed() exactly once"
      }
    }

    // Call the next interceptor in the chain.
    val next = copy(index = index + 1, request = request)
    val interceptor = interceptors[index]

    @Suppress("USELESS_ELVIS")
    val response = interceptor.intercept(next) ?: throw NullPointerException(
        "interceptor $interceptor returned null")

    if (exchange != null) {
      check(index + 1 >= interceptors.size || next.calls == 1) {
        "network interceptor $interceptor must call proceed() exactly once"
      }
    }

    check(response.body != null) { "interceptor $interceptor returned a response with no body" }

    return response
  }
}
```

这段代码流程相对也比较清晰

+ 首先    calls++ ，calls表示调用proceed的次数，确保每个拦截器恰好调用一次proceed方法，防止多次调用或者漏掉用

+ 然后 val next = copy(index = index + 1, request = request)，拷贝一份index + 1 的拦截器列表

+ val interceptor = interceptors[index]

  val response = interceptor.intercept(next)

  首先获取当前的拦截器，执行intercept，并传入拷贝的拦截器列表

+ 校验响应体是否为空， 为空就抛出异常

看下intercept方法：

```kotlin
fun intercept(chain: Chain): Response
```

可以看出是一个接口，各个拦截器自己实现该方法

这里我们没有实现自定义拦截器，我们先看下RetryAndFollowUpInterceptor的实现

## RetryAndFollowUpInterceptor

```kotlin
@Throws(IOException::class)
override fun intercept(chain: Interceptor.Chain): Response {
  val realChain = chain as RealInterceptorChain
  var request = chain.request
  val call = realChain.call
  var followUpCount = 0
  var priorResponse: Response? = null
  var newExchangeFinder = true
  var recoveredFailures = listOf<IOException>()
  while (true) {
    call.enterNetworkInterceptorExchange(request, newExchangeFinder)

    var response: Response
    var closeActiveExchange = true
    try {
      if (call.isCanceled()) {
        throw IOException("Canceled")
      }

      try {
        response = realChain.proceed(request)
        newExchangeFinder = true
      } catch (e: RouteException) {
        // The attempt to connect via a route failed. The request will not have been sent.
        if (!recover(e.lastConnectException, call, request, requestSendStarted = false)) {
          throw e.firstConnectException.withSuppressed(recoveredFailures)
        } else {
          recoveredFailures += e.firstConnectException
        }
        newExchangeFinder = false
        continue
      } catch (e: IOException) {
        // An attempt to communicate with a server failed. The request may have been sent.
        if (!recover(e, call, request, requestSendStarted = e !is ConnectionShutdownException)) {
          throw e.withSuppressed(recoveredFailures)
        } else {
          recoveredFailures += e
        }
        newExchangeFinder = false
        continue
      }

      // Attach the prior response if it exists. Such responses never have a body.
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                .body(null)
                .build())
            .build()
      }

      val exchange = call.interceptorScopedExchange
      val followUp = followUpRequest(response, exchange)

      if (followUp == null) {
        if (exchange != null && exchange.isDuplex) {
          call.timeoutEarlyExit()
        }
        closeActiveExchange = false
        return response
      }

      val followUpBody = followUp.body
      if (followUpBody != null && followUpBody.isOneShot()) {
        closeActiveExchange = false
        return response
      }

      response.body?.closeQuietly()

      if (++followUpCount > MAX_FOLLOW_UPS) {
        throw ProtocolException("Too many follow-up requests: $followUpCount")
      }

      request = followUp
      priorResponse = response
    } finally {
      call.exitNetworkInterceptorExchange(closeActiveExchange)
    }
  }
}
```

上述代码有点长，我们逐步分析：

```kotlin
while (true) {
  call.enterNetworkInterceptorExchange(request, newExchangeFinder)
  ......
  
}
```

首先进入一个while循环，通过调用enterNetworkInterceptorExchange将请求与网络拦截器关联

```kotlin
if (call.isCanceled()) {
  throw IOException("Canceled")
}

try {
  response = realChain.proceed(request)
  newExchangeFinder = true
} catch (e: RouteException) {
  // The attempt to connect via a route failed. The request will not have been sent.
  if (!recover(e.lastConnectException, call, request, requestSendStarted = false)) {
    throw e.firstConnectException.withSuppressed(recoveredFailures)
  } else {
    recoveredFailures += e.firstConnectException
  }
  newExchangeFinder = false
  continue
} catch (e: IOException) {
  // An attempt to communicate with a server failed. The request may have been sent.
  if (!recover(e, call, request, requestSendStarted = e !is ConnectionShutdownException)) {
    throw e.withSuppressed(recoveredFailures)
  } else {
    recoveredFailures += e
  }
  newExchangeFinder = false
  continue
}
```

+ 首先校验call是否被取消， 如果被取消抛出异常

+ 然后继续向下发送请求， 最终返回response

  ```
  response = realChain.proceed(request)
  ```

+ 如果发生RouteException异常， 表示路由链接异常的场景，请求不会被发送，抛出首次链接异常

+ 发生IOException异常

上述两种异常都会调用recover决定是否继续，看下recover

```kotlin
private fun recover(
  e: IOException,
  call: RealCall,
  userRequest: Request,
  requestSendStarted: Boolean
): Boolean {
  // The application layer has forbidden retries.
  if (!client.retryOnConnectionFailure) return false

  // We can't send the request body again.
  if (requestSendStarted && requestIsOneShot(e, userRequest)) return false

  // This exception is fatal.
  if (!isRecoverable(e, requestSendStarted)) return false

  // No more routes to attempt.
  if (!call.retryAfterFailure()) return false

  // For failure recovery, use the same route selector with a new connection.
  return true
}
```

+ 首先校验调用方是否允许重试
+ 然后校验，如果请求被发送， 并且该请求是一个oneshot（只请求一次）则返回
+ 然后调用isRecoverable，这个方法主要是判断该请求是否可会的，后续我们详细看下
+ 查看是否还存在其他可重试的路由

看下isRecoverable方法：

```kotlin
private fun isRecoverable(e: IOException, requestSendStarted: Boolean): Boolean {
  // If there was a protocol problem, don't recover.
  if (e is ProtocolException) {
    return false
  }

  // If there was an interruption don't recover, but if there was a timeout connecting to a route
  // we should try the next route (if there is one).
  if (e is InterruptedIOException) {
    return e is SocketTimeoutException && !requestSendStarted
  }

  // Look for known client-side or negotiation errors that are unlikely to be fixed by trying
  // again with a different route.
  if (e is SSLHandshakeException) {
    // If the problem was a CertificateException from the X509TrustManager,
    // do not retry.
    if (e.cause is CertificateException) {
      return false
    }
  }
  if (e is SSLPeerUnverifiedException) {
    // e.g. a certificate pinning error.
    return false
  }
  // An example of one we might want to retry with a different route is a problem connecting to a
  // proxy and would manifest as a standard IOException. Unless it is one we know we should not
  // retry, we return true and try a new route.
  return true
}
```

上述还是对IOException进行了校验， 主要有以下几个特点

+ ProtocolException 协议异常
+ SocketTimeoutException Socket超时且请求还未发送
+ SSLHandshakeException、CertificateException：SSL握手异常或证书异常（证书格式不正确、证书链不完整、证书无效）
+ SSLPeerUnverifiedException 表示证书未能通过验证， 例如：证书不被信任、证书过期、域名不匹配

接着回到RetryAndFollowUpInterceptor接着看

```
if (priorResponse != null) {
  response = response.newBuilder()
      .priorResponse(priorResponse.newBuilder()
          .body(null)
          .build())
      .build()
}

val exchange = call.interceptorScopedExchange
val followUp = followUpRequest(response, exchange)

if (followUp == null) {
  if (exchange != null && exchange.isDuplex) {
    call.timeoutEarlyExit()
  }
  closeActiveExchange = false
  return response
}
```

+ priorResponse表示上次的response， 第一次请求失败时为空。
+ 校验如果上次priorResponse不为空， 根据上次reponse构造新的response对象；这里是通过上次请求的错误码重新构建请求，例如在重定向情况下返回302重定向错误码， 以及新的重定向URL，就需要我们重新构建response为新的，并且对响应链进行记录，以了解用户请求历程。body为空，因为priorResponse可能正文内容为空或是不相关的，赋值为null确保后续处理不会尝试读取一个不应该存在的内容
+ 然后根据响应（HTTP响应码）决定是否构建一个新的请求
+ 如果不需要， 则followUp为null， 调用请求流程， 返回原始响应对象

接着往下看

```kotlin
response.body?.closeQuietly()

if (++followUpCount > MAX_FOLLOW_UPS) {
  throw ProtocolException("Too many follow-up requests: $followUpCount")
}

request = followUp
priorResponse = response
```

+ 首先关闭请求body， 释放资源
+ 校验重试次数是否超过规定次数， 最大次数定为20
+ 接着将构造的followUp赋值给request， 更新旧response

最后

```kotlin
finally {
  call.exitNetworkInterceptorExchange(closeActiveExchange)
}
```

解除与网络拦截器的绑定

至此我们对整个RetryAndFollowUpInterceptor分析到这结束了

总结以下主要做了以下几件事

+ 循环重试请求， 最大请求次数为20

+ 请求失败通过异常Exception， 进行校验是否继续

+ 如果继续， 通过上一次的response生成新的request、response，然后进行下一次请求；

  这里构造response是要记录整个请求历程

+ 重新请求

后续我们会对其他几个拦截器进行详细介绍

# OkHttp 入门探索（三）

之前我们对RetryAndFollowUpInterceptor有了简单的了解， 本章我们将对BridgeInterceptor进行一个简单的剖析，那么还是先上intercept。

## BridgeInterceptor

```kotlin
override fun intercept(chain: Interceptor.Chain): Response {
  val userRequest = chain.request()
  val requestBuilder = userRequest.newBuilder()

  val body = userRequest.body
  if (body != null) {
    val contentType = body.contentType()
    if (contentType != null) {
      requestBuilder.header("Content-Type", contentType.toString())
    }

    val contentLength = body.contentLength()
    if (contentLength != -1L) {
      requestBuilder.header("Content-Length", contentLength.toString())
      requestBuilder.removeHeader("Transfer-Encoding")
    } else {
      requestBuilder.header("Transfer-Encoding", "chunked")
      requestBuilder.removeHeader("Content-Length")
    }
  }

  if (userRequest.header("Host") == null) {
    requestBuilder.header("Host", userRequest.url.toHostHeader())
  }

  if (userRequest.header("Connection") == null) {
    requestBuilder.header("Connection", "Keep-Alive")
  }

  // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
  // the transfer stream.
  var transparentGzip = false
  if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
    transparentGzip = true
    requestBuilder.header("Accept-Encoding", "gzip")
  }

  val cookies = cookieJar.loadForRequest(userRequest.url)
  if (cookies.isNotEmpty()) {
    requestBuilder.header("Cookie", cookieHeader(cookies))
  }

  if (userRequest.header("User-Agent") == null) {
    requestBuilder.header("User-Agent", userAgent)
  }

  val networkResponse = chain.proceed(requestBuilder.build())

  cookieJar.receiveHeaders(userRequest.url, networkResponse.headers)

  val responseBuilder = networkResponse.newBuilder()
      .request(userRequest)

  if (transparentGzip &&
      "gzip".equals(networkResponse.header("Content-Encoding"), ignoreCase = true) &&
      networkResponse.promisesBody()) {
    val responseBody = networkResponse.body
    if (responseBody != null) {
      val gzipSource = GzipSource(responseBody.source())
      val strippedHeaders = networkResponse.headers.newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build()
      responseBuilder.headers(strippedHeaders)
      val contentType = networkResponse.header("Content-Type")
      responseBuilder.body(RealResponseBody(contentType, -1L, gzipSource.buffer()))
    }
  }

  return responseBuilder.build()
}
```

整个方法流程清晰简单，主要做了以下几件事：

+ 首先给Request添加header的必要字段：如：Content-Type、Content-Length or Transfer-Encoding、Host、Connection，然后检查是否有Accept-Encoding，没有默认gzip
+ 添加cookie信息，从cookieJar（使用者自定义， 默认为空实现）中获取， 如果没有不填写
+ 填写User-Agent
+ 发送请求，获得networkResponse
+ 根据返回结果更新cookieJar
+ 根据Request构建一个response， 校验如果返回结果是gzip，进行解压并填充到新response的body中，去除解压相关字段
+ 返回新response

整体而言，BridgeInterceptor主要是为Request添加header信息，以及进行更新body解压缩操作的

我们接着往下看CacheInterceptor

## CacheInterceptor

照例还是线上源码

```kotlin
override fun intercept(chain: Interceptor.Chain): Response {
  val call = chain.call()
  val cacheCandidate = cache?.get(chain.request())

  val now = System.currentTimeMillis()

  val strategy = CacheStrategy.Factory(now, chain.request(), cacheCandidate).compute()
  val networkRequest = strategy.networkRequest
  val cacheResponse = strategy.cacheResponse

  cache?.trackResponse(strategy)
  val listener = (call as? RealCall)?.eventListener ?: EventListener.NONE

  if (cacheCandidate != null && cacheResponse == null) {
    // The cache candidate wasn't applicable. Close it.
    cacheCandidate.body?.closeQuietly()
  }

  // If we're forbidden from using the network and the cache is insufficient, fail.
  if (networkRequest == null && cacheResponse == null) {
    return Response.Builder()
        .request(chain.request())
        .protocol(Protocol.HTTP_1_1)
        .code(HTTP_GATEWAY_TIMEOUT)
        .message("Unsatisfiable Request (only-if-cached)")
        .body(EMPTY_RESPONSE)
        .sentRequestAtMillis(-1L)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build().also {
          listener.satisfactionFailure(call, it)
        }
  }

  // If we don't need the network, we're done.
  if (networkRequest == null) {
    return cacheResponse!!.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .build().also {
          listener.cacheHit(call, it)
        }
  }

  if (cacheResponse != null) {
    listener.cacheConditionalHit(call, cacheResponse)
  } else if (cache != null) {
    listener.cacheMiss(call)
  }

  var networkResponse: Response? = null
  try {
    networkResponse = chain.proceed(networkRequest)
  } finally {
    // If we're crashing on I/O or otherwise, don't leak the cache body.
    if (networkResponse == null && cacheCandidate != null) {
      cacheCandidate.body?.closeQuietly()
    }
  }

  // If we have a cache response too, then we're doing a conditional get.
  if (cacheResponse != null) {
    if (networkResponse?.code == HTTP_NOT_MODIFIED) {
      val response = cacheResponse.newBuilder()
          .headers(combine(cacheResponse.headers, networkResponse.headers))
          .sentRequestAtMillis(networkResponse.sentRequestAtMillis)
          .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis)
          .cacheResponse(stripBody(cacheResponse))
          .networkResponse(stripBody(networkResponse))
          .build()

      networkResponse.body!!.close()

      // Update the cache after combining headers but before stripping the
      // Content-Encoding header (as performed by initContentStream()).
      cache!!.trackConditionalCacheHit()
      cache.update(cacheResponse, response)
      return response.also {
        listener.cacheHit(call, it)
      }
    } else {
      cacheResponse.body?.closeQuietly()
    }
  }

  val response = networkResponse!!.newBuilder()
      .cacheResponse(stripBody(cacheResponse))
      .networkResponse(stripBody(networkResponse))
      .build()

  if (cache != null) {
    if (response.promisesBody() && CacheStrategy.isCacheable(response, networkRequest)) {
      // Offer this request to the cache.
      val cacheRequest = cache.put(response)
      return cacheWritingResponse(cacheRequest, response).also {
        if (cacheResponse != null) {
          // This will log a conditional cache miss only.
          listener.cacheMiss(call)
        }
      }
    }

    if (HttpMethod.invalidatesCache(networkRequest.method)) {
      try {
        cache.remove(networkRequest)
      } catch (_: IOException) {
        // The cache cannot be written.
      }
    }
  }

  return response
}
```

代码还是比较长的，我们分开来看

```kotlin
val call = chain.call()
val cacheCandidate = cache?.get(chain.request())

val now = System.currentTimeMillis()

val strategy = CacheStrategy.Factory(now, chain.request(), cacheCandidate).compute()
val networkRequest = strategy.networkRequest
val cacheResponse = strategy.cacheResponse

cache?.trackResponse(strategy)
val listener = (call as? RealCall)?.eventListener ?: EventListener.NONE

if (cacheCandidate != null && cacheResponse == null) {
  // The cache candidate wasn't applicable. Close it.
  cacheCandidate.body?.closeQuietly()
}

// If we're forbidden from using the network and the cache is insufficient, fail.
if (networkRequest == null && cacheResponse == null) {
  return Response.Builder()
      .request(chain.request())
      .protocol(Protocol.HTTP_1_1)
      .code(HTTP_GATEWAY_TIMEOUT)
      .message("Unsatisfiable Request (only-if-cached)")
      .body(EMPTY_RESPONSE)
      .sentRequestAtMillis(-1L)
      .receivedResponseAtMillis(System.currentTimeMillis())
      .build().also {
        listener.satisfactionFailure(call, it)
      }
}

// If we don't need the network, we're done.
if (networkRequest == null) {
  return cacheResponse!!.newBuilder()
      .cacheResponse(stripBody(cacheResponse))
      .build().also {
        listener.cacheHit(call, it)
      }
}
```

+ 首先根据request从cache获取缓存response
+ 接着通过CacheStrategy工厂类生成strategy缓存策略对象，用来判定是使用缓存还是进行网络请求
+  接着trackResponse通过缓存策略更新命中情况， 猜测跟缓存淘汰策略有关
+ 接着校验， 如果cacheCandidate不为null的情况下cacheResponse为null， 则表示根据缓存策略判断该缓存不适用，直接关闭缓存body
+ 然后继续， 校验如果networkRequest和cacheResponse都为null的情况下，表示已经禁用了网络，并且缓存还为null，就构造一个504的response返回
+ 下面判断如果为networkRequest为nul表示网络被禁用， 直接使用强制缓存返回缓存response

接着我们向下分析

```kotlin
if (cacheResponse != null) {
  listener.cacheConditionalHit(call, cacheResponse)
} else if (cache != null) {
  listener.cacheMiss(call)
}

var networkResponse: Response? = null
try {
  networkResponse = chain.proceed(networkRequest)
} finally {
  // If we're crashing on I/O or otherwise, don't leak the cache body.
  if (networkResponse == null && cacheCandidate != null) {
    cacheCandidate.body?.closeQuietly()
  }
}

// If we have a cache response too, then we're doing a conditional get.
if (cacheResponse != null) {
  if (networkResponse?.code == HTTP_NOT_MODIFIED) {
    val response = cacheResponse.newBuilder()
        .headers(combine(cacheResponse.headers, networkResponse.headers))
        .sentRequestAtMillis(networkResponse.sentRequestAtMillis)
        .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis)
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build()

    networkResponse.body!!.close()

    // Update the cache after combining headers but before stripping the
    // Content-Encoding header (as performed by initContentStream()).
    cache!!.trackConditionalCacheHit()
    cache.update(cacheResponse, response)
    return response.also {
      listener.cacheHit(call, it)
    }
  } else {
    cacheResponse.body?.closeQuietly()
  }
}
```

+ 如果requesrt不为null， 并且缓存response不为null， 就增加缓存监听

+ 接着向下，通过网络请求获得response

+ 如果缓存response不为null的情况下， 就要检查是否更新缓存

  首先判断返回httpcode是否为304， 如果不是就直接关闭缓存；是的话直接返回缓存response，然后更新缓存命中

+ 如果cache不为null， 则表示开启了缓存模式， 根据请求方式、网络请求返回的响应码以及Cache-Control字段是否设置了no-store来判断是否可以缓存，如果可以则将响应数据写入缓存

+ 然后调用invalidatesCache校验请求方法

  ```kotlin
  fun invalidatesCache(method: String): Boolean = (method == "POST" ||
      method == "PATCH" ||
      method == "PUT" ||
      method == "DELETE" ||
      method == "MOVE") // WebDAV
  ```

  如果为上述几种，移除刚添加的缓存

+ 最后如果没有在上述的过程中命中缓存的话，则返回网络请求的结果。

总的来说，缓存拦截器就是根据缓存策略返回的结果来决定是否用缓存的，其intercept方法整体的业务逻辑可以用下面的表格来简单总结：

| 网络（networkRequest） | 缓存（cacheResponse） | 说明                                                |
| ---------------------- | --------------------- | --------------------------------------------------- |
| 不可用                 | 不可用                | 返回504                                             |
| 不可用                 | 可用                  | 直接使用缓存，不会进行网络请求                      |
| 可用                   | 不可用                | 发起网络请求，直接拿服务器返回的数据                |
| 可用                   | 可用                  | 发起网络请求，如果返回响应码304则使用缓存并更新缓存 |

总结

+ 有强制缓存和协商缓存两种， 强制缓存优先级高于协商缓存
+ 只支持磁盘缓存，不支持内存缓存
+ 只缓存GET请求

附上缓存流程图：

![image](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f312bedaafbe49daa620e2715274a924~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=915&h=858&s=67966&e=png&b=ffffff)

# OkHttp 入门探索（四）

> 之前我们详细叙述了OkHttp的使用以及前几个拦截器的实现，下面我们继续对剩下的几个拦截器进行探索，进行深入剖析

## ConnectInterceptor

还是照例先看源码实现：

```kotlin
@Throws(IOException::class)
override fun intercept(chain: Interceptor.Chain): Response {
  val realChain = chain as RealInterceptorChain
  val exchange = realChain.call.initExchange(chain)
  val connectedChain = realChain.copy(exchange = exchange)
  return connectedChain.proceed(realChain.request)
}
```

代码看上去只有几行，主要是想通过复用链接节省带宽，看下主要步骤：

+ 首先通过initExchange获取到一个exchange
+ 然后根据这个exchange去创建一条新的拦截器链执行后续拦截器

我们进去initExchange看下

```kotlin
internal fun initExchange(chain: RealInterceptorChain): Exchange {
  synchronized(this) {
    check(expectMoreExchanges) { "released" }
    check(!responseBodyOpen)
    check(!requestBodyOpen)
  }

  val exchangeFinder = this.exchangeFinder!!
  val codec = exchangeFinder.find(client, chain)
  val result = Exchange(this, eventListener, exchangeFinder, codec)
  this.interceptorScopedExchange = result
  this.exchange = result
  synchronized(this) {
    this.requestBodyOpen = true
    this.responseBodyOpen = true
  }

  if (canceled) throw IOException("Canceled")
  return result
}
```

这里有几个新的类，简单介绍下：

**ExchangeFinder类**：连接复用的关键实现类，上面代码中的ExchangeFinder对象是在重试和重定向拦截器RetryAndFollowUpInterceptor中创建的，至于是怎么创建的不必深究，只需知道ExchangeFinder的主要职责是查找和创建连接。

**ExchangeCodec类**：编解码器，主要负责对请求编码和对响应解码。

**Exchange类**：交换器，用于传输单个HTTP请求的Request和Response。

我们可以看出，获取到ExchangeCodec后生成excahnge，看下ExchangeCodec如何获取的

```kotlin
fun find(
  client: OkHttpClient,
  chain: RealInterceptorChain
): ExchangeCodec {
  try {
    val resultConnection = findHealthyConnection(
        connectTimeout = chain.connectTimeoutMillis,
        readTimeout = chain.readTimeoutMillis,
        writeTimeout = chain.writeTimeoutMillis,
        pingIntervalMillis = client.pingIntervalMillis,
        connectionRetryEnabled = client.retryOnConnectionFailure,
        doExtensiveHealthChecks = chain.request.method != "GET"
    )
    return resultConnection.newCodec(client, chain)
  } catch (e: RouteException) {
    trackFailure(e.lastConnectException)
    throw e
  } catch (e: IOException) {
    trackFailure(e)
    throw RouteException(e)
  }
}
```

代码流程比较简洁，主要是通过findHealthyConnection获取到链接，然后通过链接生成一个ExchangeCodec

看下newCodec

```kotlin
internal fun newCodec(client: OkHttpClient, chain: RealInterceptorChain): ExchangeCodec {
  val socket = this.socket!!
  val source = this.source!!
  val sink = this.sink!!
  val http2Connection = this.http2Connection

  return if (http2Connection != null) {
    Http2ExchangeCodec(client, this, chain, http2Connection)
  } else {
    socket.soTimeout = chain.readTimeoutMillis()
    source.timeout().timeout(chain.readTimeoutMillis.toLong(), MILLISECONDS)
    sink.timeout().timeout(chain.writeTimeoutMillis.toLong(), MILLISECONDS)
    Http1ExchangeCodec(client, this, source, sink)
  }
}
```

流程也很清晰，可以看出如果是http1返回Http1ExchangeCodec， http2返回Http2ExchangeCodec

接着回到find中， 看下findHealthyConnection如何获取到链接的，下面也是okhttp连接复用的核心

```kotlin
private fun findHealthyConnection(
  connectTimeout: Int,
  readTimeout: Int,
  writeTimeout: Int,
  pingIntervalMillis: Int,
  connectionRetryEnabled: Boolean,
  doExtensiveHealthChecks: Boolean
): RealConnection {
  while (true) {
    val candidate = findConnection(
        connectTimeout = connectTimeout,
        readTimeout = readTimeout,
        writeTimeout = writeTimeout,
        pingIntervalMillis = pingIntervalMillis,
        connectionRetryEnabled = connectionRetryEnabled
    )

    // Confirm that the connection is good.
    if (candidate.isHealthy(doExtensiveHealthChecks)) {
      return candidate
    }

    // If it isn't, take it out of the pool.
    candidate.noNewExchanges()

    // Make sure we have some routes left to try. One example where we may exhaust all the routes
    // would happen if we made a new connection and it immediately is detected as unhealthy.
    if (nextRouteToTry != null) continue

    val routesLeft = routeSelection?.hasNext() ?: true
    if (routesLeft) continue

    val routesSelectionLeft = routeSelector?.hasNext() ?: true
    if (routesSelectionLeft) continue

    throw IOException("exhausted all routes")
  }
}
```

这里很明确，进入一个while循环

然后通过findConnection拿到链接，通过isHealthy校验链接是否合法

```kotlin
fun isHealthy(doExtensiveChecks: Boolean): Boolean {
  assertThreadDoesntHoldLock()

  val nowNs = System.nanoTime()

  val rawSocket = this.rawSocket!!
  val socket = this.socket!!
  val source = this.source!!
  if (rawSocket.isClosed || socket.isClosed || socket.isInputShutdown ||
          socket.isOutputShutdown) {
    return false
  }
```

主要判断socket是否关闭，是否是关闭的输入输出流

如果合法，直接返回

如果不合法，标记为不可用并从连接池移除

我们一个一个看，先看下findConnection

```kotlin
private fun findConnection(
  connectTimeout: Int,
  readTimeout: Int,
  writeTimeout: Int,
  pingIntervalMillis: Int,
  connectionRetryEnabled: Boolean
): RealConnection {
  if (call.isCanceled()) throw IOException("Canceled")

  // Attempt to reuse the connection from the call.
  val callConnection = call.connection // This may be mutated by releaseConnectionNoEvents()!
  if (callConnection != null) {
    var toClose: Socket? = null
    synchronized(callConnection) {
      if (callConnection.noNewExchanges || !sameHostAndPort(callConnection.route().address.url)) {
        toClose = call.releaseConnectionNoEvents()
      }
    }

    // If the call's connection wasn't released, reuse it. We don't call connectionAcquired() here
    // because we already acquired it.
    if (call.connection != null) {
      check(toClose == null)
      return callConnection
    }

    // The call's connection was released.
    toClose?.closeQuietly()
    eventListener.connectionReleased(call, callConnection)
  }

  // We need a new connection. Give it fresh stats.
  refusedStreamCount = 0
  connectionShutdownCount = 0
  otherFailureCount = 0

  // Attempt to get a connection from the pool.
  if (connectionPool.callAcquirePooledConnection(address, call, null, false)) {
    val result = call.connection!!
    eventListener.connectionAcquired(call, result)
    return result
  }

  // Nothing in the pool. Figure out what route we'll try next.
  val routes: List<Route>?
  val route: Route
  if (nextRouteToTry != null) {
    // Use a route from a preceding coalesced connection.
    routes = null
    route = nextRouteToTry!!
    nextRouteToTry = null
  } else if (routeSelection != null && routeSelection!!.hasNext()) {
    // Use a route from an existing route selection.
    routes = null
    route = routeSelection!!.next()
  } else {
    // Compute a new route selection. This is a blocking operation!
    var localRouteSelector = routeSelector
    if (localRouteSelector == null) {
      localRouteSelector = RouteSelector(address, call.client.routeDatabase, call, eventListener)
      this.routeSelector = localRouteSelector
    }
    val localRouteSelection = localRouteSelector.next()
    routeSelection = localRouteSelection
    routes = localRouteSelection.routes

    if (call.isCanceled()) throw IOException("Canceled")

    // Now that we have a set of IP addresses, make another attempt at getting a connection from
    // the pool. We have a better chance of matching thanks to connection coalescing.
    if (connectionPool.callAcquirePooledConnection(address, call, routes, false)) {
      val result = call.connection!!
      eventListener.connectionAcquired(call, result)
      return result
    }

    route = localRouteSelection.next()
  }

  // Connect. Tell the call about the connecting call so async cancels work.
  val newConnection = RealConnection(connectionPool, route)
  call.connectionToCancel = newConnection
  try {
    newConnection.connect(
        connectTimeout,
        readTimeout,
        writeTimeout,
        pingIntervalMillis,
        connectionRetryEnabled,
        call,
        eventListener
    )
  } finally {
    call.connectionToCancel = null
  }
  call.client.routeDatabase.connected(newConnection.route())

  // If we raced another call connecting to this host, coalesce the connections. This makes for 3
  // different lookups in the connection pool!
  if (connectionPool.callAcquirePooledConnection(address, call, routes, true)) {
    val result = call.connection!!
    nextRouteToTry = route
    newConnection.socket().closeQuietly()
    eventListener.connectionAcquired(call, result)
    return result
  }

  synchronized(newConnection) {
    connectionPool.put(newConnection)
    call.acquireConnectionNoEvents(newConnection)
  }

  eventListener.connectionAcquired(call, newConnection)
  return newConnection
}
```

代码流程很长，总结以下：

+ 第一次，尝试使用已经给当前请求分配到的连接。首次发起请求的话是没有连接的，这种情况拿到的连接会是null，需要建立连接；如果这个请求在进行重定向就可能复用上次的连接。如果存在连接，则先检查这个连接的noNewExchanges标志位和主机跟端口自创建时起有没有更改过，当noNewExchanges标志位为true（即代表无法在当前连接上创建新的流）或者主机端口变了，说明这个连接不可复用，则释放当前连接，最后会关闭Socket和执行连接被释放的回调。要是连接可用就会直接返回，不会进行后续的查找；
+ 第二次，尝试从连接池里获取连接。这里传给连接池的routes和requireMultiplexed这两个参数均为null，也就是要求不带路由信息和不带多路复用，这时如果连接池中存在跟当前address匹配的连接，则直接将其返回；
+ 第三次，拿到请求的所有路由后再次尝试从连接池里获取连接。这次由于传入了路由信息（多个route，即多个IP地址），在连接池里可能会找到因为请求合并而匹配到的不带多路复用的连接，如果能找到则直接返回这个连接；
+ 第四次，前三次都实在没找到，那就新建一个连接，并进行TCP+TLS握手与服务器建立连接，注意这个新建的连接并不会立即返回，需要根据下一次查找的结果来决定要不要用这个连接；
+ 第五次，也是最后一次查找，再一次尝试从连接池里获取连接。这次带上路由信息，并要求多路复用，这里是针对HTTP/2的操作，为了确保HTTP/2多路复用的特性防止创建多个连接，这里再次确认连接池里是否已经存在同样的连接，如果存在则直接使用已有的连接，并且释放刚刚新建的连接。如果还是没有匹配到，则将新建的连接放入连接池以用于下一次复用并返回

上述就是okhttp的链接复用流程

可以看到复用主要是通过连接池来做的，我们看下连接池的代码

```kotlin
class ConnectionPool internal constructor(
  internal val delegate: RealConnectionPool
) {
  constructor(
    maxIdleConnections: Int,
    keepAliveDuration: Long,
    timeUnit: TimeUnit
  ) : this(RealConnectionPool(
      taskRunner = TaskRunner.INSTANCE,
      maxIdleConnections = maxIdleConnections,
      keepAliveDuration = keepAliveDuration,
      timeUnit = timeUnit
  ))

  constructor() : this(5, 5, TimeUnit.MINUTES)
  }
```

可以看到主要入参有 最大连接数、长连时间、时间单位

默认最大连接数5， 保活时间 5 min

内部还有一个代理类delegate，可以知道关于连接池的工作都是由他负责

我们看下关于连接的添加、获取、清理三个方法

**添加**

```kotlin
fun put(connection: RealConnection) {
  connection.assertThreadHoldsLock()

  connections.add(connection)
  cleanupQueue.schedule(cleanupTask)
}
```

可以看到进行了两个操作

+ 给connections队列添加了一个连接
+ 给cleanupQueue队列添加了一个清理任务

这里cleanupTask是进行连接清理的任务

```kotlin
private val cleanupTask = object : Task("$okHttpName ConnectionPool") {
  override fun runOnce() = cleanup(System.nanoTime())
}

fun cleanup(now: Long): Long {
    var inUseConnectionCount = 0
    var idleConnectionCount = 0
    var longestIdleConnection: RealConnection? = null
    var longestIdleDurationNs = Long.MIN_VALUE

    // Find either a connection to evict, or the time that the next eviction is due.
    for (connection in connections) {
      synchronized(connection) {
        // If the connection is in use, keep searching.
        if (pruneAndGetAllocationCount(connection, now) > 0) {
          inUseConnectionCount++
        } else {
          idleConnectionCount++

          // If the connection is ready to be evicted, we're done.
          val idleDurationNs = now - connection.idleAtNs
          if (idleDurationNs > longestIdleDurationNs) {
            longestIdleDurationNs = idleDurationNs
            longestIdleConnection = connection
          } else {
            Unit
          }
        }
      }
    }

    when {
      longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections -> {
        // We've chosen a connection to evict. Confirm it's still okay to be evict, then close it.
        val connection = longestIdleConnection!!
        synchronized(connection) {
          if (connection.calls.isNotEmpty()) return 0L // No longer idle.
          if (connection.idleAtNs + longestIdleDurationNs != now) return 0L // No longer oldest.
          connection.noNewExchanges = true
          connections.remove(longestIdleConnection)
        }

        connection.socket().closeQuietly()
        if (connections.isEmpty()) cleanupQueue.cancelAll()

        // Clean up again immediately.
        return 0L
      }

      idleConnectionCount > 0 -> {
        // A connection will be ready to evict soon.
        return keepAliveDurationNs - longestIdleDurationNs
      }

      inUseConnectionCount > 0 -> {
        // All connections are in use. It'll be at least the keep alive duration 'til we run
        // again.
        return keepAliveDurationNs
      }

      else -> {
        // No connections, idle or in use.
        return -1
      }
    }
  }
```

主要是对空闲连接进行一波清理，主要思路如下

+ 遍历连接池中的所有连接，如果当前连接正在被使用，则正在使用的连接计数+1，否则的话空闲连接计数+1，同时在遍历的过程中找出闲置时间最长的连接及其闲置时长；

+ 根据不同情况尝试清理空闲连接：

1. 当连接的最长空闲时长不小于设置的保活时长（默认是5分钟）或者空闲连接数大于设置支持的最大空闲连接数（默认是5个），则从闲置最久的连接下手，先检测这个连接有没有承载请求，如果有的话就说明这个连接即将会被用到，这时直接返回0，不会清除；然后检测这个连接的idleAtNs时间戳与最长闲置时长的和是否等于当前程序执行的时长，个人理解这是为了双重保证要被处理的连接是闲置最久的连接，要是还有其他连接比当前连接闲置更久，则直接返回0，不会清除；最后要是这个连接可以清除，则将其标记为不接受新的数据流，并从连接队列里面移除它，关闭Socket并返回0。
2. 当没有超出上面所述的限制，并且空闲连接数大于0，则返回最长空闲时间距离设置的保活时长的时间差，也就是执行下一次清除要等待的时间；
3. 走到这里说明现在没有空闲的连接，当正在使用的连接数大于0，则直接返回设置的保活时长，过了这段时间之后再来尝试清理；
4. 其他情况，也就是连接池里没有连接，则直接返回-1，不需要清理。

其中，cleanup方法返回的Long型值代表下一次执行定时清除任务的周期，等时间到了就会自动执行cleanup重新尝试清除空闲连接，如果返回的是-1就说明不需要重新安排任务了。追踪cleanup方法的调用处，可以发现在往连接池存入新连接和外部通知连接池要释放池中某个连接时会执行cleanup方法，这两种情况都会触发定时清除任务移除池中长时间用不到的连接，以达到连接池自动维护连接的目的。

**获取连接**

```kotlin
fun callAcquirePooledConnection(
  address: Address,
  call: RealCall,
  routes: List<Route>?,
  requireMultiplexed: Boolean
): Boolean {
  for (connection in connections) {
    synchronized(connection) {
      if (requireMultiplexed && !connection.isMultiplexed) return@synchronized
      if (!connection.isEligible(address, routes)) return@synchronized
      call.acquireConnectionNoEvents(connection)
      return true
    }
  }
  return false
}
```

获取连接的时候会遍历连接池里所有的连接去判断是否有匹配的连接：

- 如果要求多路复用而当前的连接不是HTTP/2连接时，说明当前连接不适用，则跳过当前连接，继续遍历；
- 判断当前连接是否能够承载指向对应address的数据流，如果不能则跳过当前连接继续遍历，如果能则将这个连接设置到call里去，为这个连接所承载的call集合添加一个对当前call的弱引用，最后返回这个连接。

我们看到isEligible决定了连接能够匹配，我们看下如何校验的



```kotlin
internal fun isEligible(address: Address, routes: List<Route>?): Boolean {
  assertThreadHoldsLock()

  // If this connection is not accepting new exchanges, we're done.
  if (calls.size >= allocationLimit || noNewExchanges) return false

  // If the non-host fields of the address don't overlap, we're done.
  if (!this.route.address.equalsNonHost(address)) return false

  // If the host exactly matches, we're done: this connection can carry the address.
  if (address.url.host == this.route().address.url.host) {
    return true // This connection is a perfect match.
  }

  // At this point we don't have a hostname match. But we still be able to carry the request if
  // our connection coalescing requirements are met. See also:
  // https://hpbn.co/optimizing-application-delivery/#eliminate-domain-sharding
  // https://daniel.haxx.se/blog/2016/08/18/http2-connection-coalescing/

  // 1. This connection must be HTTP/2.
  if (http2Connection == null) return false

  // 2. The routes must share an IP address.
  if (routes == null || !routeMatchesAny(routes)) return false

  // 3. This connection's server certificate's must cover the new host.
  if (address.hostnameVerifier !== OkHostnameVerifier) return false
  if (!supportsUrl(address.url)) return false

  // 4. Certificate pinning must match the host.
  try {
    address.certificatePinner!!.check(address.url.host, handshake()!!.peerCertificates)
  } catch (_: SSLPeerUnverifiedException) {
    return false
  }

  return true // The caller's address can be carried by this connection.
}
```

匹配逻辑如下：

- 如果连接已达到可以承载的最大并发流数或者不允许在此连接上创建新的流，则匹配失败；
- 如果address中非host部分的字段匹配不上，则匹配失败；
- 如果address的host完全匹配，则匹配成功，直接返回true；
- 前面的判定都没有返回结果的话，说明主机名不匹配，不过在后续判定中有可能因为连接合并而匹配到，而后续判定是针对HTTP/2的，如果不是HTTP/2则直接返回false，匹配失败；
- 如果没有路由信息或者IP地址不匹配，则匹配失败；
- 如果证书不匹配，则匹配失败；
- 如果url不合法，则匹配失败；
- 进行证书pinning匹配，能匹配上就返回true。

**清理连接**

```kotlin
fun evictAll() {
  val i = connections.iterator()
  while (i.hasNext()) {
    val connection = i.next()
    val socketToClose = synchronized(connection) {
      if (connection.calls.isEmpty()) {
        i.remove()
        connection.noNewExchanges = true
        return@synchronized connection.socket()
      } else {
        return@synchronized null
      }
    }
    socketToClose?.closeQuietly()
  }

  if (connections.isEmpty()) cleanupQueue.cancelAll()
}
```

很直观的能看到，这里先对链接进行遍历

如果链接处于空闲态就标记并进行清理

上述就是关于ConnectInterceptor的全部内容，下面我们乘胜追击，看下最后一个拦截器CallServerInterceptor

## CallServerInterceptor

先上源码

```kotlin
override fun intercept(chain: Interceptor.Chain): Response {
  val realChain = chain as RealInterceptorChain
  val exchange = realChain.exchange!!
  val request = realChain.request
  val requestBody = request.body
  val sentRequestMillis = System.currentTimeMillis()

  exchange.writeRequestHeaders(request)

  var invokeStartEvent = true
  var responseBuilder: Response.Builder? = null
  if (HttpMethod.permitsRequestBody(request.method) && requestBody != null) {
    // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
    // Continue" response before transmitting the request body. If we don't get that, return
    // what we did get (such as a 4xx response) without ever transmitting the request body.
    if ("100-continue".equals(request.header("Expect"), ignoreCase = true)) {
      exchange.flushRequest()
      responseBuilder = exchange.readResponseHeaders(expectContinue = true)
      exchange.responseHeadersStart()
      invokeStartEvent = false
    }
    if (responseBuilder == null) {
      if (requestBody.isDuplex()) {
        // Prepare a duplex body so that the application can send a request body later.
        exchange.flushRequest()
        val bufferedRequestBody = exchange.createRequestBody(request, true).buffer()
        requestBody.writeTo(bufferedRequestBody)
      } else {
        // Write the request body if the "Expect: 100-continue" expectation was met.
        val bufferedRequestBody = exchange.createRequestBody(request, false).buffer()
        requestBody.writeTo(bufferedRequestBody)
        bufferedRequestBody.close()
      }
    } else {
      exchange.noRequestBody()
      if (!exchange.connection.isMultiplexed) {
        // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
        // from being reused. Otherwise we're still obligated to transmit the request body to
        // leave the connection in a consistent state.
        exchange.noNewExchangesOnConnection()
      }
    }
  } else {
    exchange.noRequestBody()
  }

  if (requestBody == null || !requestBody.isDuplex()) {
    exchange.finishRequest()
  }
  if (responseBuilder == null) {
    responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
    if (invokeStartEvent) {
      exchange.responseHeadersStart()
      invokeStartEvent = false
    }
  }
  var response = responseBuilder
      .request(request)
      .handshake(exchange.connection.handshake())
      .sentRequestAtMillis(sentRequestMillis)
      .receivedResponseAtMillis(System.currentTimeMillis())
      .build()
  var code = response.code
  if (code == 100) {
    // Server sent a 100-continue even though we did not request one. Try again to read the actual
    // response status.
    responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
    if (invokeStartEvent) {
      exchange.responseHeadersStart()
    }
    response = responseBuilder
        .request(request)
        .handshake(exchange.connection.handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build()
    code = response.code
  }

  exchange.responseHeadersEnd(response)

  response = if (forWebSocket && code == 101) {
    // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
    response.newBuilder()
        .body(EMPTY_RESPONSE)
        .build()
  } else {
    response.newBuilder()
        .body(exchange.openResponseBody(response))
        .build()
  }
  if ("close".equals(response.request.header("Connection"), ignoreCase = true) ||
      "close".equals(response.header("Connection"), ignoreCase = true)) {
    exchange.noNewExchangesOnConnection()
  }
  if ((code == 204 || code == 205) && response.body?.contentLength() ?: -1L > 0L) {
    throw ProtocolException(
        "HTTP $code had non-zero Content-Length: ${response.body?.contentLength()}")
  }
  return response
}
```

代码篇幅较长，我们分段来看：

```kotlin
exchange.writeRequestHeaders(request)
```

首先这里写入请求头

```kotlin
fun writeRequestHeaders(request: Request) {
  try {
    eventListener.requestHeadersStart(call)
    codec.writeRequestHeaders(request)
    eventListener.requestHeadersEnd(call, request)
  } catch (e: IOException) {
    eventListener.requestFailed(call, e)
    trackFailure(e)
    throw e
  }
}

  override fun writeRequestHeaders(request: Request) {
    val requestLine = RequestLine.get(request, connection.route().proxy.type())
    writeRequest(request.headers, requestLine)
  }
  
    fun writeRequest(headers: Headers, requestLine: String) {
    check(state == STATE_IDLE) { "state: $state" }
    sink.writeUtf8(requestLine).writeUtf8("\r\n")
    for (i in 0 until headers.size) {
      sink.writeUtf8(headers.name(i))
          .writeUtf8(": ")
          .writeUtf8(headers.value(i))
          .writeUtf8("\r\n")
    }
    sink.writeUtf8("\r\n")
    state = STATE_OPEN_REQUEST_BODY
  }
```

写入是通过codec来做的， 具体实现有两种，分别为http1 和 http2

上面代码是以http1为例，主要是通过循环方式不断写入头部， 以\r\n作为分隔符

这里的state判断是做状态标记的，类似状态机，如果跨状态执行操作，就会出现异常

```kotlin
var invokeStartEvent = true
var responseBuilder: Response.Builder? = null
if (HttpMethod.permitsRequestBody(request.method) && requestBody != null) {
  // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
  // Continue" response before transmitting the request body. If we don't get that, return
  // what we did get (such as a 4xx response) without ever transmitting the request body.
  if ("100-continue".equals(request.header("Expect"), ignoreCase = true)) {
    exchange.flushRequest()
    responseBuilder = exchange.readResponseHeaders(expectContinue = true)
    exchange.responseHeadersStart()
    invokeStartEvent = false
  }
  if (responseBuilder == null) {
    if (requestBody.isDuplex()) {
      // Prepare a duplex body so that the application can send a request body later.
      exchange.flushRequest()
      val bufferedRequestBody = exchange.createRequestBody(request, true).buffer()
      requestBody.writeTo(bufferedRequestBody)
    } else {
      // Write the request body if the "Expect: 100-continue" expectation was met.
      val bufferedRequestBody = exchange.createRequestBody(request, false).buffer()
      requestBody.writeTo(bufferedRequestBody)
      bufferedRequestBody.close()
    }
  } else {
    exchange.noRequestBody()
    if (!exchange.connection.isMultiplexed) {
      // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
      // from being reused. Otherwise we're still obligated to transmit the request body to
      // leave the connection in a consistent state.
      exchange.noNewExchangesOnConnection()
    }
  }
} else {
  exchange.noRequestBody()
}
```

这里有个预先的处理， 如果请求的Expect字段为100-continue的情况下，首先会先请求服务端，询问是否继续写入请求响应部分，如果可以服务端返回100 ，不可以返回对应错误码。（这里这么做主要是在大文件传输中，能够提高效率减少不必要的数据传输）

首先判断了如果Expect字段是100-continue，那么会直接调用httpCodec.flushRequest()请求网络，把刚才写入的Header发送到服务器，请求的执行就是这么简单，先写入再请求。

根据responseBuilder的结果决定是否写入body，这里也有两个实现：

FormBody：用于发送表单数据（通常是 `application/x-www-form-urlencoded` 格式）。

MultipartBody：用于发送多部分数据（例如文件上传），通常是 `multipart/form-data` 格式。

我们选择FormBody来看下：

```kotlin
@Throws(IOException::class)
override fun writeTo(sink: BufferedSink) {
  writeOrCountBytes(sink, false)
}

  private fun writeOrCountBytes(sink: BufferedSink?, countBytes: Boolean): Long {
    var byteCount = 0L
    val buffer: Buffer = if (countBytes) Buffer() else sink!!.buffer

    for (i in 0 until encodedNames.size) {
      if (i > 0) buffer.writeByte('&'.toInt())
      buffer.writeUtf8(encodedNames[i])
      buffer.writeByte('='.toInt())
      buffer.writeUtf8(encodedValues[i])
    }

    if (countBytes) {
      byteCount = buffer.size
      buffer.clear()
    }

    return byteCount
  }
```

这里可以看出也是循环遍历encodedNames

使用 & 来分割各个键值对

最后返回总的字节数

```kotlin
if (requestBody == null || !requestBody.isDuplex()) {
  exchange.finishRequest()
}
```

在发送 HTTP 请求时，如果请求体不存在或不支持双工模式，则完成请求的发送。

```kotlin
if (responseBuilder == null) {
  responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
  if (invokeStartEvent) {
    exchange.responseHeadersStart()
    invokeStartEvent = false
  }
}
var response = responseBuilder
    .request(request)
    .handshake(exchange.connection.handshake())
    .sentRequestAtMillis(sentRequestMillis)
    .receivedResponseAtMillis(System.currentTimeMillis())
    .build()
var code = response.code
if (code == 100) {
  // Server sent a 100-continue even though we did not request one. Try again to read the actual
  // response status.
  responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
  if (invokeStartEvent) {
    exchange.responseHeadersStart()
  }
  response = responseBuilder
      .request(request)
      .handshake(exchange.connection.handshake())
      .sentRequestAtMillis(sentRequestMillis)
      .receivedResponseAtMillis(System.currentTimeMillis())
      .build()
  code = response.code
}

exchange.responseHeadersEnd(response)
    response = if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      response.newBuilder()
          .body(EMPTY_RESPONSE)
          .build()
    } else {
      response.newBuilder()
          .body(exchange.openResponseBody(response))
          .build()
    }
```

这里是对respnse进行构建， 填充body

```kotlin
if ("close".equals(response.request.header("Connection"), ignoreCase = true) ||
    "close".equals(response.header("Connection"), ignoreCase = true)) {
  exchange.noNewExchangesOnConnection()
}
if ((code == 204 || code == 205) && response.body?.contentLength() ?: -1L > 0L) {
  throw ProtocolException(
      "HTTP $code had non-zero Content-Length: ${response.body?.contentLength()}")
}
return response
```

+ 如果Connection为关闭就关闭连接
+ 如果返回错误码为 204或 205，不应该包含内容的情况下， body有长度则抛出异常
+ 然后返回response

到这里整个okhttp的入门分析就结束了， 通过这段时间的学习也对这个网络请求框架有了一个新的认识，分析代码的过程中也回顾了http的一些知识， 通过阅读源码，编程能力和代码分析能力也在不断提高，希望自己能够坚持下去，继续写一些开源库的分析吧！
