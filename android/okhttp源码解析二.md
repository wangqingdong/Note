## 一、前言

前一篇文章简单的分析了OkHttp的源码，本篇继续介绍okhttp的请求过程。
我们知道OkHttp有两种方式发起请求：
- 同步：直接在当前线程执行请求
```java
Response response = okhttpClient.newCall(request).execute();
```
- 异步：将当前任务加到任务队列中，执行异步请求。
```java
okhttpClient.newCall(request).enqueue(callback);
```
下面看下具体的请求过程。

## 二、请求过程

在上一篇说到OkHttpClient的newCall()实际上返回的是RealCall的实例，看下它的构造函数：

```java
private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    //是不是webSocket请求，webSocket是用来建立长连接的
    this.forWebSocket = forWebSocket;
    //创建重试与重定向拦截器
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
    this.timeout = new AsyncTimeout() {
      @Override protected void timedOut() {
        cancel();
      }
    };
    this.timeout.timeout(client.callTimeoutMillis(), MILLISECONDS);
}
```
RealCall的构造函数需要传入一个OKHttpClient对象和Request对象。因此RealCall包装了OKHttpClient和Request对象，可以很方便地使用它们。

在RealCall类中还有三个重要的方法：
- execute():同步请求
- enqueue(callback):异步请求
- getResponseWithInterceptorChain()：添加拦截器、发起请求、获取响应


### 2.1、同步请求

```java
@Override 
public Response execute() throws IOException {
    synchroized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    timeout.enter();
    eventListener.callStart(this);
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      e = timeoutExit(e);
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      client.dispatcher().finished(this);
    }
}
```
execute的主要实现了下面几个功能：

1. 判断call是否执行过，可以看出每个Call对象只能使用一次原则
2. 把RealCall交给Okhttp的Dispathcer,放到一个队列里。
3. 调用getResponseWithInterceptorChain()函数发起完整的网络请求流程,获取HTTP返回结果。
4. 最后通知dispatcher执行完毕，从队列里删除该对象。

## 2.2、异步请求

异步请求也很简单，无非就是将getResponseWithInterceptorChain 放到线程中去执行而已。因为线程创建是比较好资源的，所以有了线程池，okhttp也不例外。线程池执行的单元为runanble，所以Okhttp也将异步请求封装了一个Ruannble，这个Ruannalbe就是AsyncCall.需要注意的是这个AsyncCall可不是RealCall的子类，也不是Call接口的实现类。具体看下okhttp是怎么实现异步请求的吧：

```java
@Override 
public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```
同样的，异步请求也是将执行请求的线程放入dispatcher维护的队列里，由dispatcher调度。而发起请求的方法放在子线程AsyncCall中：
```java
void executeOn(ExecutorService executorService) {
      assert (!Thread.holdsLock(client.dispatcher()));
      boolean success = false;
      try {
        executorService.execute(this);
        success = true;
      } catch (RejectedExecutionException e) {
        InterruptedIOException ioException = new InterruptedIOException("executor rejected");
        ioException.initCause(e);
        eventListener.callFailed(RealCall.this, ioException);
        responseCallback.onFailure(RealCall.this, ioException);
      } finally {
        if (!success) {
          client.dispatcher().finished(this); // This call is no longer running!
        }
      }
}

@Override protected void execute() {
    boolean signalledCallback = false;
    timeout.enter();
    try {
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
    } catch (IOException e) {
        e = timeoutExit(e);
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
}
```

看下OkHttpClient的dispatcher()的方法：
```java
public Dispatcher dispatcher() {
    return dispatcher;
}
```
返回一个Dispatcher实例，在OkHttpClient.Bulider的构造函数中会初始化Dispatche：
```java
public Builder() {
    dispatcher = new Dispatcher();
    //省略部分代码
}
```

Dispatcher（任务调度器）是操作同步和异步Call的地方，并负责执行异步AsyncCall。首先看下Dispatcher类几个重要变量的说明：
```java
  //最大并发请求数为64
  private int maxRequests = 64;

  //每个主机最大请求数为5 
  private int maxRequestsPerHost = 5;

  //请求线程池
  private @Nullable ExecutorService executorService;

  //将要运行的异步请求队列
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  //正在运行的异步请求队列，包含已经取消但未执行完的请求
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  //正在运行的同步请求队列，包含已经取消但未执行完的请求
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

```
Dispatcher使用了一个readyAsyncCalls保存了同步任务；对于异步请求，Dispatcher使用了两个Deque，分别是runningAsyncCalls,runningSyncCalls 一个保存准备执行的请求，一个保存正在执行的请求。

为什么要用两个呢？因为Dispatcher默认支持最大的并发请求是64个，单个Host最多执行5个并发请求，如果超过，则Call会先被放入到readyAsyncCall中，当出现空闲的线程时，再将readyAsyncCall中的线程移入到runningAsynCalls中，执行请求。




继续看看executorService，如下：
```java
public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
}
```
Dispatcher初始化了一个线程池，核心线程的数量为0 ，最大的线程数量为Integer.MAX_VALUE，空闲线程存在的最大时间为60秒，这个线程类似于CacheThreadPool，比较适合执行大量的耗时比较少的任务。同时Dispatcher也可以来设置自己线程池。

Dispatcher大概了解之后，回到之前说的，call的同步和异步请求方法。

### 2.1、同步请求

在上一篇文章中说到了同步请求：
```java
...
client.dispatcher().executed(this);
Response result = getResponseWithInterceptorChain();
...
client.dispatcher().finished(this);

```
这里调用了Dispatche的executed()方法:
```java
/** Used by {@code Call#execute} to signal it is in-flight. */
synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
}
```
将RealCall放入正在请求的同步队列中，而实际上执行网络请求的是