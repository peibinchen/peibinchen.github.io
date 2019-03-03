---
title: Okhttp源码解析
date: 2017-02-24 10:25:05
tags:
    - 项目深化
---

OkHttp源码中我们要解决的问题有：
- 责任链模式怎么在OkHttp中应用的？责任链中的每一环具体在做什么？
- 同步请求与异步请求的区别以及怎么实现？


## 责任链模式怎么在OkHttp中应用的？责任链中的每一环具体在做什么？
以一个实际例子来跟踪责任链。
```java
public String run(String url)throws IOException{
    Request request = new Request.Builder().url(url).build();
    try(Response response = okHttpClient.newCall(request).execute()){
        return response.body().string();
    }
}
```
这个过程分为两步：
1. 构建Request
2. 发送请求


### 1. 构建Request
``` java
  Request request = new Request.Builder().url(url).build();
```

<div style="text-align: center">
<img src="http://p1.bqimg.com/567571/03643f8867c4321b.png"/>
</div>

这个过程分为三步：
- 构建Builder对象，初始化Builder中的heads和method参数
- 调用url函数，构建HttpUrl结构，初始化Builder中的url参数
- 调用builder函数，使用Builder构建Request对象



#### 1.1 构建Builder对象，初始化Builder中的heads和method参数
首先搞清楚Request.Builder中到底有哪些参数：

参数 | 类型
---|---
url | HttpUrl
method | String
headers | Headers.Builder
body | RequestBody
tag | Object

Builder构造函数：
``` java
public Builder() {
      this.method = "GET";
      this.headers = new Headers.Builder();
    }
```

注意：
- 这里的Headers.Builder()构造函数是空的，并且现在的这个Builder构造函数是Request.Builder的构造函数。也就是说，Request.Builder中初始化了一个Headers.Builder作为headers。
- Headers.Builder中只有一个属性且没有继承其他类：
```java
final List<String> namesAndValues = new ArrayList<>(20);
```
- 默认的method为GET

总结一下，通过【1.构建Builder对象，初始化Builder中的heads和method参数】步骤，有哪些参数已经完成了解析：
- method（默认是GET）
- headers（默认大小为20）

#### 1.2 调用url函数，构建HttpUrl结构，初始化Builder中的url参数
```java
/**
     * Sets the URL target of this request.
     *
     * @throws IllegalArgumentException if {@code url} is not a valid HTTP or HTTPS URL. Avoid this
     * exception by calling {@link HttpUrl#parse}; it returns null for invalid URLs.
     */
    public Builder url(String url) {
      if (url == null) throw new NullPointerException("url == null");

      // Silently replace web socket URLs with HTTP URLs.
      if (url.regionMatches(true, 0, "ws:", 0, 3)) {
        url = "http:" + url.substring(3);
      } else if (url.regionMatches(true, 0, "wss:", 0, 4)) {
        url = "https:" + url.substring(4);
      }

      HttpUrl parsed = HttpUrl.parse(url);
      if (parsed == null) throw new IllegalArgumentException("unexpected url: " + url);
      return url(parsed);
    }
```
url函数中做了三件事：
- 将Web Socket的URL转化为Http的URL
- 构建HttpUrl结构
- 用HttpUrl结构初始化Builder的url参数

##### 1.2.1 将Web Socket的URL转化为Http的URL
首先有两个问题，Web Socket是什么？它的URL长什么样子？
- Web Socket是什么？

Web Socket是一种在单个TCP连接上进行全双工通讯的协议，允许服务端主动向客户端推送数据。并且更神奇的是，Web Socket只需要一次握手，两者之间就可以直接建立起持久性的连接，并进行双向数据传输。
一次握手：浏览器发送请求，服务器做出回应，也就完成了握手过程。

- 它的URL长什么样子？

ws://example.com/wsapi
wss://secure.example.com/（基于TLS协议）
相当于Http协议中的http和https。

