# OKHttp（3.12.0）源码分析（一）

## 一、基本使用方法
```java
//1.创建OkHttpClient对象
OkHttpClient  okHttpClient = new OkHttpClient.Builder()
                .build();
//2.通过new FormBody()调用build方法,创建一个RequestBody,可以用add添加键值对
RequestBody  requestBody = new FormBody.Builder()
                .add("name","jack")
                .add("age","25")
                .build();
//3.创建Request对象，设置URL地址，将RequestBody作为post方法的参数传入
Request request = new Request.Builder()
                .url("url")
                .post(requestBody)
                .build();
//4.创建一个call对象,参数就是Request请求对象
Call call = okHttpClient.newCall(request);
//5.请求加入调度,重写回调方法
call.enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {

    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {

    }
});
```
可以总结成四个步骤：
- 构建OkHttpClient实例
- 创建Request对象
- 创建Call对象
- 发起请求：同步请求调用call.execute()；异步请求调用call.enqueue(callback)，处理响应

## 二、OkHttpClient
OkHttp支持两种构造方式：

### 2.1、new方式
```java
public OkHttpClient() {
    this(new Builder());
}
```
可以看出这种方式不需要配置参数，都是通过Builder默认的：

```java
public Builder() {
    dispatcher = new Dispatcher();
    protocols = DEFAULT_PROTOCOLS;
    connectionSpecs = DEFAULT_CONNECTION_SPECS;
    eventListenerFactory = EventListener.factory(EventListener.NONE);
    proxySelector = ProxySelector.getDefault();
    if (proxySelector == null) {
        proxySelector = new NullProxySelector();
    }
    cookieJar = CookieJar.NO_COOKIES;
    socketFactory = SocketFactory.getDefault();
    hostnameVerifier = OkHostnameVerifier.INSTANCE;
    certificatePinner = CertificatePinner.DEFAULT;
    proxyAuthenticator = Authenticator.NONE;
    authenticator = Authenticator.NONE;
    connectionPool = new ConnectionPool();
    dns = Dns.SYSTEM;
    followSslRedirects = true;
    followRedirects = true;
    retryOnConnectionFailure = true;
    callTimeout = 0;
    connectTimeout = 10_000;
    readTimeout = 10_000;
    writeTimeout = 10_000;
    pingInterval = 0;
}
```
### 2.2、build模式

通过Builder配置参数，build()方法返回一个OkHttpCilent实例
```java
public OkHttpClient build() {
    return new OkHttpClient(this);
}
```
OkHttpClient初始化的时候使用了两种设计模式：

- 建造者模式
- 外观模式
  
## 三、RequestBody
RequestBody抽象类主要做了

- 获得请求体的数据类型、
- 获得请求体的数据长度、
- 将请求体写入到流中

这三件事。通过RequestBody源码可以看到：

### 3.1、两个抽象方法

```java
/** Returns the Content-Type header for this body. */
public abstract @Nullable MediaType contentType();


/** Writes the content of this request to {@code sink}. */
public abstract void writeTo(BufferedSink sink) throws IOException;
```
RequestBody的实现类中必须实现contentType()方法指定报文类型（具体类型可以查看HTTP协议的相关内容）。通过writeTo()方法将报文写入流中。

### 3.2、获取请求内容长度

```java
/**
* Returns the number of bytes that will be written to {@code sink} in a call to {@link #writeTo},
* or -1 if that count is unknown.
*/
public long contentLength() throws IOException {
    return -1;
}
```

这不是抽象方法，是因为它的具体实现不会影响到RequestBody能不能用，但是一般这个方法都会在子类中复写。

### 3.3、创建RequestBody实例

同时为了发送字符串和文件内容方便，RequestBody还提供了静态工具方法，create的多种重载形式：

