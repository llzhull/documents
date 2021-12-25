#### 

# 简介

```
An HTTP & HTTP/2 client for Android and Java applications. 
```

OkHttp是一个处理网络请求的高性能框架。

## 初识流程

<img src="https://upload-images.jianshu.io/upload_images/5982616-c9b805378329b087.png?imageMogr2/auto-orient/strip|imageView2/2/w/1000/format/webp" alt="img" style="zoom: 67%;" />

## okhttp的子系统层级结构

![img](https://upload-images.jianshu.io/upload_images/5982616-a44688d678695fd0.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



- **网络配置层**：利用Builder模式配置各种参数，例如：超时时间、拦截器等，这些参数都会由Okhttp分发给各个需要的子系统。
- **重定向层(RetryAndFollowUpInterceptor)**：负责重定向。
- **Header拼接层(BridgeInterceptor)：**负责把用户构造的请求转换为发送给服务器的请求，把服务器返回的响应转换为对用户友好的响应。
- **HTTP缓存层(CacheInterceptor)**：负责读取缓存以及更新缓存。
- **连接层(ConnectInterceptor)**：连接层是一个比较复杂的层级，它实现了网络协议、内部的拦截器、安全性认证，连接与连接池等功能，但这一层还没有发起真正的连接，它只是做了连接器一些参数的处理。
- **数据响应层(CallServerInterceptor)**：负责从服务器读取响应的数据。

## 发起请求

### 同步

```java
  public static void main(String... args)
	OkHttpClient client = new OkHttpClient();

    // Create request for remote resource.
    Request request = new Request.Builder()
        .url(ENDPOINT)
        .build();

    // Execute the request and retrieve the response.
    try (Response response = client.newCall(request).execute()) {
      // Deserialize HTTP response to concrete type.
      ResponseBody body = response.body();
      }
   }

//OkHttpClent.java
  @Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }


//RealCall.java
    @Override
    public Response execute() throws IOException {
        synchronized (this) {
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

```java
执行步骤：client.newCall(request).execute() ---> RealCall.execute() ---> Dispatcher.executed()
    --> RealCall.getResponseWithInterceptorChain()
```



### 异步

```java
public final class AsynchronousGet {
  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://publicobject.com/helloworld.txt")
        .build();

    client.newCall(request).enqueue(new Callback() {
      @Override public void onFailure(Call call, IOException e) {
        e.printStackTrace();
      }

      @Override public void onResponse(Call call, Response response) throws IOException {
        try (ResponseBody responseBody = response.body()) {
          if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

          Headers responseHeaders = response.headers();
          for (int i = 0, size = responseHeaders.size(); i < size; i++) {
            System.out.println(responseHeaders.name(i) + ": " + responseHeaders.value(i));
          }

          System.out.println(responseBody.string());
        }
      }
    });
  }

  public static void main(String... args) throws Exception {
    new AsynchronousGet().run();
  }
}
```

#### RealCall.java

```java
  @Override 
  public Response execute() throws IOException {
    synchronized (this) {
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



// AsyncCall
    final class AsyncCall extends NamedRunnable {
        private final Callback responseCallback;
        private volatile AtomicInteger callsPerHost = new AtomicInteger(0);

        AsyncCall(Callback responseCallback) {
            super("OkHttp %s", redactedUrl());
            this.responseCallback = responseCallback;
        }

        AtomicInteger callsPerHost() {
            return callsPerHost;
        }

        void reuseCallsPerHostFrom(AsyncCall other) {
            this.callsPerHost = other.callsPerHost;
        }

        String host() {
            return originalRequest.url().host();
        }

        Request request() {
            return originalRequest;
        }

        RealCall get() {
            return RealCall.this;
        }

        /**
         * Attempt to enqueue this async call on {@code executorService}. This will attempt to clean up
         * if the executor has been shut down by reporting the call as failed.
         */
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

        @Override
        protected void execute() {
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
    }

```



#### Dispatcher

```java

public final class Dispatcher {
  private int maxRequests = 64;
  private int maxRequestsPerHost = 5;
  private @Nullable Runnable idleCallback;

  /** Executes calls. Created lazily. */
  private @Nullable ExecutorService executorService;

  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

  public Dispatcher(ExecutorService executorService) {
    this.executorService = executorService;
  }

  public Dispatcher() {
  }

  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
    //..........

  void enqueue(AsyncCall call) {
    synchronized (this) {
      readyAsyncCalls.add(call);

      // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
      // the same host.
      if (!call.get().forWebSocket) {
        AsyncCall existingCall = findExistingCallWithHost(call.host());
        if (existingCall != null) call.reuseCallsPerHostFrom(existingCall);
      }
    }
    promoteAndExecute();
  }

  @Nullable private AsyncCall findExistingCallWithHost(String host) {
    for (AsyncCall existingCall : runningAsyncCalls) {
      if (existingCall.host().equals(host)) return existingCall;
    }
    for (AsyncCall existingCall : readyAsyncCalls) {
      if (existingCall.host().equals(host)) return existingCall;
    }
    return null;
  }

  /**
   * Cancel all calls currently enqueued or executing. Includes calls executed both {@linkplain
   * Call#execute() synchronously} and {@linkplain Call#enqueue asynchronously}.
   */
  public synchronized void cancelAll() {
    for (AsyncCall call : readyAsyncCalls) {
      call.get().cancel();
    }

    for (AsyncCall call : runningAsyncCalls) {
      call.get().cancel();
    }

    for (RealCall call : runningSyncCalls) {
      call.cancel();
    }
  }

  //从readyAsyncCalls 队列中取任务
  private boolean promoteAndExecute() {
    assert (!Thread.holdsLock(this));

    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall asyncCall = i.next();

          //正在请求的数量是有限的，默认64,用一个域名正在请求的数量也是有限的，默认5
        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
        if (asyncCall.callsPerHost().get() >= maxRequestsPerHost) continue; // Host max capacity.

        i.remove();
        asyncCall.callsPerHost().incrementAndGet();
        executableCalls.add(asyncCall);
          //放入Running 列表
        runningAsyncCalls.add(asyncCall);
      }
      isRunning = runningCallsCount() > 0;
    }

    for (int i = 0, size = executableCalls.size(); i < size; i++) {
      AsyncCall asyncCall = executableCalls.get(i);
        //使用线程池，执行异步请求任务
      asyncCall.executeOn(executorService());
    }

    return isRunning;
  }

  /** Used by {@code Call#execute} to signal it is in-flight. */
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }

  /** Used by {@code AsyncCall#run} to signal completion. */
  void finished(AsyncCall call) {
    call.callsPerHost().decrementAndGet();
    finished(runningAsyncCalls, call);
  }

  /** Used by {@code Call#execute} to signal completion. */
  void finished(RealCall call) {
    finished(runningSyncCalls, call);
  }

  private <T> void finished(Deque<T> calls, T call) {
    Runnable idleCallback;
    synchronized (this) {
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      idleCallback = this.idleCallback;
    }

    boolean isRunning = promoteAndExecute();

    if (!isRunning && idleCallback != null) {
      idleCallback.run();
    }
  }

  /** Returns a snapshot of the calls currently awaiting execution. */
  public synchronized List<Call> queuedCalls() {
    List<Call> result = new ArrayList<>();
    for (AsyncCall asyncCall : readyAsyncCalls) {
      result.add(asyncCall.get());
    }
    return Collections.unmodifiableList(result);
  }

  /** Returns a snapshot of the calls currently being executed. */
  public synchronized List<Call> runningCalls() {
    List<Call> result = new ArrayList<>();
    result.addAll(runningSyncCalls);
    for (AsyncCall asyncCall : runningAsyncCalls) {
      result.add(asyncCall.get());
    }
    return Collections.unmodifiableList(result);
  }

  public synchronized int queuedCallsCount() {
    return readyAsyncCalls.size();
  }

  public synchronized int runningCallsCount() {
    return runningAsyncCalls.size() + runningSyncCalls.size();
  }
}