##### 1.2.2 构建HttpUrl结构
```java
public static HttpUrl parse(String url) {
    Builder builder = new Builder();
    Builder.ParseResult result = builder.parse(null, url);
    return result == Builder.ParseResult.SUCCESS ? builder.build() : null;
  }
```

包含三个步骤：
- 构造Builder对象
- 构造Builder.ParseResult对象
- 通过Builder构造HttpUrl对象

###### 1.2.2.1 构造Builder对象
```java
public Builder() {
      encodedPathSegments.add(""); // The default path is '/' which needs a trailing space.
    }
```
encodedPathSegements是一个ArrayList，用来记录整个url中的各个部分：
```java
final List<String> encodedPathSegments = new ArrayList<>();
```

###### 1.2.2.2 构造Builder.ParseResult对象
HttpUrl.Builder.ParseResult的结构为：
```java
enum ParseResult {
      SUCCESS,
      MISSING_SCHEME,
      UNSUPPORTED_SCHEME,
      INVALID_PORT,
      INVALID_HOST,
    }
```

构造ParseResult对象本质上是进一步填充第一步构造出来的Builder中的参数，
这个过程比较复杂，但是核心是做了以下几件事：
- 过滤url中的前后空格
- 设置Builder.scheme（http或者https）
- 解析Builder.encodedUsername和Builder.encodedPassword
- 解析Builder.host和Builder.port
- 以 / 作为分界点分割从host和port之后并且在 ？或者# 之前的url字符串，将分割结果储存在Builder.encodePathSegments
<div style="text-align: center">
<img src="http://p1.bqimg.com/567571/764f0f1a4fa0d4ee.png"/>
</div>
- 如果有 ？，那么解析后面的url，填充到encodedQueryNamesAndValues变量中（也是ArrayList），encodedQueryNamesAndValues中记录？后面的字符串是按照 key，value，key，value这种格式记录的。比如url = https://www.baidu.com/s?ie=utf-8&f=8&rsv_bp=0 ，解析出来的结构如下：
<div style="text-align: center">
<img src="http://i1.piimg.com/567571/1598472a6b7fd68f.png"/>
</div>

- 如果有 # ，那么解析出来的结果放在encodedFragment变量中（String类型）
- 解析完毕



###### 1.2.2.3 通过Builder构造HttpUrl对象
```java
public HttpUrl build() {
      if (scheme == null) throw new IllegalStateException("scheme == null");
      if (host == null) throw new IllegalStateException("host == null");
      return new HttpUrl(this);
    }
```
```java
HttpUrl(Builder builder) {
    this.scheme = builder.scheme;
    this.username = percentDecode(builder.encodedUsername, false);
    this.password = percentDecode(builder.encodedPassword, false);
    this.host = builder.host;
    this.port = builder.effectivePort();
    this.pathSegments = percentDecode(builder.encodedPathSegments, false);
    this.queryNamesAndValues = builder.encodedQueryNamesAndValues != null
        ? percentDecode(builder.encodedQueryNamesAndValues, true)
        : null;
    this.fragment = builder.encodedFragment != null
        ? percentDecode(builder.encodedFragment, false)
        : null;
    this.url = builder.toString();
  }
```

builder.toString()函数就是把分割出来的东西重新组装成一个String返回。
```java
@Override public String toString() {
      StringBuilder result = new StringBuilder();
      result.append(scheme);
      result.append("://");

      if (!encodedUsername.isEmpty() || !encodedPassword.isEmpty()) {
        result.append(encodedUsername);
        if (!encodedPassword.isEmpty()) {
          result.append(':');
          result.append(encodedPassword);
        }
        result.append('@');
      }

      if (host.indexOf(':') != -1) {
        // Host is an IPv6 address.
        result.append('[');
        result.append(host);
        result.append(']');
      } else {
        result.append(host);
      }

      int effectivePort = effectivePort();
      if (effectivePort != defaultPort(scheme)) {
        result.append(':');
        result.append(effectivePort);
      }

      pathSegmentsToString(result, encodedPathSegments);

      if (encodedQueryNamesAndValues != null) {
        result.append('?');
        namesAndValuesToQueryString(result, encodedQueryNamesAndValues);
      }

      if (encodedFragment != null) {
        result.append('#');
        result.append(encodedFragment);
      }

      return result.toString();
    }
```

