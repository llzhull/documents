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