```java
/**
* Returns a new request body that transmits {@code content}. If {@code contentType} is non-null
* and lacks a charset, this will use UTF-8.
*/
public static RequestBody create(@Nullable MediaType contentType, String content) {
    Charset charset = Util.UTF_8;
    if (contentType != null) {
        charset = contentType.charset();
        if (charset == null) {
            charset = Util.UTF_8;
            contentType = MediaType.parse(contentType + "; charset=utf-8");
        }
    }
    byte[] bytes = content.getBytes(charset);
    return create(contentType, bytes);
}

/** Returns a new request body that transmits {@code content}. */
public static RequestBody create(final @Nullable MediaType contentType, final byte[] content) {
    return create(contentType, content, 0, content.length);
}
```
最终调用的是：
```java
/** Returns a new request body that transmits {@code content}. */
public static RequestBody create(final @Nullable MediaType contentType, final byte[] content,final int offset, final int byteCount) {
    if (content == null) throw new NullPointerException("content == null");
    Util.checkOffsetAndCount(content.length, offset, byteCount);
    return new RequestBody() {
      @Override public @Nullable MediaType contentType() {
        return contentType;
      }

      @Override public long contentLength() {
        return byteCount;
      }

      @Override public void writeTo(BufferedSink sink) throws IOException {
        sink.write(content, offset, byteCount);
      }
    };
}
```
上面是String类型参数，而要上传文件时：
```java
/** Returns a new request body that transmits the content of {@code file}. */
public static RequestBody create(final @Nullable MediaType contentType, final File file) {
    if (file == null) throw new NullPointerException("file == null");

    return new RequestBody() {
      @Override public @Nullable MediaType contentType() {
        return contentType;
      }

      @Override public long contentLength() {
        return file.length();
      }

      @Override public void writeTo(BufferedSink sink) throws IOException {
        Source source = null;
        try {
          source = Okio.source(file);
          sink.writeAll(source);
        } finally {
          Util.closeQuietly(source);
        }
      }
    };
}
```
通过上面的方法我们可以创建出我们想要的任何类型的请求正文。而OkHttp为我们提供了两个RequestBody的子类：

### 3.4、RequestBody的两个子类

**FormBody（表单）类：**

用于实现一般键值对（都是字符串类型）的上传。

主要通过两个List集合来存储动态增加的键和值，最后一次性写入输入流的缓存区。

**MultipartBody（多部分）类：**

提到“部分”，要说Http的“多部分上传”，每一“部分”都包含请求头（可有可无）和请求体，“部分”可以看作是一个简略版的小报文（小请求），只不过这里的报文首部字段只能用实体首部字段。

“部分”这一概念通过MultipartBody类的内部类来具体实现。

通过集合的方式存储动态添加的“部分”。

## 四、Request

Request的主要作用是用来描述一次HTTP请求，是对url（地址），method（请求方式），header（请求头），body（请求体）的组合封装。看下Request的构造函数：
```java
Request(Builder builder) {
    this.url = builder.url;
    this.method = builder.method;
    this.headers = builder.headers.build();
    this.body = builder.body;
    this.tags = Util.immutableMap(builder.tags);
}
```
Request类中没有无参构造，只能通过Builder模式创建对象：

```java
public Request build() {
    if (url == null) throw new IllegalStateException("url == null");
    return new Request(this);
}
```
## 五、Call
Call是一个接口，是对Request的进一步封装：
```java
/**
 * A call is a request that has been prepared for execution. A call can be canceled. As this object
 * represents a single request/response pair (stream), it cannot be executed twice.
 */
public interface Call extends Cloneable {
  
  //返回当前Call的Request
  Request request();
  
  //同步请求
  Response execute() throws IOException;

  //异步请求
  void enqueue(Callback responseCallback);

  //取消请求
  void cancel();

  //判断是否执行过
  boolean isExecuted();

  //判断是否取消
  boolean isCanceled();

  Timeout timeout();

  Call clone();

  interface Factory {
    Call newCall(Request request);
  }
}
```
从源码中可以清楚地看到它是封装的请求，可以获取它所封装的请求，同时提供同步和异步的执行方法，最后是可以取消或复制，以及判断它的状态的方法，最后工厂类，可以提供Call的实例。

上面请求的示例中我们可以知道在发起请求前，都会通过okhttpClient的newCall()方法获取Call的实例，然后Call发起请求。


```java
/**
* okhttpClient#newCall()
* 
* Prepares the {@code request} to be executed at some point in the future.
*/
@Override 
public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
}
```

OkHttpClient的newCall()方法实际上调用的是RealCall类的newRealCall():

```java
/**
* RealCall#newRealCal()
*/
static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
}
```
所以OkHttpClient的newCall()实际上返回的是RealCall的实例。RealCall是call的实现类，是它发起的请求。至于请求的过程，在下一篇文章继续分析。

>参考资料:  
https://blog.piasy.com/2016/07/11/Understand-OkHttp/  
https://blog.csdn.net/zhangqilugrubby/article/details/80181373  
https://blog.csdn.net/chunqiuwei/article/details/71939952

## 六、RealCall


### 6.1、同步请求

这个方法主要做了4件事：

- 检查当前call是否已经被执行，每个call只能被执行一次。如果想要一个完全一样的call，可以利用call类的clone()方法。
- 
- 

Dispatcher任务调度器，和拦截器不同的是它不做Action事件处理，只做事件流向。在Okhttp中Dispatcher负责将每一次Requst进行分发，压栈到自己的线程池，并通过调用者自己不同的方式进行异步和同步处理！

真正发出请求，返回相应结果的还是getResponeWithInterceptorChain()方法。



所以在看下Dispatcher的enqueue方法：

同样的，该方法中首先判断请求有没有被执行，如果请求已经执行，那么直接抛出异常，如果请求没有执行，就会执行Dispatcher对象的enqueue方法。