##### 1.2.3 用HttpUrl结构初始化Builder的url参数
```java
//Request.Builder
public Builder url(HttpUrl url) {
      if (url == null) throw new NullPointerException("url == null");
      this.url = url;
      return this;
    }
```
总结一下，通过【2. 调用url函数，构建HttpUrl结构，初始化Builder中的url参数】这一步骤，
Builder中有哪些参数完成了解析：
- scheme（http或者https）
- username（如果有的话）
- password（如果有的话）
- host（主机名，也就是 www.xxx.com）
- port（如果有设定特定端口号的话，如果没有，http默认是80，https默认是443，否则为-1）
- pathSegements（从port之后，在？之前的字段）
- queryNamesAndValues（？之后，保存方式是key，value，key，value.....）
- fragment（#之后，解析出String）
- url（也就是原始的url字符串，这里是通过builder.toString()重新组装得到的）

#### 1.3 调用builder函数，使用Builder构建Request对象
``` java
public Request build() {
      if (url == null) throw new IllegalStateException("url == null");
      return new Request(this);
    }
```

``` java
Request(Builder builder) {
    this.url = builder.url;
    this.method = builder.method;
    this.headers = builder.headers.build();
    this.body = builder.body;
    this.tag = builder.tag != null ? builder.tag : this;
  }
```

总结一下，通过【1.3 调用builder函数，使用Builder构建Request对象】，将builder中的参数复制到Request中：
- url（在②中完成解析，也就是builder的HttpUrl结构）
- scheme（http或者https）
- username（如果有的话）
- password（如果有的话）
- host（主机名，也就是www.xxx.com）
- port（如果有设定特定端口号的话，如果没有，http默认是80，https默认是443，否则为-1）
- pathSegements（从port之后，在？之前的字段）
- queryNamesAndValues（？之后，保存方式是key，value，key，value。。。）
- fragment（#之后，解析出String）
- url（也就是原始的url字符串，这里是通过builder.toString()重新组装得到的）
- method（在①中默认是GET）
- headers（在①中默认大小为20，尚未添加数据）
- body（在①中尚未初始化）
- tag（在①中尚未初始化）


再搞清楚内部的逻辑关系:  
Request通过Request.Builder构造，而Request.Builder内部需要一个HttpUrl结构。  
HttpUrl结构内部也是通过Builder设计模式，采用构造HttpUrl.Builder结构，在HttpUrl.Builder中解析出了众多参数，再通过build函数将这些参数赋予HttpUrl结构。  
Request.Builder中的HttpUrl结构构造完毕后，就设置Request.Builder.url = url（HttpUrl），最终通过build函数获得最终的Request结构。

### 2. 发送请求
```java
Response response = okHttpClient.newCall(request).execute()
```
这里主要分为两步：  
- 构造Call对象  
- 执行Request

#### 2.1 构造Call对象
okHttpClient.newCall(request)调用OkHttpClient.newCall函数：
```java
@Override public Call newCall(Request request) {
    return new RealCall(this, request, false /* for web socket */);
  }
```

本质上得到的是RealCall结构：
```java
RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
  }
```

RealCall数据结构：

变量 | 类型
---|---
client | final OkHttpClient
retryAndFollowUpInterceptor | final RetryAndFollowUpInterceptor
originalRequest | final Request
forWebSocket | final boolean
executed | boolean

注意到RealCall构造函数中初始化了一个拦截器：retryAndFollowUpInterceptor拦截器。
```java
public RetryAndFollowUpInterceptor(OkHttpClient client, boolean forWebSocket) {
    this.client = client;
    this.forWebSocket = forWebSocket;
  }
  ```

