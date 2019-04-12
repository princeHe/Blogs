@[TOC](OkHttp源码阅读理解系列（一）)
# OkHttp

> HTTP是现代应用程序网络的方式。这是我们交换数据和媒体的方式。高效地执行HTTP可以使您的工作负载更快，并节省带宽。
> OkHttp是一个默认高效的HTTP客户端:
> - HTTP/2支持允许对同一主机的所有请求共享一个套接字。
> - 连接池减少了请求延迟(如果HTTP/2不可用)。
> - 透明GZIP压缩下载大小。
> - 响应缓存完全避免了网络重复请求。
> 
> 当网络出现问题时，OkHttp会坚持下去:它会从常见的连接问题中悄悄地恢复。如果您的服务有多个IP地址，如果第一个连接失败，OkHttp将尝试替代地址。这对于IPv4+IPv6和托管在冗余数据中心中的服务是必要的。OkHttp支持现代TLS特性(TLS
> 1.3、ALPN、证书固定)。可以将其配置为后退以实现广泛的连接。 使用OkHttp很容易。它的请求/响应API设计为流畅的构建器和不变性。它同时支持同步阻塞调用和带回调的异步调用。
> OkHttp支持Android 5.0+ (API级别21+)和Java 8+。

[官方文档](http://square.github.io/okhttp/)

## 使用示例
- GET：

```java
OkHttpClient client = new OkHttpClient();

String run(String url) throws IOException {
  Request request = new Request.Builder()
      .url(url)
      .build();

  try (Response response = client.newCall(request).execute()) {
    return response.body().string();
  }
}
```
- POST：

```java
public static final MediaType JSON = MediaType.get("application/json; charset=utf-8");

OkHttpClient client = new OkHttpClient();

String post(String url, String json) throws IOException {
  RequestBody body = RequestBody.create(JSON, json);
  Request request = new Request.Builder()
      .url(url)
      .post(body)
      .build();
  try (Response response = client.newCall(request).execute()) {
    return response.body().string();
  }
}
```

## 源码分析
**这里要说一下，OkHttp的源代码大部分已经用kotlin重写了，如果还不熟悉kotlin语法的同学要赶紧去学习一下啦。**
我们就从这两个同步请求的入口来分析一下，首先初始化OkHttpClient()，内部的成员变量等用到的时候再分析。接着通过RequestBuilder去构建一个Request，再接着通过client新建一个Call去执行这个request，就会同步阻塞式地得到Response响应。

- Request的构建：
request中主要就是配置了请求的url和请求方式、请求体。
一共有六种请求方式：
```kotlin
open fun get() = method("GET", null)

open fun head() = method("HEAD", null)

open fun post(body: RequestBody) = method("POST", body)

@JvmOverloads
open fun delete(body: RequestBody? = Util.EMPTY_REQUEST) = method("DELETE", body)

open fun put(body: RequestBody) = method("PUT", body)

open fun patch(body: RequestBody) = method("PATCH", body)
```

- Call的创建：
首先看一下Call的文档说明：

> A call is a request that has been prepared for execution. A call can be canceled. As this object represents a single request/response pair (stream), it cannot be executed twice.

调用是已经准备好执行的请求。一个调用可以取消。由于此对象表示单个请求/响应对(流)，因此不能执行两次。

```kotlin
interface Call : Cloneable {
  /** Returns the original request that initiated this call.  */
  fun request(): Request

  /**
   * Invokes the request immediately, and blocks until the response can be processed or is in error.
   *
   * To avoid leaking resources callers should close the [Response] which in turn will close the
   * underlying [ResponseBody].
   *
   * ```
   * // ensure the response (and underlying response body) is closed
   * try (Response response = client.newCall(request).execute()) {
   *   ...
   * }
   * ```
   *
   * The caller may read the response body with the response's [Response.body] method. To avoid
   * leaking resources callers must [close the response body][ResponseBody] or the response.
   *
   * Note that transport-layer success (receiving a HTTP response code, headers and body) does not
   * necessarily indicate application-layer success: `response` may still indicate an unhappy HTTP
   * response code like 404 or 500.
   *
   * @throws IOException if the request could not be executed due to cancellation, a connectivity
   *     problem or timeout. Because networks can fail during an exchange, it is possible that the
   *     remote server accepted the request before the failure.
   * @throws IllegalStateException when the call has already been executed.
   */
  @Throws(IOException::class)
  fun execute(): Response

  /**
   * Schedules the request to be executed at some point in the future.
   *
   * The [dispatcher][OkHttpClient.dispatcher] defines when the request will run: usually
   * immediately unless there are several other requests currently being executed.
   *
   * This client will later call back `responseCallback` with either an HTTP response or a failure
   * exception.
   *
   * @throws IllegalStateException when the call has already been executed.
   */
  fun enqueue(responseCallback: Callback)

  /** Cancels the request, if possible. Requests that are already complete cannot be canceled.  */
  fun cancel()

  /**
   * Returns true if this call has been either [executed][execute] or [enqueued][enqueue]. It is an
   * error to execute a call more than once.
   */
  fun isExecuted(): Boolean

  fun isCanceled(): Boolean

  /**
   * Returns a timeout that spans the entire call: resolving DNS, connecting, writing the request
   * body, server processing, and reading the response body. If the call requires redirects or
   * retries all must complete within one timeout period.
   *
   * Configure the client's default timeout with [OkHttpClient.Builder.callTimeout].
   */
  fun timeout(): Timeout

  /**
   * Create a new, identical call to this one which can be enqueued or executed even if this call
   * has already been.
   */
  override fun clone(): Call

  interface Factory {
    fun newCall(request: Request): Call
  }
}
```
主要是execute和enqueue方法，分别代表同步阻塞的线程模型和异步回调的线程模型。
回到我们Call的创建过程client.newCall：

```kotlin
/** Prepares the [request] to be executed at some point in the future. */
override fun newCall(request: Request): Call {
  return RealCall.newRealCall(this, request, false /* for web socket */)
}
fun newRealCall(
  client: OkHttpClient,
  originalRequest: Request,
  forWebSocket: Boolean
): RealCall {
  // Safely publish the Call instance to the EventListener.
  return RealCall(client, originalRequest, forWebSocket).apply {
    transmitter = Transmitter(client, this)
  }
}
```
这里的RealCall是Call接口的一个实现类，我们来看execute方法的实现：

```kotlin
override fun execute(): Response {
  synchronized(this) {
    check(!executed) { "Already Executed" }
    executed = true
  }
  transmitter.timeoutEnter()
  transmitter.callStart()
  try {
    client.dispatcher().executed(this)
    return getResponseWithInterceptorChain()
  } finally {
    client.dispatcher().finished(this)
  }
}
```
我们看到，这个Call被加锁了，已经执行过的调用就不能再执行了。transmitter两个方法在这里一个用来控制超时，一个用来做事件回调。关键我们来看try代码块：client的dispatcher的executed方法：

```kotlin
private val runningSyncCalls = ArrayDeque<RealCall>()
@Synchronized internal fun executed(call: RealCall) {
  runningSyncCalls.add(call)
}
```
这里用两端进出的数组队列保存了这个调用，由于ArrayDeque是非线程安全的数据结构，因此这个方法加了锁；接着我们看getResponseWithInterceptorChain()方法，从方法名上看这里应该是有一个拦截链，事实上也确实是这样，这里的拦截链的设计给我们的扩展提供了很大的灵活性，其实这就是传说中的责任链模式，我们来看代码：

```kotlin
@Throws(IOException::class)
fun getResponseWithInterceptorChain(): Response {
  // Build a full stack of interceptors.
  val interceptors = ArrayList<Interceptor>()
  interceptors.addAll(client.interceptors())
  interceptors.add(RetryAndFollowUpInterceptor(client))
  interceptors.add(BridgeInterceptor(client.cookieJar()))
  interceptors.add(CacheInterceptor(client.internalCache()))
  interceptors.add(ConnectInterceptor(client))
  if (!forWebSocket) {
    interceptors.addAll(client.networkInterceptors())
  }
  interceptors.add(CallServerInterceptor(forWebSocket))

  val chain = RealInterceptorChain(interceptors, transmitter, null, 0,
      originalRequest, this, client.connectTimeoutMillis(),
      client.readTimeoutMillis(), client.writeTimeoutMillis())

  var calledNoMoreExchanges = false
  try {
    val response = chain.proceed(originalRequest)
    if (transmitter.isCanceled) {
      closeQuietly(response)
      throw IOException("Canceled")
    }
    return response
  } catch (e: IOException) {
    calledNoMoreExchanges = true
    throw transmitter.noMoreExchanges(e) as Throwable
  } finally {
    if (!calledNoMoreExchanges) {
      transmitter.noMoreExchanges(null)
    }
  }
}
```
果然，首先就是添加一些默认的拦截器和我们自定义的拦截器。可以看到，client中的interceptors就是我们自定义的：

```kotlin
fun addInterceptor(interceptor: Interceptor) = apply {
  interceptors += interceptor
}
```
接着创建拦截器调用链，并执行它的proceed方法，返回值就是我们的response了，那这里应该就是关键了，我们进proceed方法：

```kotlin
@Throws(IOException::class)
fun proceed(request: Request, transmitter: Transmitter, exchange: Exchange?): Response {
  if (index >= interceptors.size) throw AssertionError()

  calls++

  // If we already have a stream, confirm that the incoming request will use it.
  if (this.exchange != null && !this.exchange.connection().supportsUrl(request.url())) {
    throw IllegalStateException("network interceptor " + interceptors[index - 1]
        + " must retain the same host and port")
  }

  // If we already have a stream, confirm that this is the only call to chain.proceed().
  if (this.exchange != null && calls > 1) {
    throw IllegalStateException("network interceptor " + interceptors[index - 1]
        + " must call proceed() exactly once")
  }

  // Call the next interceptor in the chain.
  val next = RealInterceptorChain(interceptors, transmitter, exchange,
      index + 1, request, call, connectTimeout, readTimeout, writeTimeout)
  val interceptor = interceptors[index]
  val response = interceptor.intercept(next)

  // Confirm that the next interceptor made its required call to chain.proceed().
  if (exchange != null && index + 1 < interceptors.size && next.calls != 1) {
    throw IllegalStateException("network interceptor " + interceptor
        + " must call proceed() exactly once")
  }

  // Confirm that the intercepted response isn't null.
  if (response == null) {
    throw NullPointerException("interceptor $interceptor returned null")
  }

  if (response.body() == null) {
    throw IllegalStateException(
        "interceptor $interceptor returned a response with no body")
  }

  return response
}
```
前面依旧是一些状态校验的健壮性代码，我们暂时不管，关键看到next，这里又创建了一个RealInterceptorChain，这里的调用链之所以能成为链状结构，主要就是这里interceptors数组和RealInterceptorChain创建时不断递增的index，其实这里的命名我觉得换成Node更好，因为这里的Chain其实是链状结构上的一个节点。接着我们看interceptor的intercept方法，我们这里默认的第一个interceptor是RetryAndFollowUpInterceptor：

```java
@Override public Response intercept(Chain chain) throws IOException {
  Request request = chain.request();
  RealInterceptorChain realChain = (RealInterceptorChain) chain;
  Transmitter transmitter = realChain.transmitter();

  int followUpCount = 0;
  Response priorResponse = null;
  while (true) {
    transmitter.prepareToConnect(request);

    if (transmitter.isCanceled()) {
      throw new IOException("Canceled");
    }

    Response response;
    boolean success = false;
    try {
      response = realChain.proceed(request, transmitter, null);
      success = true;
    } catch (RouteException e) {
      // The attempt to connect via a route failed. The request will not have been sent.
      if (!recover(e.getLastConnectException(), transmitter, false, request)) {
        throw e.getFirstConnectException();
      }
      continue;
    } catch (IOException e) {
      // An attempt to communicate with a server failed. The request may have been sent.
      boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
      if (!recover(e, transmitter, requestSendStarted, request)) throw e;
      continue;
    } finally {
      // The network call threw an exception. Release any resources.
      if (!success) {
        transmitter.exchangeDoneDueToException();
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

    Exchange exchange = Internal.instance.exchange(response);
    Route route = exchange != null ? exchange.connection().route() : null;
    Request followUp = followUpRequest(response, route);

    if (followUp == null) {
      if (exchange != null && exchange.isDuplex()) {
        transmitter.timeoutEarlyExit();
      }
      return response;
    }

    RequestBody followUpBody = followUp.body();
    if (followUpBody != null && followUpBody.isOneShot()) {
      return response;
    }

    closeQuietly(response.body());
    if (transmitter.hasExchange()) {
      exchange.detachWithViolence();
    }

    if (++followUpCount > MAX_FOLLOW_UPS) {
      throw new ProtocolException("Too many follow-up requests: " + followUpCount);
    }

    request = followUp;
    priorResponse = response;
  }
}
```
这边的while死循环和try catch代码块帮助我们实现了错误时的重试机制以及责任链的递归调用，由于这里是递归（其实并不是严格意义上的递归，而是两个方法循环调用，直到一个interceptor不再通过proceed方法获取response，也就终止了这个“递归”），因此我们最终拦截器是由后向前执行的，所以我们找到最后添加的interceptor-CallServerInterceptor，也就是我们真正向服务器发出请求的拦截器：

```java
@Override public Response intercept(Chain chain) throws IOException {
  RealInterceptorChain realChain = (RealInterceptorChain) chain;
  Exchange exchange = realChain.exchange();
  Request request = realChain.request();

  long sentRequestMillis = System.currentTimeMillis();

  exchange.writeRequestHeaders(request);

  boolean responseHeadersStarted = false;
  Response.Builder responseBuilder = null;
  if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
    // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
    // Continue" response before transmitting the request body. If we don't get that, return
    // what we did get (such as a 4xx response) without ever transmitting the request body.
    if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
      exchange.flushRequest();
      responseHeadersStarted = true;
      exchange.responseHeadersStart();
      responseBuilder = exchange.readResponseHeaders(true);
    }

    if (responseBuilder == null) {
      if (request.body().isDuplex()) {
        // Prepare a duplex body so that the application can send a request body later.
        exchange.flushRequest();
        BufferedSink bufferedRequestBody = Okio.buffer(
            exchange.createRequestBody(request, true));
        request.body().writeTo(bufferedRequestBody);
      } else {
        // Write the request body if the "Expect: 100-continue" expectation was met.
        BufferedSink bufferedRequestBody = Okio.buffer(
            exchange.createRequestBody(request, false));
        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
      }
    } else {
      exchange.noRequestBody();
      if (!exchange.connection().isMultiplexed()) {
        // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
        // from being reused. Otherwise we're still obligated to transmit the request body to
        // leave the connection in a consistent state.
        exchange.noNewExchangesOnConnection();
      }
    }
  } else {
    exchange.noRequestBody();
  }

  if (request.body() == null || !request.body().isDuplex()) {
    exchange.finishRequest();
  }

  if (!responseHeadersStarted) {
    exchange.responseHeadersStart();
  }

  if (responseBuilder == null) {
    responseBuilder = exchange.readResponseHeaders(false);
  }

  Response response = responseBuilder
      .request(request)
      .handshake(exchange.connection().handshake())
      .sentRequestAtMillis(sentRequestMillis)
      .receivedResponseAtMillis(System.currentTimeMillis())
      .build();

  int code = response.code();
  if (code == 100) {
    // server sent a 100-continue even though we did not request one.
    // try again to read the actual response
    response = exchange.readResponseHeaders(false)
        .request(request)
        .handshake(exchange.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    code = response.code();
  }

  exchange.responseHeadersEnd(response);

  if (forWebSocket && code == 101) {
    // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
    response = response.newBuilder()
        .body(Util.EMPTY_RESPONSE)
        .build();
  } else {
    response = response.newBuilder()
        .body(exchange.openResponseBody(response))
        .build();
  }

  if ("close".equalsIgnoreCase(response.request().header("Connection"))
      || "close".equalsIgnoreCase(response.header("Connection"))) {
    exchange.noNewExchangesOnConnection();
  }

  if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
    throw new ProtocolException(
        "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
  }

  return response;
}
```
这里的exchange是在上层拦截器ConnectInterceptor中传入的

```java
@Override public Response intercept(Chain chain) throws IOException {
  RealInterceptorChain realChain = (RealInterceptorChain) chain;
  Request request = realChain.request();
  Transmitter transmitter = realChain.transmitter();

  // We need the network to satisfy this request. Possibly for validating a conditional GET.
  boolean doExtensiveHealthChecks = !request.method().equals("GET");
  Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);

  return realChain.proceed(request, transmitter, exchange);
}

Exchange newExchange(Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
  synchronized (connectionPool) {
    if (noMoreExchanges) throw new IllegalStateException("released");
    if (exchange != null) throw new IllegalStateException("exchange != null");
  }

  ExchangeCodec codec = exchangeFinder.find(client, chain, doExtensiveHealthChecks);
  Exchange result = new Exchange(this, call, eventListener, exchangeFinder, codec);

  synchronized (connectionPool) {
    this.exchange = result;
    this.exchangeRequestDone = false;
    this.exchangeResponseDone = false;
    return result;
  }
}
```
这里的Exchange其实就是真实的数据交换层，真正的IO操作就是在这里，我们看到这里还有一个连接池对连接进行管理，在ExchangeFinder的find方法中会从连接池中取出一条可用连接，如果没有的话会创建一条连接丢入连接池并进行TCP和TLS握手等准备工作。我们看到exchangeFinder的find方法：

```java
public ExchangeCodec find(
    OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
  int connectTimeout = chain.connectTimeoutMillis();
  int readTimeout = chain.readTimeoutMillis();
  int writeTimeout = chain.writeTimeoutMillis();
  int pingIntervalMillis = client.pingIntervalMillis();
  boolean connectionRetryEnabled = client.retryOnConnectionFailure();

  try {
    RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
        writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
    return resultConnection.newCodec(client, chain);
  } catch (RouteException e) {
    trackFailure();
    throw e;
  } catch (IOException e) {
    trackFailure();
    throw new RouteException(e);
  }
}
```
很显然这里就是要去找一条可用的连接，这其中打开了socket建立了TCP连接，然后把编码解码器返回：

```java
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
      candidate.noNewExchanges();
      continue;
    }

    return candidate;
  }
}
```
findConnection()方法代码很长，就不贴上来了，贴上注释：

> Returns a connection to host a new stream. This prefers the existing connection if it exists,then the pool, finally building a new connection.

和我们之前的说明一致，优先找已存在的连接，如果找不到就去连接池中找，连接池中也找不到就新建一条连接。接着我们回到CallServerInterceptor的writeRequestHeaders()方法，这里会调用ExchangeCodec的writeRequestHeaders()方法，这里的ExchangeCodec是一个接口，我们找到初始化这个编码解码器地方：

```java
ExchangeCodec newCodec(OkHttpClient client, Interceptor.Chain chain) throws SocketException {
 if (http2Connection != null) {
    return new Http2ExchangeCodec(client, this, chain, http2Connection);
  } else {
    socket.setSoTimeout(chain.readTimeoutMillis());
    source.timeout().timeout(chain.readTimeoutMillis(), MILLISECONDS);
    sink.timeout().timeout(chain.writeTimeoutMillis(), MILLISECONDS);
    return new Http1ExchangeCodec(client, this, source, sink);
  }
}
```
这里我们看到目前依旧是HTTP1.1，我们看他的writeRequestHeaders()：

```java
public void writeRequest(Headers headers, String requestLine) throws IOException {
  if (state != STATE_IDLE) throw new IllegalStateException("state: " + state);
  sink.writeUtf8(requestLine).writeUtf8("\r\n");
  for (int i = 0, size = headers.size(); i < size; i++) {
    sink.writeUtf8(headers.name(i))
        .writeUtf8(": ")
        .writeUtf8(headers.value(i))
        .writeUtf8("\r\n");
  }
  sink.writeUtf8("\r\n");
  state = STATE_OPEN_REQUEST_BODY;
}
```
这里可以看到我们把基本的http请求头信息和我们自定义的请求头信息全都以utf-8编码格式写入了缓冲区并进行io操作。如果是post请求，我们看请求体的创建：

```java
@Override public Sink createRequestBody(Request request, long contentLength) throws IOException {
  if (request.body() != null && request.body().isDuplex()) {
    throw new ProtocolException("Duplex connections are not supported for HTTP/1");
  }

  if ("chunked".equalsIgnoreCase(request.header("Transfer-Encoding"))) {
    // Stream a request body of unknown length.
    return newChunkedSink();
  }

  if (contentLength != -1L) {
    // Stream a request body of a known length.
    return newKnownLengthSink();
  }

  throw new IllegalStateException(
      "Cannot stream a request body without chunked encoding or a known content length!");
}
```
可以看到这里支持分块传输编码和单次传输编码，这里把请求体也写入到了请求头的缓冲区中，用Segment保存。接着调用requestBody的writeTo方法，由于这里是一个抽象方法，我们需要找到我们创建的RequestBody中重写的writeTo方法，这里可以看到，就是调用了sink的writeTo方法和close方法进行数据的发送

```kotlin
override fun writeTo(sink: BufferedSink) {
 sink.write(content)
}
override fun close() {
  if (closed) return

  // Emit buffered data to the underlying sink. If this fails, we still need
  // to close the sink; otherwise we risk leaking resources.
  var thrown: Throwable? = null
  try {
    if (buffer.size > 0) {
      sink.write(buffer, buffer.size)
    }
  } catch (e: Throwable) {
    thrown = e
  }

  try {
    sink.close()
  } catch (e: Throwable) {
    if (thrown == null) thrown = e
  }

  closed = true

  if (thrown != null) throw thrown
}
```
写入工作完成后就是阻塞地等待响应了

```java
public @Nullable Response.Builder readResponseHeaders(boolean expectContinue) throws IOException {
  try {
    Response.Builder result = codec.readResponseHeaders(expectContinue);
    if (result != null) {
      Internal.instance.initExchange(result, this);
    }
    return result;
  } catch (IOException e) {
    eventListener.responseFailed(call, e);
    trackFailure(e);
    throw e;
  }
}
@Override public Response.Builder readResponseHeaders(boolean expectContinue) throws IOException {
  if (state != STATE_OPEN_REQUEST_BODY && state != STATE_READ_RESPONSE_HEADERS) {
    throw new IllegalStateException("state: " + state);
  }

  try {
    StatusLine statusLine = StatusLine.parse(readHeaderLine());

    Response.Builder responseBuilder = new Response.Builder()
        .protocol(statusLine.protocol)
        .code(statusLine.code)
        .message(statusLine.message)
        .headers(readHeaders());

    if (expectContinue && statusLine.code == HTTP_CONTINUE) {
      return null;
    } else if (statusLine.code == HTTP_CONTINUE) {
      state = STATE_READ_RESPONSE_HEADERS;
      return responseBuilder;
    }

    state = STATE_OPEN_RESPONSE_BODY;
    return responseBuilder;
  } catch (EOFException e) {
    // Provide more context if the server ends the stream before sending a response.
    throw new IOException("unexpected end of stream on "
        + realConnection.route().address().url().redact(), e);
  }
}
```
这里首先读取状态信息，接着根据状态信息构建响应并返回，再打开输入流读取body并设置到Response中，我们关键看codec的openResponseBodySource：

```java
@Override public Source openResponseBodySource(Response response) {
  if (!HttpHeaders.hasBody(response)) {
    return newFixedLengthSource(0);
  }

  if ("chunked".equalsIgnoreCase(response.header("Transfer-Encoding"))) {
    return newChunkedSource(response.request().url());
  }

  long contentLength = HttpHeaders.contentLength(response);
  if (contentLength != -1) {
    return newFixedLengthSource(contentLength);
  }

  return newUnknownLengthSource();
}
```
在这里我们同样能看到HTTP1.1协议支持分块传输和单次传输，这里我们就根据不同的情况采用不同的方式去读取socket中的数据并封装到Response的Body中，最终将response返回。
OkHttp的执行流程基本就是这样了，大体阅读完一遍源码之后深深体会到了设计模式的精妙，清楚的各个类的职责，拦截器责任链模式将Http协议需要做的工作清楚地划分开，并且具有高度的可扩展性。
由于能力所限，分析中难免有不正确或者不准确的地方，欢迎指正～
