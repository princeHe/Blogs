@[TOC](Glide源码阅读理解系列（一）)
# Glide
[官方中文文档](https://muyangmin.github.io/glide-docs-cn/)
## 简介
> Glide是一个快速高效的Android图片加载库，注重于平滑的滚动。Glide提供了易用的API，高性能、可扩展的图片解码管道（decode pipeline），以及自动的资源池技术。
Glide 支持拉取，解码和展示视频快照，图片，和GIF动画。Glide的Api是如此的灵活，开发者甚至可以插入和替换成自己喜爱的任何网络栈。默认情况下，Glide使用的是一个定制化的基于HttpUrlConnection的栈，但同时也提供了与Google Volley和Square OkHttp快速集成的工具库。
虽然Glide 的主要目标是让任何形式的图片列表的滚动尽可能地变得更快、更平滑，但实际上，Glide几乎能满足你对远程图片的拉取/缩放/显示的一切需求。
## 简单使用

```java
Glide.with(context)
    .load(url)
    .into(imageView);

```
## 源码执行流程分析

```java
@NonNull
public static RequestManager with(@NonNull Context context) {
  return getRetriever(context).get(context);
}
```
with方法首先给我们返回了一个RequestManager，我们看下with方法，这里应该做了初始化的工作，因为Glide并不需要我们在Application初始化的时候手动去做初始化的工作。

```java
@NonNull
private static RequestManagerRetriever getRetriever(@Nullable Context context) {
  // Context could be null for other reasons (ie the user passes in null), but in practice it will
  // only occur due to errors with the Fragment lifecycle.
  Preconditions.checkNotNull(
      context,
      "You cannot start a load on a not yet attached View or a Fragment where getActivity() "
          + "returns null (which usually occurs when getActivity() is called before the Fragment "
          + "is attached or after the Fragment is destroyed).");
  return Glide.get(context).getRequestManagerRetriever();
}
/**
 * Get the singleton.
 *
 * @return the singleton
 */
@NonNull
public static Glide get(@NonNull Context context) {
  if (glide == null) {
    synchronized (Glide.class) {
      if (glide == null) {
        checkAndInitializeGlide(context);
      }
    }
  }

  return glide;
}
```
果然如我们所料，这里初始化了Glide，是个double check的单例。checkAndInitializeGlide方法就不看了，和普通的初始化工作没什么两样。getRequestManagerRetriever()就是把Glide初始化过程中new出来的RequestManagerRetriever返回了，接着我们看下获取请求管理器的代码：

```java
@NonNull
public RequestManager get(@NonNull Context context) {
  if (context == null) {
    throw new IllegalArgumentException("You cannot start a load on a null Context");
  } else if (Util.isOnMainThread() && !(context instanceof Application)) {
    if (context instanceof FragmentActivity) {
      return get((FragmentActivity) context);
    } else if (context instanceof Activity) {
      return get((Activity) context);
    } else if (context instanceof ContextWrapper) {
      return get(((ContextWrapper) context).getBaseContext());
    }
  }

  return getApplicationManager(context);
}
```
这里根据不同的context类型创建了不同的RequestManager，主要是用于区分不同容器的生命周期的管理，这里我们不细看，后面会有专门的章节分析Glide的生命周期管理，内存泄漏的控制。
到这里，我们的Glide的初始化工作就完成了，如果不是首次调用，都不需要初始化。接着我们看load方法，我们拿网络图片为例：

```java
@NonNull
@CheckResult
@Override
public RequestBuilder<Drawable> load(@Nullable String string) {
  return asDrawable().load(string);
}
@NonNull
@CheckResult
public RequestBuilder<Drawable> asDrawable() {
  return as(Drawable.class);
}
@NonNull
@CheckResult
public <ResourceType> RequestBuilder<ResourceType> as(
    @NonNull Class<ResourceType> resourceClass) {
  return new RequestBuilder<>(glide, this, resourceClass, context);
}
```
这里默认情况下我们会创建一个ResourceType为Drawable类型的RequestBuilder，我们继续看ResourceBuilder的load方法：

```java
@NonNull
@Override
@CheckResult
public RequestBuilder<TranscodeType> load(@Nullable String string) {
  return loadGeneric(string);
}
@NonNull
private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
  this.model = model;
  isModelSet = true;
  return this;
}
```
到这里load方法也已经结束了，他的主要的工作就是创建一个RequestBuilder并把设置存起来。继续，来看into方法：

```java
@NonNull
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
  Util.assertMainThread();
  Preconditions.checkNotNull(view);

  BaseRequestOptions<?> requestOptions = this;
  if (!requestOptions.isTransformationSet()
      && requestOptions.isTransformationAllowed()
      && view.getScaleType() != null) {
    // Clone in this method so that if we use this RequestBuilder to load into a View and then
    // into a different target, we don't retain the transformation applied based on the previous
    // View's scale type.
    switch (view.getScaleType()) {
      case CENTER_CROP:
        requestOptions = requestOptions.clone().optionalCenterCrop();
        break;
      case CENTER_INSIDE:
        requestOptions = requestOptions.clone().optionalCenterInside();
        break;
      case FIT_CENTER:
      case FIT_START:
      case FIT_END:
        requestOptions = requestOptions.clone().optionalFitCenter();
        break;
      case FIT_XY:
        requestOptions = requestOptions.clone().optionalCenterInside();
        break;
      case CENTER:
      case MATRIX:
      default:
        // Do nothing.
    }
  }

  return into(
      glideContext.buildImageViewTarget(view, transcodeClass),
      /*targetListener=*/ null,
      requestOptions,
      Executors.mainThreadExecutor());
}
```
前面一部分是缩放模式，我们看真正的into方法：

```java
private <Y extends Target<TranscodeType>> Y into(
    @NonNull Y target,
    @Nullable RequestListener<TranscodeType> targetListener,
    BaseRequestOptions<?> options,
    Executor callbackExecutor) {
  Preconditions.checkNotNull(target);
  if (!isModelSet) {
    throw new IllegalArgumentException("You must call #load() before calling #into()");
  }

  Request request = buildRequest(target, targetListener, options, callbackExecutor);

  Request previous = target.getRequest();
  if (request.isEquivalentTo(previous)
      && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
    request.recycle();
    // If the request is completed, beginning again will ensure the result is re-delivered,
    // triggering RequestListeners and Targets. If the request is failed, beginning again will
    // restart the request, giving it another chance to complete. If the request is already
    // running, we can let it continue running without interruption.
    if (!Preconditions.checkNotNull(previous).isRunning()) {
      // Use the previous request rather than the new one to allow for optimizations like skipping
      // setting placeholders, tracking and un-tracking Targets, and obtaining View dimensions
      // that are done in the individual Request.
      previous.begin();
    }
    return target;
  }

  requestManager.clear(target);
  target.setRequest(request);
  requestManager.track(target, request);

  return target;
}
```
首先看下方法的四个参数：

 - 第一个是Target，可以将资源加载到其中，并在加载期间通知相关的生命周期事件。这里帮我们把我们传入的ImageView做了一次wrap操作，包装成了一个Target；
 - 第二个参数是资源加载的回调，有需要的话可以自己实现；
 - 第三个参数是请求参数，也就是这里的RequestBuilder;
 - 第四个参数是指定回调执行的线程，这里是在主线程上执行，也就是在主线程操作ui。

接着我们看方法实现：首先我们创建一个请求：

```java
private Request buildRequest(
    Target<TranscodeType> target,
    @Nullable RequestListener<TranscodeType> targetListener,
    BaseRequestOptions<?> requestOptions,
    Executor callbackExecutor) {
  return buildRequestRecursive(
      target,
      targetListener,
      /*parentCoordinator=*/ null,
      transitionOptions,
      requestOptions.getPriority(),
      requestOptions.getOverrideWidth(),
      requestOptions.getOverrideHeight(),
      requestOptions,
      callbackExecutor);
}
////////////////////////////////////////////////////////////////////////////////////
private Request buildRequestRecursive(
      Target<TranscodeType> target,
      @Nullable RequestListener<TranscodeType> targetListener,
      @Nullable RequestCoordinator parentCoordinator,
      TransitionOptions<?, ? super TranscodeType> transitionOptions,
      Priority priority,
      int overrideWidth,
      int overrideHeight,
      BaseRequestOptions<?> requestOptions,
      Executor callbackExecutor) {

    // Build the ErrorRequestCoordinator first if necessary so we can update parentCoordinator.
    ErrorRequestCoordinator errorRequestCoordinator = null;
    if (errorBuilder != null) {
      errorRequestCoordinator = new ErrorRequestCoordinator(parentCoordinator);
      parentCoordinator = errorRequestCoordinator;
    }

    Request mainRequest =
        buildThumbnailRequestRecursive(
            target,
            targetListener,
            parentCoordinator,
            transitionOptions,
            priority,
            overrideWidth,
            overrideHeight,
            requestOptions,
            callbackExecutor);

    if (errorRequestCoordinator == null) {
      return mainRequest;
    }

    int errorOverrideWidth = errorBuilder.getOverrideWidth();
    int errorOverrideHeight = errorBuilder.getOverrideHeight();
    if (Util.isValidDimensions(overrideWidth, overrideHeight)
        && !errorBuilder.isValidOverride()) {
      errorOverrideWidth = requestOptions.getOverrideWidth();
      errorOverrideHeight = requestOptions.getOverrideHeight();
    }

    Request errorRequest =
        errorBuilder.buildRequestRecursive(
            target,
            targetListener,
            errorRequestCoordinator,
            errorBuilder.transitionOptions,
            errorBuilder.getPriority(),
            errorOverrideWidth,
            errorOverrideHeight,
            errorBuilder,
            callbackExecutor);
    errorRequestCoordinator.setRequests(mainRequest, errorRequest);
    return errorRequestCoordinator;
  }
```
这里由于没有errorBuilder，所以我们就来看mainRequest(由于代码块太长，只截取默认情况下的逻辑分支):

```java
return obtainRequest(
          target,
          targetListener,
          requestOptions,
          parentCoordinator,
          transitionOptions,
          priority,
          overrideWidth,
          overrideHeight,
          callbackExecutor);
return SingleRequest.obtain(
        context,
        glideContext,
        model,
        transcodeClass,
        requestOptions,
        overrideWidth,
        overrideHeight,
        priority,
        target,
        targetListener,
        requestListeners,
        requestCoordinator,
        glideContext.getEngine(),
        transitionOptions.getTransitionFactory(),
        callbackExecutor);
 
 public static <R> SingleRequest<R> obtain(
    Context context,
    GlideContext glideContext,
    Object model,
    Class<R> transcodeClass,
    BaseRequestOptions<?> requestOptions,
    int overrideWidth,
    int overrideHeight,
    Priority priority,
    Target<R> target,
    RequestListener<R> targetListener,
    @Nullable List<RequestListener<R>> requestListeners,
    RequestCoordinator requestCoordinator,
    Engine engine,
    TransitionFactory<? super R> animationFactory,
    Executor callbackExecutor) {
  @SuppressWarnings("unchecked") SingleRequest<R> request =
      (SingleRequest<R>) POOL.acquire();
  if (request == null) {
    request = new SingleRequest<>();
  }
  request.init(
      context,
      glideContext,
      model,
      transcodeClass,
      requestOptions,
      overrideWidth,
      overrideHeight,
      priority,
      target,
      targetListener,
      requestListeners,
      requestCoordinator,
      engine,
      animationFactory,
      callbackExecutor);
  return request;
}
```
我们看到在SingleRequest的obtain方法中，首先去POOL中取一个可用的，如果没有取到再去创建一个，之后将请求初始化，并返回，到这里我们创建请求就完成了。
接着target的getRequest方法返回的应该是之前设置的request，这里的判断应该是为了防止重复的请求，如果之前已经有了这个请求，则把这次的请求释放掉，再看之前的请求是否在执行，不在执行就开始执行。如果没有这个请求，调用requestManager的track方法：

```java
synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
  targetTracker.track(target);
  requestTracker.runRequest(request);
}
```
我们主要看请求：

```java
public void runRequest(@NonNull Request request) {
  requests.add(request);
  if (!isPaused) {
    request.begin();
  } else {
    request.clear();
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(TAG, "Paused, delaying request");
    }
    pendingRequests.add(request);
  }
}
```
这里请求就开始执行了，我们看具体实现类SingleRequest的begin()方法：

```java
@Override
public synchronized void begin() {
  assertNotCallingCallbacks();
  stateVerifier.throwIfRecycled();
  startTime = LogTime.getLogTime();
  if (model == null) {
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
      width = overrideWidth;
      height = overrideHeight;
    }
    // Only log at more verbose log levels if the user has set a fallback drawable, because
    // fallback Drawables indicate the user expects null models occasionally.
    int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
    onLoadFailed(new GlideException("Received null model"), logLevel);
    return;
  }

  if (status == Status.RUNNING) {
    throw new IllegalArgumentException("Cannot restart a running request");
  }

  // If we're restarted after we're complete (usually via something like a notifyDataSetChanged
  // that starts an identical request into the same Target or View), we can simply use the
  // resource and size we retrieved the last time around and skip obtaining a new size, starting a
  // new load etc. This does mean that users who want to restart a load because they expect that
  // the view size has changed will need to explicitly clear the View or Target before starting
  // the new load.
  if (status == Status.COMPLETE) {
    onResourceReady(resource, DataSource.MEMORY_CACHE);
    return;
  }

  // Restarts for requests that are neither complete nor running can be treated as new requests
  // and can run again from the beginning.

  status = Status.WAITING_FOR_SIZE;
  if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
    onSizeReady(overrideWidth, overrideHeight);
  } else {
    target.getSize(this);
  }

  if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
      && canNotifyStatusChanged()) {
    target.onLoadStarted(getPlaceholderDrawable());
  }
  if (IS_VERBOSE_LOGGABLE) {
    logV("finished run method in " + LogTime.getElapsedMillis(startTime));
  }
}
```
这里绝大部分都是校验代码，关键在修改状态为Status.WAITING_FOR_SIZE后，这里如果校验尺寸是合法的，那么会执行onSizeReady()，我们进入方法看一看：
```java
@Override
public synchronized void onSizeReady(int width, int height) {
  stateVerifier.throwIfRecycled();
  if (IS_VERBOSE_LOGGABLE) {
    logV("Got onSizeReady in " + LogTime.getElapsedMillis(startTime));
  }
  if (status != Status.WAITING_FOR_SIZE) {
    return;
  }
  status = Status.RUNNING;

  float sizeMultiplier = requestOptions.getSizeMultiplier();
  this.width = maybeApplySizeMultiplier(width, sizeMultiplier);
  this.height = maybeApplySizeMultiplier(height, sizeMultiplier);

  if (IS_VERBOSE_LOGGABLE) {
    logV("finished setup for calling load in " + LogTime.getElapsedMillis(startTime));
  }
  loadStatus =
      engine.load(
          glideContext,
          model,
          requestOptions.getSignature(),
          this.width,
          this.height,
          requestOptions.getResourceClass(),
          transcodeClass,
          priority,
          requestOptions.getDiskCacheStrategy(),
          requestOptions.getTransformations(),
          requestOptions.isTransformationRequired(),
          requestOptions.isScaleOnlyOrNoTransform(),
          requestOptions.getOptions(),
          requestOptions.isMemoryCacheable(),
          requestOptions.getUseUnlimitedSourceGeneratorsPool(),
          requestOptions.getUseAnimationPool(),
          requestOptions.getOnlyRetrieveFromCache(),
          this,
          callbackExecutor);

  // This is a hack that's only useful for testing right now where loads complete synchronously
  // even though under any executor running on any thread but the main thread, the load would
  // have completed asynchronously.
  if (status != Status.RUNNING) {
    loadStatus = null;
  }
  if (IS_VERBOSE_LOGGABLE) {
    logV("finished onSizeReady in " + LogTime.getElapsedMillis(startTime));
  }
}
```
这里就是engine真正的load方法了：

```java
public synchronized <R> LoadStatus load(
   GlideContext glideContext,
    Object model,
    Key signature,
    int width,
    int height,
    Class<?> resourceClass,
    Class<R> transcodeClass,
    Priority priority,
    DiskCacheStrategy diskCacheStrategy,
    Map<Class<?>, Transformation<?>> transformations,
    boolean isTransformationRequired,
    boolean isScaleOnlyOrNoTransform,
    Options options,
    boolean isMemoryCacheable,
    boolean useUnlimitedSourceExecutorPool,
    boolean useAnimationPool,
    boolean onlyRetrieveFromCache,
    ResourceCallback cb,
    Executor callbackExecutor) {
  long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;

  EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
      resourceClass, transcodeClass, options);

  EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
  if (active != null) {
    cb.onResourceReady(active, DataSource.MEMORY_CACHE);
    if (VERBOSE_IS_LOGGABLE) {
      logWithTimeAndKey("Loaded resource from active resources", startTime, key);
    }
    return null;
  }

  EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
  if (cached != null) {
    cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
    if (VERBOSE_IS_LOGGABLE) {
      logWithTimeAndKey("Loaded resource from cache", startTime, key);
    }
    return null;
  }

  EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
  if (current != null) {
    current.addCallback(cb, callbackExecutor);
    if (VERBOSE_IS_LOGGABLE) {
      logWithTimeAndKey("Added to existing load", startTime, key);
    }
    return new LoadStatus(cb, current);
  }

  EngineJob<R> engineJob =
      engineJobFactory.build(
          key,
          isMemoryCacheable,
          useUnlimitedSourceExecutorPool,
          useAnimationPool,
          onlyRetrieveFromCache);

  DecodeJob<R> decodeJob =
      decodeJobFactory.build(
          glideContext,
          model,
          key,
          signature,
          width,
          height,
          resourceClass,
          transcodeClass,
          priority,
          diskCacheStrategy,
          transformations,
          isTransformationRequired,
          isScaleOnlyOrNoTransform,
          onlyRetrieveFromCache,
          options,
          engineJob);

  jobs.put(key, engineJob);

  engineJob.addCallback(cb, callbackExecutor);
  engineJob.start(decodeJob);

  if (VERBOSE_IS_LOGGABLE) {
    logWithTimeAndKey("Started new load", startTime, key);
  }
  return new LoadStatus(cb, engineJob);
}
```
首先我们根据入参构建EngineKey，接着根据EngineKey去寻找ActiveResource，我们看下是如何寻找的：

```java
@Nullable
private EngineResource<?> loadFromActiveResources(Key key, boolean isMemoryCacheable) {
  if (!isMemoryCacheable) {
    return null;
  }
  EngineResource<?> active = activeResources.get(key);
  if (active != null) {
    active.acquire();
  }

  return active;
}
```
这里的activeResource肯定是使用了内存缓存的，如果内存缓存配置为不可用，直接返空值，接着我们看下activeResource的get方法：

```java
final Map<Key, ResourceWeakReference> activeEngineResources = new HashMap<>();
@Nullable
synchronized EngineResource<?> get(Key key) {
  ResourceWeakReference activeRef = activeEngineResources.get(key);
  if (activeRef == null) {
    return null;
  }

  EngineResource<?> active = activeRef.get();
  if (active == null) {
    cleanupActiveReference(activeRef);
  }
  return active;
}
```
这边是把弱引用的EngineResource作为值存在Map中。
如果loadFromActiveResource没有找到资源，那么就调用loadFromCache:

```java
private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
  if (!isMemoryCacheable) {
    return null;
  }

  EngineResource<?> cached = getEngineResourceFromCache(key);
  if (cached != null) {
    cached.acquire();
    activeResources.activate(key, cached);
  }
  return cached;
}
private EngineResource<?> getEngineResourceFromCache(Key key) {
  Resource<?> cached = cache.remove(key);

  final EngineResource<?> result;
  if (cached == null) {
    result = null;
  } else if (cached instanceof EngineResource) {
    // Save an object allocation if we've cached an EngineResource (the typical case).
    result = (EngineResource<?>) cached;
  } else {
    result = new EngineResource<>(
        cached, /*isMemoryCacheable=*/ true, /*isRecyclable=*/ true, key, /*listener=*/ this);
  }
  return result;
}
```
首先我们把key从MemoryCache中移除并返回这个key对应的Resource，这里的MemoryCache是一个借口，我们在这里的具体实现是LruResourceCache，也就是大名鼎鼎的LruCache：

```java
if (memoryCache == null) {
  memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
}
```
这里就不深入探究LruCache的实现原理了，有很多文章都有介绍，您也可以自己阅读源码研究。这里我们把从缓存获取到的资源返回并且加入ActiveResources。再接着，我们从Jobs中查找key对应的EngineJob，如果存在就去调用addCallback，这里似乎并没有做真正的工作。我们再往下看，我们创建了一个EngineJob和一个DecodeJob，把EngineJob加入了jobs中，再添加回调，然后开始解码工作，我们看下start(decodeJob)方法：

```java
public synchronized void start(DecodeJob<R> decodeJob) {
  this.decodeJob = decodeJob;
  GlideExecutor executor = decodeJob.willDecodeFromCache()
      ? diskCacheExecutor
      : getActiveSourceExecutor();
  executor.execute(decodeJob);
}
```
这里我们可以看到，我们把decodeJob丢到了线程池中执行，我们看下decodeJob的run方法：

```java
@Override
public void run() {
  // This should be much more fine grained, but since Java's thread pool implementation silently
  // swallows all otherwise fatal exceptions, this will at least make it obvious to developers
  // that something is failing.
  GlideTrace.beginSectionFormat("DecodeJob#run(model=%s)", model);
  // Methods in the try statement can invalidate currentFetcher, so set a local variable here to
  // ensure that the fetcher is cleaned up either way.
  DataFetcher<?> localFetcher = currentFetcher;
  try {
    if (isCancelled) {
      notifyFailed();
      return;
    }
    runWrapped();
  } catch (CallbackException e) {
    // If a callback not controlled by Glide throws an exception, we should avoid the Glide
    // specific debug logic below.
    throw e;
  } catch (Throwable t) {
    // Catch Throwable and not Exception to handle OOMs. Throwables are swallowed by our
    // usage of .submit() in GlideExecutor so we're not silently hiding crashes by doing this. We
    // are however ensuring that our callbacks are always notified when a load fails. Without this
    // notification, uncaught throwables never notify the corresponding callbacks, which can cause
    // loads to silently hang forever, a case that's especially bad for users using Futures on
    // background threads.
    if (Log.isLoggable(TAG, Log.DEBUG)) {
      Log.d(TAG, "DecodeJob threw unexpectedly"
          + ", isCancelled: " + isCancelled
          + ", stage: " + stage, t);
    }
    // When we're encoding we've already notified our callback and it isn't safe to do so again.
    if (stage != Stage.ENCODE) {
      throwables.add(t);
      notifyFailed();
    }
    if (!isCancelled) {
      throw t;
    }
    throw t;
  } finally {
    // Keeping track of the fetcher here and calling cleanup is excessively paranoid, we call
    // close in all cases anyway.
    if (localFetcher != null) {
      localFetcher.cleanup();
    }
    GlideTrace.endSection();
  }
}
private void runWrapped() {
  switch (runReason) {
    case INITIALIZE:
      stage = getNextStage(Stage.INITIALIZE);
      currentGenerator = getNextGenerator();
      runGenerators();
      break;
    case SWITCH_TO_SOURCE_SERVICE:
      runGenerators();
      break;
    case DECODE_DATA:
      decodeFromRetrievedData();
      break;
    default:
      throw new IllegalStateException("Unrecognized run reason: " + runReason);
  }
}

private DataFetcherGenerator getNextGenerator() {
  switch (stage) {
    case RESOURCE_CACHE:
      return new ResourceCacheGenerator(decodeHelper, this);
    case DATA_CACHE:
      return new DataCacheGenerator(decodeHelper, this);
    case SOURCE:
      return new SourceGenerator(decodeHelper, this);
    case FINISHED:
      return null;
    default:
      throw new IllegalStateException("Unrecognized stage: " + stage);
  }
}

private void runGenerators() {
  currentThread = Thread.currentThread();
  startFetchTime = LogTime.getLogTime();
  boolean isStarted = false;
  while (!isCancelled && currentGenerator != null
      && !(isStarted = currentGenerator.startNext())) {
    stage = getNextStage(stage);
    currentGenerator = getNextGenerator();

    if (stage == Stage.SOURCE) {
      reschedule();
      return;
    }
  }
  // We've run out of stages and generators, give up.
  if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
    notifyFailed();
  }

  // Otherwise a generator started a new load and we expect to be called back in
  // onDataFetcherReady.
}
```
这里的DataFetcher就是拉取数据的，是一个接口，我们可以自定义扩展它，因此可以实现自定义网络库的能力。Glide中也自带了很多的实现类用来获取不同环境下的资源，这里不再多做分析了。在runWrap方法中封装了decode的所有阶段和各个阶段执行的工作，最终把解码后的数据回调出去，最终把资源设置到封装了ImageView的DrawableImageViewTarget中
```java
@Override
protected void setResource(@Nullable Drawable resource) {
  view.setImageDrawable(resource);
}
```
**结束语**
Glide的基本执行流程就分析到这里了，可以看到和我们之前分析的RxJava一样，对于职责的分工非常明确，功能的抽象设计的非常赞，深深感受到了自己的差距还是很大，还需要好好修炼设计的内功鸭。