所以，在2.1中，构造了一个RealCall数据结构，并且在RealCall内部记录了调用的okHttpClient以及request，并且初始化了RetryAndFollowUpInterceptor拦截器（记录了okHttpClient以及forWebSocket参数）


#### 2.2 发送请求
发送请求主要调用的是Call.execute函数，因为这里Call对象是RealCall类型，因此调用的是RealCall.execute函数。  
execute函数：
```java
@Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
      client.dispatcher().finished(this);
    }
  }
```

execute函数主要有三步
- 设置debug相关参数
- 将RealCall添加到Dispatcher调度器中
- 通过责任链消化request，返回结果
- 从OkHttpClient的Dispatcher中移除已经处理完毕的当前request

因为debug步骤我们不关心，所以只看后面三步。

##### 2.2.1 将RealCall添加到Dispatcher调度器中
```java
client.dispatcher().executed(this);//this是RealCall
```
内部封装了request，这里分为两步：
- 获得dispatcher
- 将realCall添加到dispatcher中的队列中

###### 2.2.1.1 获得dispatcher
```java
public Dispatcher dispatcher() {
    return dispatcher;
  }
```
那这个dispatcher是在哪里初始化设置的呢？
在OkHttpClient的构造函数中：
```java
this.dispatcher = builder.dispatcher;
```
而因为我们一开始创建OkHttpClient对象是通过OkHttpClient c = new OkHttpClient()创建的，因此调用的是以下的构造函数:
```
public OkHttpClient() {
    this(new Builder());
  }
```
也就是说dispatcher是在OkHttpClient.Builder的构造函数中初始化的。
在OkHttpClient.Builder构造函数中有这么一句：
```java
dispatcher = new Dispatcher();
```
可以看到OkHttpClient的dispatcher是Builder内部通过直接new构造出来的，而Dispatcher构造函数是一个空实现，内部什么也没有。
```java
public Dispatcher() {
  }
```
那么就来看Dispatcher内部有什么属性？
```java
  private int maxRequests = 64;
  private int maxRequestsPerHost = 5;
  private Runnable idleCallback;

  /** Executes calls. Created lazily. */
  private ExecutorService executorService;

  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
  ```
从上面可以得出以下几点信息：  
I 一个Dispatcher最多能够处理64个请求，并且每个域名最多只能有5个请求  
II 内部有三个Deque队列，两个是为异步请求设置的（readyAsyncCalls，runningAsyncCalls），一       		个是为同步请求设置的（runningSyncCalls）

所以【2.2.1.1 获得dispatcher】步骤实际上是获得一个OkHttpClient的dispatcher，而OkHttpClient的dispatcher是通过Builder内部new一个出来的。

##### 2.2.2.2 将realCall添加到dispatcher中的队列中
```java
synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }
```
因为在a中Dispatcher已经将runningSyncCalls初始化了，所以这里直接将call添加到runningSyncCalls队列中。  
从这里可以看出，通过应用层的execute函数执行的请求实际上是以同步方式执行的。

##### 2.2.2.3 通过责任链消化request，返回结果
```java
Response result = getResponseWithInterceptorChain();//这个是整个调用过程的核心
```

RealCall.getResponseWithInterceptorChain()函数：
```java
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

    Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
    return chain.proceed(originalRequest);
  }
  ```
 整个责任链模式在这个函数中得到充分体现：
- 构造List数组interceptors
- 添加okHttpClient的所有已有的拦截器（默认为size = 0）
- 添加retryAndFollowUpInterceptor
- 添加BridgeInterceptor
- 添加CacheInterceptor
- 添加ConnectInterceptor
- 如果不是web socket，那么添加okHttpClient的所有networkInterceptors（用户自定义的Interceptors，默认为size = 0）
- 添加CallServerInterceptor
- 构建责任链RealInterceptorChain
- 将原始的请求originalRequest交给责任链消化

这里面有2个问题亟待解决：
- 每个拦截器的作用是什么？
- 责任链是怎么将请求逐步分发给责任链中的每一环的？

