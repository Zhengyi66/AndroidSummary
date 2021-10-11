### 一、用法

```java
OkHttpClient okHttpClient = new OkHttpClient.Builder()
        .build();
Request request = new Request.Builder()
        .url("")
        .build();
Call realCall = okHttpClient.newCall(request);
realCall.enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {

    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {

    }
});
Response response = realCall.execute();
```

RealCall的enqueue

```java
  @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    transmitter.callStart();
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```

AsyncCall实现Runnable接口，

```java
    @Override protected void execute() {
      boolean signalledCallback = false;
      transmitter.timeoutEnter();
      try {
        Response response = getResponseWithInterceptorChain();
        signalledCallback = true;
        responseCallback.onResponse(RealCall.this, response);
      } catch (IOException e) {
        if (signalledCallback) {
          ...
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
```

执行**getResponseWithInterceptorChain();** 拦截器责任链

### 二、拦截器责任链流程

#### 1、RetryAndFollowUpInterceptor

**重试以及重定向拦截器**：主要完成**重试**和**重定向**

1、重试

请求阶段发生了 RouteException 或者 IOException会进行判断是否重新发起请求。

2、重定向

重定向发生在followUpRequest方法中，

如果此方法返回空，那就表示不需要再重定向了，直接返回响应；但是如果返回非空，那就要重新请求返回的`Request`，但是需要注意的是，我们的`followup`在拦截器中定义的最大次数为**20**次。

**总结**
本拦截器是整个责任链中的第一个，这意味着它会是首次接触到**Request**与最后接收到**Response**的角色，在这个拦截器中主要功能就是判断是否需要**重试与重定向**。

- **重试**的前提是出现了`RouteException`或者`IOException`。一但在后续的拦截器执行过程中出现这两个异常，就会通过recover方法进行判断是否进行连接重试。

- **重定向**发生在重试的判定之后，如果不满足重试的条件，还需要进一步调用`followUpRequest`根据Response 的响应码(当然，如果直接请求失败，Response都不存在就会抛出异常)。followup最大发生20次。

#### 2、BridgeInterceptor

**桥接拦截器**：连接应用程序和服务器的桥梁，我们发出的请求将会经过它的处理才能发给服务器，比如设置请求内容长度，编码，gzip压缩，cookie等，获取响应后保存Cookie等操作。这个拦截器相对比较简单。就是**补全请求头**

**总结**
桥接拦截器的执行逻辑主要就是以下几点

- 对用户构建的Request进行添加或者删除相关头部信息，以转化成能够真正进行网络请求的Request
- 将符合网络请求规范的Request交给下一个拦截器处理，并获取Response
- 如果响应体经过了GZIP压缩，那就需要解压，再构建成用户可用的Response并返回

#### 3、CacheInterceptor

**缓存拦截器**：在发出请求前，判断是否命中缓存。如果命中则可以不请求，直接使用缓存的响应。 (只会存在Get请求的缓存)

步骤为:

1、从缓存中获得对应请求的响应缓存

2、创建`CacheStrategy` ,创建时会判断是否能够使用缓存，在`CacheStrategy` 中存在两个成员:`networkRequest`与`cacheResponse`

3、交给下一个责任链继续处理

4、后续工作，返回304则用缓存的响应；否则使用网络响应并缓存本次响应（只缓存Get请求的响应）

#### 4、ConnectInterceptor

**连接拦截器**：打开与目标服务器的连接，并执行下一个拦截器

```java
public final class ConnectInterceptor implements Interceptor {
  public final OkHttpClient client;

  public ConnectInterceptor(OkHttpClient client) {
    this.client = client;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
}

```

这个拦截器中的所有实现都是为了获得一份与目标服务器的连接，在这个连接上进行HTTP数据的收发

#### 5、CallServerInterceptor

**请求服务器拦截器**：利用`HttpCodec`发出请求到服务器并且解析生成`Response`

**总结**

在这个拦截器中就是完成HTTP协议报文的封装与解析。

### OkHttp总结

整个OkHttp功能的实现就在这五个默认的拦截器中，这五个拦截器分别为: 重试拦截器、桥接拦截器、缓存拦截器、连接拦截器、请求服务拦截器。

OkHttp中的拦截器每次发起请求都会在交给下一个拦截器之前干一些事情，在获得了结果之后又干一些事情。整个过程在请求向是顺序的，而响应向则是逆序。

当用户发起一个请求后，会由任务分发起Dispatcher将请求包装并交给重试拦截器处理。

1、重试拦截器在交出(交给下一个拦截器)之前，负责判断用户是否取消了请求；在获得了结果之后，会根据响应码判断是否需要重定向，如果满足条件那么就会重启执行所有拦截器。

2、桥接拦截器在交出之前，负责将HTTP协议必备的请求头加入其中(如：Host)并添加一些默认的行为(如：GZIP压缩)；在获得了结果后，调用保存cookie接口并解析GZIP数据。

3、缓存拦截器顾名思义，交出之前读取并判断是否使用缓存；获得结果后判断是否缓存。

4、连接拦截器在交出之前，负责找到或者新建一个连接，并获得对应的socket流；在获得结果后不进行额外的处理。

5、请求服务器拦截器进行真正的与服务器的通信，向服务器发送数据，解析读取的响应数据。

在经过了这一系列的流程后，就完成了一次HTTP请求！

参考：https://blog.csdn.net/my_csdnboke/article/details/103840783