```

```java
执行步骤：client.newCall(request).enqueue() ---> RealCall.enqueue(new AsyncCall(responseCallback))
    ---> Dispatcher.enqueue() 将AsyncCall 加入到 readyAsyncCalls
    ---> Dispatcher.promoteAndExecute() 遍历readyAsyncCalls，如果当前请求数<64 && 同一域名主机的请求数<5,则放入runningAsyncCalls 
    ---> AsyncCall.executeOn() 通过线程池执行该任务
    ---> AsyncCall.execute()
    --> RealCall.getResponseWithInterceptorChain()
```



## 拦截器概述

### 拦截器都有哪些？

```java
    //RealCall.java
	Response getResponseWithInterceptorChain() throws IOException {
        // Build a full stack of interceptors.
        List<Interceptor> interceptors = new ArrayList<>();
        interceptors.addAll(client.interceptors());
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



所有的拦截器（包括我们自定义的）都实现了Interceptor接口

```java
//Interceptor.java
public interface Interceptor {
  Response intercept(Chain chain) throws IOException;

  interface Chain {
    Request request();

    Response proceed(Request request) throws IOException;

    /**
     * Returns the connection the request will be executed on. This is only available in the chains
     * of network interceptors; for application interceptors this is always null.
     */
    @Nullable Connection connection();

    Call call();

    int connectTimeoutMillis();

    Chain withConnectTimeout(int timeout, TimeUnit unit);

    int readTimeoutMillis();

    Chain withReadTimeout(int timeout, TimeUnit unit);

    int writeTimeoutMillis();

    Chain withWriteTimeout(int timeout, TimeUnit unit);
  }
}
```

**Okhttp内置的拦截器**

- RetryAndFollowUpInterceptor：负责失败重试以及重定向。
- BridgeInterceptor：负责把用户构造的请求转换为发送给服务器的请求，把服务器返回的响应转换为对用户友好的响应。
- CacheInterceptor：负责读取缓存以及更新缓存。
- ConnectInterceptor：负责与服务器建立连接。
- CallServerInterceptor：负责从服务器读取响应的数据。



### RetryAndFollowUpInterceptor

#### 概述

RetryAndFollowUpInterceptor负责失败重试以及重定向。
RetryAndFollowUpInterceptor的作用就是处理了一些连接异常以及重定向。

**整个方法的流程**

- 构建一个StreamAllocation对象，StreamAllocation相当于是个管理类，维护了Connections、Streams和Calls之间的管理，该类初始化一个Socket连接对象，获取输入/输出流对象。
- 继续执行下一个Interceptor，即BridgeInterceptor
- 抛出异常，则检测连接是否还可以继续，**以下情况不会重试：**
   客户端配置出错不再重试
   出错后，request body不能再次发送
   发生以下Exception也无法恢复连接
   ProtocolException：协议异常
   InterruptedIOException：中断异常
   SSLHandshakeException：SSL握手异常
   SSLPeerUnverifiedException：SSL握手未授权异常
   没有更多线路可以选择。
- 根据响应码处理请求，返回Request不为空时则进行重定向处理，重定向的次数不能超过20次。
- 由followUpRequest()方法完成

#### 代码

```java
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        RealInterceptorChain realChain = (RealInterceptorChain) chain;
        Call call = realChain.call();
        EventListener eventListener = realChain.eventListener();
        //1. 构建一个StreamAllocation对象，StreamAllocation相当于是个管理类，维护了
        //Connections、Streams和Calls之间的管理，该类初始化一个Socket连接对象，获取输入/输出流对象。
        StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
                createAddress(request.url()), call, eventListener, callStackTrace);
        this.streamAllocation = streamAllocation;
        //重定向次数
        int followUpCount = 0;
        Response priorResponse = null;
        while (true) {
            if (canceled) {
                //删除连接上的call请求
                streamAllocation.release();
                throw new IOException("Canceled");
            }

            Response response;
            boolean releaseConnection = true;
            try {
                //2. 继续执行下一个Interceptor，即BridgeInterceptor
                response = realChain.proceed(request, streamAllocation, null, null);
                releaseConnection = false;
            } catch (RouteException e) {
                //抛出异常以指示通过单个路由连接的问题。
                // The attempt to connect via a route failed. The request will not have been sent.
                //尝试通过路由进行连接失败。 该请求不会被发送
                if (!recover(e.getLastConnectException(), streamAllocation, false, request)) {
                    throw e.getLastConnectException();
                }
                releaseConnection = false;
                continue;
            } catch (IOException e) {
                // An attempt to communicate with a server failed. The request may have been sent.
                //尝试与服务器通信失败。 该请求可能已发送
                boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
                if (!recover(e, streamAllocation, requestSendStarted, request)) throw e;
                releaseConnection = false;
                continue;
            } finally {
                // We're throwing an unchecked exception. Release any resources.
                // 检测到其他未知异常，则释放连接和资源
                if (releaseConnection) {
                    streamAllocation.streamFailed(null);
                    streamAllocation.release();
                }
            }

            // Attach the prior response if it exists. Such responses never have a body.
            //构建响应体，这个响应体的body为空。
            if (priorResponse != null) {
                response = response.newBuilder()
                        .priorResponse(priorResponse.newBuilder()
                                .body(null)
                                .build())
                        .build();
            }

            Request followUp;
            try {
                // 根据响应码处理请求，返回Request不为空时则进行重定向处理
                followUp = followUpRequest(response, streamAllocation.route());
            } catch (IOException e) {
                streamAllocation.release();
                throw e;
            }

            if (followUp == null) {
                if (!forWebSocket) {
                    streamAllocation.release();
                }
                return response;
            }

            closeQuietly(response.body());

            //重定向的次数不能超过20次
            if (++followUpCount > MAX_FOLLOW_UPS) {
                streamAllocation.release();
                throw new ProtocolException("Too many follow-up requests: " + followUpCount);
            }

            if (followUp.body() instanceof UnrepeatableRequestBody) {
                streamAllocation.release();
                throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
            }

            if (!sameConnection(response, followUp.url())) {
                streamAllocation.release();
                streamAllocation = new StreamAllocation(client.connectionPool(),
                        createAddress(followUp.url()), call, eventListener, callStackTrace);
                this.streamAllocation = streamAllocation;
            } else if (streamAllocation.codec() != null) {
                throw new IllegalStateException("Closing the body of " + response
                        + " didn't close its backing stream. Bad interceptor?");
            }

            request = followUp;
            priorResponse = response;
        }
    }
```



### BridgeInterceptor

#### 概述

BridgeInterceptor就跟它的名字那样，它是一个连接桥，它负责把用户构造的请求转换为发送给服务器的请求，把服务器返回的响应转换为对用户友好的响应。

转换的过程就是添加一些服务端需要的header信息。

在Request阶段配置用户信息，并添加一些请求头。在Response阶段，进行gzip解压。

#### 代码

```java
public final class BridgeInterceptor implements Interceptor {
  private final CookieJar cookieJar;

  public BridgeInterceptor(CookieJar cookieJar) {
    this.cookieJar = cookieJar;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

    RequestBody body = userRequest.body();
    if (body != null) {
      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }

    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }

    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    //如果我们添加一个“Accept-Encoding：gzip”头字段，我们也负责解压缩传输流。 
    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    //创建OkhttpClient配置的cookieJar
    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    //判断服务器是否支持gzip压缩，如果支持，则将压缩提交给Okio库来处理
    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      String contentType = networkResponse.header("Content-Type");
      responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
    }

    return responseBuilder.build();
  }

}

```



### CacheInterceptor

#### 概述

Okhttp是有自己的一套缓存机制的，CacheInterceptor就是用来负责读取缓存以及更新缓存的。
提供来自缓存的请求并将响应写入缓存

**整个方法的流程如下所示：**

1.读取候选缓存，具体如何读取的我们下面会讲

2.创建缓存策略，强制缓存、对比缓存等，关于缓存策略我们下面也会讲。

3.根据策略，不使用网络，又没有缓存的直接报错，并返回错误码504。

4.根据策略，不使用网络，有缓存的直接返回。

5.前面两个都没有返回，继续执行下一个Interceptor，即ConnectInterceptor。

6.接收到网络结果，如果响应code式304，则使用缓存，返回缓存结果。

7.读取网络结果。

8.对数据进行缓存。

9.返回网络读取的结果。



#### 代码

```java
public final class CacheInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        //1. 读取候选缓存
        Response cacheCandidate = cache != null
                ? cache.get(chain.request())
                : null;

        long now = System.currentTimeMillis();
        //2. 创建缓存策略，强制缓存、对比缓存等
        CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
        Request networkRequest = strategy.networkRequest;
        Response cacheResponse = strategy.cacheResponse;

        if (cache != null) {
            cache.trackResponse(strategy);
        }

        if (cacheCandidate != null && cacheResponse == null) {
            closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
        }

        // If we're forbidden from using the network and the cache is insufficient, fail.
        //3. 根据策略，不使用网络，又没有缓存的直接报错，并返回错误码504。
        if (networkRequest == null && cacheResponse == null) {
            return new Response.Builder()
                    .request(chain.request())
                    .protocol(Protocol.HTTP_1_1)
                    .code(504)
                    .message("Unsatisfiable Request (only-if-cached)")
                    .body(Util.EMPTY_RESPONSE)
                    .sentRequestAtMillis(-1L)
                    .receivedResponseAtMillis(System.currentTimeMillis())
                    .build();
        }

        // If we don't need the network, we're done.
        //4. 根据策略，不使用网络，有缓存的直接返回。
        if (networkRequest == null) {
            return cacheResponse.newBuilder()
                    .cacheResponse(stripBody(cacheResponse))
                    .build();
        }

        Response networkResponse = null;
        try {
            networkResponse = chain.proceed(networkRequest);
        } finally {
            // If we're crashing on I/O or otherwise, don't leak the cache body.
            // 5. 前面两个都没有返回，继续执行下一个Interceptor，即ConnectInterceptor。
            if (networkResponse == null && cacheCandidate != null) {
                closeQuietly(cacheCandidate.body());
            }
        }

        // If we have a cache response too, then we're doing a conditional get.
        //6. 接收到网络结果，如果响应code式304，则使用缓存，返回缓存结果。
        if (cacheResponse != null) {
            if (networkResponse.code() == HTTP_NOT_MODIFIED) {
                Response response = cacheResponse.newBuilder()
                        .headers(combine(cacheResponse.headers(), networkResponse.headers()))
                        .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
                        .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
                        .cacheResponse(stripBody(cacheResponse))
                        .networkResponse(stripBody(networkResponse))
                        .build();
                networkResponse.body().close();

                // Update the cache after combining headers but before stripping the
                // Content-Encoding header (as performed by initContentStream()).
                cache.trackConditionalCacheHit();
                cache.update(cacheResponse, response);
                return response;
            } else {
                closeQuietly(cacheResponse.body());
            }
        }
        //7. 读取网络结果。
        Response response = networkResponse.newBuilder()
                .cacheResponse(stripBody(cacheResponse))
                .networkResponse(stripBody(networkResponse))
                .build();
        //8. 对数据进行缓存。
        if (cache != null) {
            if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
                // Offer this request to the cache.
                CacheRequest cacheRequest = cache.put(response);
                return cacheWritingResponse(cacheRequest, response);
            }
            //9. 返回网络读取的结果。
            if (HttpMethod.invalidatesCache(networkRequest.method())) {
                try {
                    cache.remove(networkRequest);
                } catch (IOException ignored) {
                    // The cache cannot be written.
                }
            }
        }

        return response;
    }
}
```

#### 缓存策略

HTTP的缓存机制也是依赖于请求和响应header里的参数类实现的，最终响应式从缓存中去，还是从服务端重新拉取，HTTP的缓存机制的流程如下所示：

<img src="https://upload-images.jianshu.io/upload_images/5982616-805c2fda22ddf1c1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp" alt="img" style="zoom:80%;" />



##### **Http的缓存分为两种：**

- 强制缓存：需要服务端参与判断是否继续使用缓存，当客户端第一次请求数据是，服务端返回了缓存的过期时间（Expires与Cache-Control），没有过期就可以继续使用缓存，否则则不适用，无需再向服务端询问。
- 对比缓存：需要服务端参与判断是否继续使用缓存，当客户端第一次请求数据时，服务端会将缓存标识（Last-Modified/If-Modified-Since与Etag/If-None-Match）与数据一起返回给客户端，客户端将两者都备份到缓存中 ，再次请求数据时，客户端将上次备份的缓存
   标识发送给服务端，服务端根据缓存标识进行判断，如果返回304，则表示通知客户端可以继续使用缓存。
- 强制缓存优先于对比缓存。

###### 强制缓存

强制缓存使用的的两个标识：

- Expires：Expires的值为服务端返回的到期时间，即下一次请求时，请求时间小于服务端返回的到期时间，直接使用缓存数据。到期时间是服务端生成的，客户端和服务端的时间可能有误差。
- Cache-Control：Expires有个时间校验的问题，所有HTTP1.1采用Cache-Control替代Expires。

###### 对比缓存

对比缓存的两个标识：

-  时间戳标记资源是否修改

  Last-Modified 表示资源上次修改的时间。

  服务端接收到客户端发来的资源修改时间，与自己当前的资源修改时间进行对比，如果自己的资源修改时间大于客户端发来的资源修改时间，则说明资源做过修改， 则返回200表示需要重新请求资源，否则返回304表示资源没有被修改，可以继续使用缓存。

  

- 资源标识码ETag的方式来标记是否修改

  如果标识码发生改变，则说明资源已经被修改，ETag优先级高于Last-Modified。

  服务端接收到客户端发来的资源标识码，则会与自己当前的资源吗进行比较，如果不同，则说明资源已经被修改，则返回200，如果相同则说明资源没有被修改，返回 304，客户端可以继续使用缓存。

##### 具体实现类CacheStrategy



### ConnectInterceptor

#### 概述

在RetryAndFollowUpInterceptor里初始化了一个StreamAllocation对象，我们说在这个StreamAllocation对象里初始化了一个Socket对象用来做连接，但是并没有真正的连接，等到处理完hader和缓存信息之后，才调用ConnectInterceptor来进行**真正的连接**。

```java
public final class ConnectInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        RealInterceptorChain realChain = (RealInterceptorChain) chain;
        Request request = realChain.request();
        StreamAllocation streamAllocation = realChain.streamAllocation();

        // We need the network to satisfy this request. Possibly for validating a conditional GET.
        //我们需要网络来满足这个要求。 可能用于验证条件GET
        boolean doExtensiveHealthChecks = !request.method().equals("GET");
        //创建输出流
        HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
        //建立连接
        RealConnection connection = streamAllocation.connection();

        return realChain.proceed(request, streamAllocation, httpCodec, connection);
    }
}
```

ConnectInterceptor在Request阶段建立连接，处理方式也很简单，创建了两个对象：

HttpCodec：用来编码HTTP requests和解码HTTP responses

RealConnection：连接对象，负责发起与服务器的连接。

ConnectionPool: 连接池，实现连接的复用；

这里事实上包含了连接、连接池等一整套的Okhttp的连接机制

StreamAllocation、ConnectionPool、RealConnection

```java
streamAllocation.newStream()---> 
    findHealthyConnection()--->
      findConnection()---->
    	1、从ConnectionPool 中找到一个可用的连接
    		// Attempt to get a connection from the pool.
        	Internal.instance.acquire(connectionPool, address, this, null);
    	2、如果找不到，这创建一个RealConnection，进行三次握手并创建连接，并放入到连接池
 			RealConnection  result = new RealConnection(connectionPool, selectedRoute);
        	acquire(result, false);
			// Do TCP + TLS handshakes. This is a blocking operation.
    		result.connect()
            // Pool the connection.
      		Internal.instance.put(connectionPool, result);

```

#### StreamAllocation

```java
  public HttpCodec newStream(
      OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    
    try {
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
      HttpCodec resultCodec = resultConnection.newCodec(client, chain, this);

      synchronized (connectionPool) {
        codec = resultCodec;
        return resultCodec;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }

//查找连接并在它健康时返回它。如果它不健康，则重复该过程，直到找到健康的连接。
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled,
      boolean doExtensiveHealthChecks) throws IOException {
    while (true) {
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          pingIntervalMillis, connectionRetryEnabled);

      // If this is a brand new connection, we can skip the extensive health checks.
      synchronized (connectionPool) {
        if (candidate.successCount == 0) {
          return candidate;
        }
      }

      // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
      // isn't, take it out of the pool and start again.
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        noNewStreams();
        continue;
      }

      return candidate;
    }
  }
//返回一个连接来承载一个新的流。如果存在，则优先选择现有连接，然后是连接池，最后建立新连接。
 private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
    boolean foundPooledConnection = false;
    RealConnection result = null;
    Route selectedRoute = null;
    Connection releasedConnection;
    Socket toClose;
    synchronized (connectionPool) {
     
      if (this.connection != null) {
        // We had an already-allocated connection and it's good.
        result = this.connection;
        releasedConnection = null;
      }
      if (!reportedAcquired) {
        // If the connection was never reported acquired, don't report it as released!
        releasedConnection = null;
      }

      if (result == null) {
        // Attempt to get a connection from the pool.
        Internal.instance.acquire(connectionPool, address, this, null);
        if (connection != null) {
          foundPooledConnection = true;
          result = connection;
        } else {
          selectedRoute = route;
        }
      }
    }
    closeQuietly(toClose);

    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
    }
    if (result != null) {
      // If we found an already-allocated or pooled connection, we're done.
      return result;
    }

    // If we need a route selection, make one. This is a blocking operation.
    boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      routeSelection = routeSelector.next();
    }

    synchronized (connectionPool) {
      if (canceled) throw new IOException("Canceled");

      if (newRouteSelection) {
        // Now that we have a set of IP addresses, make another attempt at getting a connection from
        // the pool. This could match due to connection coalescing.
        List<Route> routes = routeSelection.getAll();
        for (int i = 0, size = routes.size(); i < size; i++) {
          Route route = routes.get(i);
          Internal.instance.acquire(connectionPool, address, this, route);
          if (connection != null) {
            foundPooledConnection = true;
            result = connection;
            this.route = route;
            break;
          }
        }
      }

       //没有在连接池中找到可用连接
      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }

        // Create a connection and assign it to this allocation immediately. This makes it possible
        // for an asynchronous cancel() to interrupt the handshake we're about to do.
        //创建连接并立即将其分配给此分配。
        route = selectedRoute;
        refusedStreamCount = 0;
        result = new RealConnection(connectionPool, selectedRoute);
        acquire(result, false);
      }
    }

    // Do TCP + TLS handshakes. This is a blocking operation.
    result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
        connectionRetryEnabled, call, eventListener);
    routeDatabase().connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      reportedAcquired = true;

      // Pool the connection.
      Internal.instance.put(connectionPool, result);

      // If another multiplexed connection to the same address was created concurrently, then
      // release this connection and acquire that one.
      if (result.isMultiplexed()) {
        socket = Internal.instance.deduplicate(connectionPool, address, this);
        result = connection;
      }
    }

    return result;
  }

//分配一个连接 StreamAllocationReference 弱引用，引用计数法，这里相当于给connection的引用计数器加1
  public void acquire(RealConnection connection, boolean reportedAcquired) {
    assert (Thread.holdsLock(connectionPool));
    if (this.connection != null) throw new IllegalStateException();

    this.connection = connection;
    this.reportedAcquired = reportedAcquired;
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  }
```



#### ConnectionPool

连接池，实现连接的复用

```java

/**
* 管理 HTTP 和 HTTP2 连接的重用以减少网络延迟。共享同一 {@link 地址} 的 HTTP 请求可能共享一个 {@link 连接}。
*这个类实现了哪些连接保持打开以备将来使用的策略。
 */
public final class ConnectionPool {
  /**
   * 后台线程用于清理过期的连接。每个连接池最多运行一个线程。线程池执行器允许池本身被垃圾收集。
   */
  private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
      Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
      new SynchronousQueue<>(), Util.threadFactory("OkHttp ConnectionPool", true));

  /** The maximum number of idle connections for each address. */
  private final int maxIdleConnections;
  private final long keepAliveDurationNs;
  private final Runnable cleanupRunnable = () -> {
    while (true) {
      long waitNanos = cleanup(System.nanoTime());
      if (waitNanos == -1) return;
      if (waitNanos > 0) {
        long waitMillis = waitNanos / 1000000L;
        waitNanos -= (waitMillis * 1000000L);
        synchronized (ConnectionPool.this) {
          try {
            ConnectionPool.this.wait(waitMillis, (int) waitNanos);
          } catch (InterruptedException ignored) {
          }
        }
      }
    }
  };

  private final Deque<RealConnection> connections = new ArrayDeque<>();
  final RouteDatabase routeDatabase = new RouteDatabase();
  boolean cleanupRunning;

  /**
   * 使用适合单用户应用程序的调整参数创建新的连接池。
   目前这个池最多可容纳 5 个空闲连接，如果超过5分钟不被使用，则将被连接池移除
   */
  public ConnectionPool() {
    this(5, 5, TimeUnit.MINUTES);
  }

  public ConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
    this.maxIdleConnections = maxIdleConnections;
    this.keepAliveDurationNs = timeUnit.toNanos(keepAliveDuration);

    // Put a floor on the keep alive duration, otherwise cleanup will spin loop.
    if (keepAliveDuration <= 0) {
      throw new IllegalArgumentException("keepAliveDuration <= 0: " + keepAliveDuration);
    }
  }

  /** Returns the number of idle connections in the pool. */
  public synchronized int idleConnectionCount() {
    int total = 0;
    for (RealConnection connection : connections) {
      if (connection.allocations.isEmpty()) total++;
    }
    return total;
  }

  /** Returns total number of connections in the pool. */
  public synchronized int connectionCount() {
    return connections.size();
  }

  /**
   * Acquires a recycled connection to {@code address} for {@code streamAllocation}. If non-null
   * {@code route} is the resolved route for a connection.
   */
  void acquire(Address address, StreamAllocation streamAllocation, @Nullable Route route) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      if (connection.isEligible(address, route)) {
        streamAllocation.acquire(connection, true);
        return;
      }
    }
  }

    /**
    * 当添加一个connection时，会触发执行一次清理
    */
  void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      cleanupRunning = true;
      executor.execute(cleanupRunnable);
    }
    connections.add(connection);
  }

    
    /**
    * 清理未使用且超时连接
    * 1、如果还未超时，计算出超时时间
    * 2、如果所有连接都在使用中，下次添加任务的时候再执行清理
    */
  long cleanup(long now) {
    int inUseConnectionCount = 0;
    int idleConnectionCount = 0;
    RealConnection longestIdleConnection = null;
    long longestIdleDurationNs = Long.MIN_VALUE;

    // Find either a connection to evict, or the time that the next eviction is due.
    synchronized (this) {
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();

        // If the connection is in use, keep searching.
        if (pruneAndGetAllocationCount(connection, now) > 0) {
          inUseConnectionCount++;
          continue;
        }

        idleConnectionCount++;

        // If the connection is ready to be evicted, we're done.
        long idleDurationNs = now - connection.idleAtNanos;
        if (idleDurationNs > longestIdleDurationNs) {
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }

        //超过最大连接数，或者超时，则移除
      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) {
        // We've found a connection to evict. Remove it from the list, then close it below (outside
        // of the synchronized block).
        connections.remove(longestIdleConnection);
      } else if (idleConnectionCount > 0) {
        // A connection will be ready to evict soon.
        return keepAliveDurationNs - longestIdleDurationNs;
      } else if (inUseConnectionCount > 0) {
        // All connections are in use. It'll be at least the keep alive duration 'til we run again.
        return keepAliveDurationNs;
      } else {
        // No connections, idle or in use.
        cleanupRunning = false;
        return -1;
      }
    }

    closeQuietly(longestIdleConnection.socket());

    // Cleanup again immediately.
    return 0;
  }

  /**
   * Prunes any leaked allocations and then returns the number of remaining live allocations on
   * {@code connection}. Allocations are leaked if the connection is tracking them but the
   * application code has abandoned them. Leak detection is imprecise and relies on garbage
   * collection.
   */
  private int pruneAndGetAllocationCount(RealConnection connection, long now) {
    List<Reference<StreamAllocation>> references = connection.allocations;
    for (int i = 0; i < references.size(); ) {
      Reference<StreamAllocation> reference = references.get(i);

      if (reference.get() != null) {
        i++;
        continue;
      }

      // We've discovered a leaked allocation. This is an application bug.
      StreamAllocation.StreamAllocationReference streamAllocRef =
          (StreamAllocation.StreamAllocationReference) reference;
      String message = "A connection to " + connection.route().address().url()
          + " was leaked. Did you forget to close a response body?";
      Platform.get().logCloseableLeak(message, streamAllocRef.callStackTrace);

      references.remove(i);
      connection.noNewStreams = true;

      // If this was the last allocation, the connection is eligible for immediate eviction.
      if (references.isEmpty()) {
        connection.idleAtNanos = now - keepAliveDurationNs;
        return 0;
      }
    }

    return references.size();
  }
}

```



### CallServerInterceptor

#### 概述

这是链中的最后一个拦截器。 它会对服务器进行网络请求

我们通过ConnectInterceptor已经连接到服务器了，接下来我们就是写入请求数据以及读出返回数据了。整个流程：

写入请求头

写入请求体

读取响应头

读取响应体

#### 代码

```java
public final class CallServerInterceptor implements Interceptor 
    @Override
    public Response intercept(Chain chain) throws IOException {
        //这些对象在前面的Interceptor都已经创建完毕
        RealInterceptorChain realChain = (RealInterceptorChain) chain;
        HttpCodec httpCodec = realChain.httpStream();
        StreamAllocation streamAllocation = realChain.streamAllocation();
        RealConnection connection = (RealConnection) realChain.connection();
        Request request = realChain.request();

        long sentRequestMillis = System.currentTimeMillis();
        //1. 写入请求头
        realChain.eventListener().requestHeadersStart(realChain.call());
        httpCodec.writeRequestHeaders(request);
        realChain.eventListener().requestHeadersEnd(realChain.call(), request);

        Response.Builder responseBuilder = null;
        if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
            // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
            // Continue" response before transmitting the request body. If we don't get that, return
            // what we did get (such as a 4xx response) without ever transmitting the request body.
            //如果请求中存在“Expect：100-continue”标头，请在发送请求主体之前等待“HTTP / 1.1 100 Continue”响应。
            // 如果我们没有得到，请返回我们得到的内容（例如4xx响应），而不发送请求主体。
            if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
                httpCodec.flushRequest();
                realChain.eventListener().responseHeadersStart(realChain.call());
                responseBuilder = httpCodec.readResponseHeaders(true);
            }
            //2 写入请求体
            if (responseBuilder == null) {
                // Write the request body if the "Expect: 100-continue" expectation was met.
                realChain.eventListener().requestBodyStart(realChain.call());
                long contentLength = request.body().contentLength();
                CountingSink requestBodyOut =
                        new CountingSink(httpCodec.createRequestBody(request, contentLength));
                BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);

                request.body().writeTo(bufferedRequestBody);
                bufferedRequestBody.close();
                realChain.eventListener()
                        .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
            } else if (!connection.isMultiplexed()) {
                // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
                // from being reused. Otherwise we're still obligated to transmit the request body to
                // leave the connection in a consistent state.
                streamAllocation.noNewStreams();
            }
        }

        httpCodec.finishRequest();

        if (responseBuilder == null) {
            realChain.eventListener().responseHeadersStart(realChain.call());
            responseBuilder = httpCodec.readResponseHeaders(false);
        }

        Response response = responseBuilder
                .request(request)
                .handshake(streamAllocation.connection().handshake())
                .sentRequestAtMillis(sentRequestMillis)
                .receivedResponseAtMillis(System.currentTimeMillis())
                .build();
        //4 读取响应体
        int code = response.code();
        if (code == 100) {
            // server sent a 100-continue even though we did not request one.
            // try again to read the actual response
            responseBuilder = httpCodec.readResponseHeaders(false);

            response = responseBuilder
                    .request(request)
                    .handshake(streamAllocation.connection().handshake())
                    .sentRequestAtMillis(sentRequestMillis)
                    .receivedResponseAtMillis(System.currentTimeMillis())
                    .build();

            code = response.code();
        }

        realChain.eventListener()
                .responseHeadersEnd(realChain.call(), response);

        if (forWebSocket && code == 101) {
            // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
            response = response.newBuilder()
                    .body(Util.EMPTY_RESPONSE)
                    .build();
        } else {
            response = response.newBuilder()
                    .body(httpCodec.openResponseBody(response))
                    .build();
        }

        if ("close".equalsIgnoreCase(response.request().header("Connection"))
                || "close".equalsIgnoreCase(response.header("Connection"))) {
            streamAllocation.noNewStreams();
        }

        if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
            throw new ProtocolException(
                    "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
        }

        return response;
    }
}
```



### 应用拦截器和网络拦截器的区别

看一下拦截器执行顺序图



<img src="https://upload-images.jianshu.io/upload_images/5700431-da7df4bd19ff42be.png?imageMogr2/auto-orient/strip|imageView2/2/w/938/format/webp" alt="img" style="zoom:80%;" />

应用拦截器的作用

- 不需要关心是否重定向或者失败重连
- 应用拦截器只会调用一次，即使数据来源于缓存
- 可以执行多次调用其他拦截器，通过Chain.proceed().
- 自定义的应用拦截器是第一个开始执行的拦截器，所以这句话的意思就是，应用拦截器可以决定是否执行其他的拦截器，通过Chain.proceed().

网络拦截器的作用

- 网络拦截器可以操作重定向和失败重连的返回值
- 取缓存中的数据就不会去还行Chain.proceed().所以就不能执行网络拦截器
- 通过网络拦截器可以观察到所有通过网络传输的数据
- 请求服务连接的拦截器先于网络拦截器执行，所以网络拦截器可以看到Request中服务器请求连接信息，但是应用拦截器是获取不到对应的连接信息的。