同步请求和异步请求都涉及到了Dispatcher。在下一篇中继续分析Dispatcher。





```java
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    //获取用户自定义的拦截器对象：也可以不设置
    interceptors.addAll(client.interceptors());
    //以下拦截器是Okhttp内置的拦截器
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
}
```
从代码上来看，这个方法就是将客户端自定义的拦截器和Okhttp内置的拦截器放到一个List集合interceptors里面，然后跟最初构建的Request对象一块创建了RealInterceptorChain对象，RealInterceptorChain是Chain接口的实现类。这样就把一系列拦截器组合成一个链跟请求绑定起来，最终调用RealInterceptorChain的proceed来返回一个Response；

从getResponseWithInterceptorChain()方法中可以看出Okhttp添加自定义拦截器有两个位置：
1. 调用OkhttpClient对象addInterceptor方法添加的拦截器集合，会添加到拦截器链的顶部位置。
2. 调用OkhttpClient对象addNetwordInterceptor方法添加的拦截器集合，会将这些拦截器插入到ConnectInterceptor和CallServiceInterceptor两个拦截器的中间。

这两种自定义拦截器的区别就是：通过addInterceptor添加的拦截器可以不需要调用proceed方法。
而addNetwordInterceptor的拦截器则必须调用拦截器的procceed方法，以确保CallServerInterceptor拦截器的执行。

那么怎么样让 这些拦截器对象逐一工作呢？getResponseWithInterceptorChain方法的最后调用了RealInterceptorChain的proceed方法：

```java
//正式开始调用拦截器工作  
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,RealConnection connection){
  
    //省略部分与本文无关的代码
   
    // 调用链中的下一个拦截器
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpCodec, connection, index + 1, request);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

   //确保每个拦截器都调用了proceed方法（）
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }
   //省略部分代码
    return response;
}
```
上面的代码中，先实例化了RealInterceptorChain的对象，看下它的构造函数：
```java
//拦截器当前索引
private final int index;
private final Request request;
 
public RealInterceptorChain(List<Interceptor> interceptors, StreamAllocation streamAllocation,HttpCodec httpCodec, RealConnection connection, int index, Request request) {
    this.interceptors = interceptors;
   //省略部分与本文无关的代码
    this.index = index;//0
    this.request = request;
}
```
通过其构造函数可以知道RealInterceptorChain有一个index变量获取拦截器列表中对应位置的拦截器对象，但是procced方法并没有用for循环来遍历interceptors集合，而是重新new 一个RealInterceptorChain对象，且新对象的index在原来RealInterceptorChain对象index之上进行index+1,并把新的拦截器链对象RealInterceptorChain交给当前拦截器Interceptor 的intercept方法:

既然Interceptor的intercept方法接受的是一个新RealInterceptorChain对象，通过上面的代码来看目前index只能达到index=1,那么剩下的CacheIntercept对象是如何调用的呢？其实可以猜测一下：

**既然拦截器组成了一个链，那么肯定是 第一个内置拦截器RetryAndFollowUpInterceptor对象接受的Chain对象在intercept方法里面继续对新的Chain做了遍历下一个拦截器的操作！**

所以大致看下RetryAndFollowUpInterceptor对象的intercept方法：

```java
Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    //省略部分与本文无关的代码
 
    Response priorResponse = null;
    //死循环
    while (true) {
      //省略部分与本文无关的代码
      //上一个拦截器返回的响应对象
      Response response = null;
      //省略部分与本文无关的代码
	  //继续执行下一个chain的下一个
      response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);
      //省略部分与本文无关的代码

      Request followUp = followUpRequest(response);

      if (followUp == null) {
	    //返回return
        return response;
      }
 
     //省略部分与本文无关的代码
      priorResponse = response;
    }//end while
}
```
可以发现RetryAndFollowUpInterceptor对象的intercept有一个while（true）的循环，在循环里面有一行很重要的代码：
```java
response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);
```
跟getResponseWithInterceptorChain（）一样调用了proceed方法，根据上面的的讲解：调用Chain的proceed方法的时候会新生成一个RealInterceptorChain，其index的值是上一个拦截器所持有的Chain的index+1，这样就确保拦截器链逐条运行。查看BridgeInterceptor、CacheInterceptor等Okhttp内置拦截器就可以印证这一点：在它们intercept的内部都调用了chain.proceed()方法，且每次调用都在会创建一个RealInterceptorChain对象（当然最后一个拦截器CallServerInterceptor除外）！

这也是为什么RealInterceptorChain的procced方法中有如下的异常抛出，目的就是为了让拦截器链一个个执行下去，确保整个请求过程的完成：
```java
if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
}
```
也即是说自定义的拦截器Interceptor必须调Chain的proceed一次（可多次调用）