##### 2.2.2.3.1 每个拦截器的作用是什么？
<div style="text-align: center">
<img src="http://p1.bpimg.com/567571/548e46f32afed5ae.png"/>
</div>

##### 2.2.2.3.2 责任链是怎么将请求逐步分发给责任链中的每一环的？
通过interceptors构建整条责任链上的拦截器后，通过两个步骤来分发request。  
I、构造责任链  
II、分发request

I、构造责任链  
```java
Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
```

```java
public RealInterceptorChain(List<Interceptor> interceptors, StreamAllocation streamAllocation,
      HttpCodec httpCodec, Connection connection, int index, Request request) {
    this.interceptors = interceptors;
    this.connection = connection;
    this.streamAllocation = streamAllocation;
    this.httpCodec = httpCodec;
    this.index = index;
    this.request = request;
  }
```

II、分发request
```java
return chain.proceed(originalRequest);
```

```java
 @Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
  }
  ```
注意到这里的streamAllocation，httpCodec，connection都为null。
```java
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      Connection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpCodec != null && !sameConnection(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpCodec, connection, index + 1, request);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    return response;
  }
```

proceed函数主要做了以下5件事：
- 检查上一个拦截器传递过来的httpCodec是否为null，如果不为null，那么当前的RealInterceptorChain的HttpUrl中的host和port必须和当前的request一致，否则说明上一个拦截器修改了host和port，出错处理
- 确认当前的RealInterceptorChain的proceed函数只被调用一次，否则出错处理
- 构造下一条RealInterceptorChain链
- 获得当前的拦截器，调用该拦截器的interceptor函数
- 返回response


核心的过程是第四步，因此着重分析第四步。
Interceptor interceptor = interceptors.get(index);
Response response = interceptor.intercept(next);

I：RetryAndFollowUpInterceptor拦截器  
一开始的index为0，所以获得的是RetryAndFollowUpInterceptor，因此调用的是RetryAndFollowUpInterceptor中的interceptor函数。
```java
@Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();

    streamAllocation = new StreamAllocation(
        client.connectionPool(), createAddress(request.url()), callStackTrace);

    int followUpCount = 0;
    Response priorResponse = null;
    while (true) {
      if (canceled) {
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response = null;
      boolean releaseConnection = true;
      try {
        response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        // The attempt to connect via a route failed. The request will not have been sent.
        if (!recover(e.getLastConnectException(), false, request)) {
          throw e.getLastConnectException();
        }
        releaseConnection = false;
        continue;
      } catch (IOException e) {
        // An attempt to communicate with a server failed. The request may have been sent.
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, requestSendStarted, request)) throw e;
        releaseConnection = false;
        continue;
      } finally {
        // We're throwing an unchecked exception. Release any resources.
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
        }
      }

      // Attach the prior response if it exists. Such responses never have a body.
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }

      Request followUp = followUpRequest(response);

      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        return response;
      }

      closeQuietly(response.body());

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
        streamAllocation = new StreamAllocation(
            client.connectionPool(), createAddress(followUp.url()), callStackTrace);
      } else if (streamAllocation.codec() != null) {
        throw new IllegalStateException("Closing the body of " + response
            + " didn't close its backing stream. Bad interceptor?");
      }

      request = followUp;
      priorResponse = response;
    }
  }
```

RetryAndFollowUpInterceptor中的interceptor函数主要做了以下几件事：  
- 获得request
- 构造StreamAllocation
- 进入while（true）循环
- 调用下一个RealInterceptorChain的proceed函数（也就是一开始传入来的next，其实就是index发生改变）
- response返回后根据response决定是否要进行重定向或者重传

**立flag：RetryAndFollowUpInterceptor的重定向和重传技术等到后面整条链走完后再讲**。

II：BridgeInterceptor拦截器  
RetryAndFollowUpInterceptor的第四步就是调用下一个调用下一个RealInterceptorChain的proceed函数。proceed函数在前面讲过了，它会调用index的interceptor的intercept函数，在这里也就是BridgeInterceptor的intercept函数。

