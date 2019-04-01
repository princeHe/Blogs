@[TOC](RxJava源码阅读理解系列（四）)
# 操作符
RxJava中的操作符超级多，打开官方文档可以看到如下的说明
![官网操作符说明](https://img-blog.csdnimg.cn/20190326164011850.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3NTMyMTQw,size_16,color_FFFFFF,t_70)
这其中Transformation转换操作符是最值得分析的，接下来我们就来探究转换操作符的奥秘吧。

## buffer
![buffer](https://img-blog.csdnimg.cn/201903261654176.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3NTMyMTQw,size_16,color_FFFFFF,t_70)
如图所示，buffer的作用是定期将可观察到的项收集到包中，并将这些包发出，而不是一次一个地发出这些项，我们再来看下基本用法：

```java
Observable.range(0, 10)
    .buffer(4)
    .subscribe((List<Integer> buffer) -> System.out.println(buffer));
```
输出如下：
[0, 1, 2, 3]
[4, 5, 6, 7]
[8, 9]
可以看到，[0,10)范围的数字每四个被分成了一组。
接下来我们看源码入口：

```java
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
public final Observable<List<T>> buffer(int count) {
    return buffer(count, count);
}
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
public final Observable<List<T>> buffer(int count, int skip) {
    return buffer(count, skip, ArrayListSupplier.<T>asCallable());
}
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
public final <U extends Collection<? super T>> Observable<U> buffer(int count, int skip, Callable<U> bufferSupplier) {
    ObjectHelper.verifyPositive(count, "count");
    ObjectHelper.verifyPositive(skip, "skip");
    ObjectHelper.requireNonNull(bufferSupplier, "bufferSupplier is null");
    return RxJavaPlugins.onAssembly(new ObservableBuffer<T, U>(this, count, skip, bufferSupplier));
}
```
再接下来，进入ObservableBuffer的订阅函数：

```java
@Override
protected void subscribeActual(Observer<? super U> t) {
    if (skip == count) {
        BufferExactObserver<T, U> bes = new BufferExactObserver<T, U>(t, count, bufferSupplier);
        if (bes.createBuffer()) {
            source.subscribe(bes);
        }
    } else {
        source.subscribe(new BufferSkipObserver<T, U>(t, count, skip, bufferSupplier));
    }
}
```
我们这里skip是等于count的，所以会走到第一个分支，我们看下createBuffer()方法干了什么：

```java
boolean createBuffer() {
    U b;
    try {
        b = ObjectHelper.requireNonNull(bufferSupplier.call(), "Empty buffer supplied");
    } catch (Throwable t) {
        Exceptions.throwIfFatal(t);
        buffer = null;
        if (upstream == null) {
            EmptyDisposable.error(t, downstream);
        } else {
            upstream.dispose();
            downstream.onError(t);
        }
        return false;
    }

    buffer = b;

    return true;
}
```
这里的bufferSupplier的call()方法实现如下，就是创建了一个ArrayList

```java
@Override
public List<Object> call() throws Exception {
    return new ArrayList<Object>();
}
```
接着我们看下onNext方法：

```java
@Override
public void onNext(T t) {
    U b = buffer;
    if (b != null) {
        b.add(t);

        if (++size >= count) {
            downstream.onNext(b);

            size = 0;
            createBuffer();
        }
    }
}
```
显而易见，一旦ArrayList的size大于等于count，那就调用观察者的onNext()，观察者中就能接收到这个数组啦，这样就实现了数据的切片功能～
## map
![map](https://img-blog.csdnimg.cn/20190327104548826.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3NTMyMTQw,size_16,color_FFFFFF,t_70)
map是对订阅数据的转换，可以自定义转换规则，使用如下：

```java
Observable.just(1, 2, 3)
    .map(x -> x * x)
    .subscribe(System.out::println);
输出:
1
4
9
```
再来看源码整个的调用链：

```java
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
    ObjectHelper.requireNonNull(mapper, "mapper is null");
    return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
}
@Override
public void subscribeActual(Observer<? super U> t) {
    source.subscribe(new MapObserver<T, U>(t, function));
}
@Override
public void onNext(T t) {
    if (done) {
        return;
    }

    if (sourceMode != NONE) {
        downstream.onNext(null);
        return;
    }

    U v;

    try {
        v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
    } catch (Throwable ex) {
        fail(ex);
        return;
    }
    downstream.onNext(v);
}
```
可以看到，其实关键就在于mapper的apply，mapper就是之前分析过的Function函数，是一个数据转换的接口，具体实现就是我们自己重写的这个lambda表达式中的操作，返回值被丢到了观察者的onNext方法中，这样我们就完成了数据转换的操作，很简单，不多说了。
## flatMap
![flatMap](https://img-blog.csdnimg.cn/20190327104944547.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3NTMyMTQw,size_16,color_FFFFFF,t_70)
flatMap是把一个被观察者生产的多个数据转换到多个被观察者中，然后再把这些被观察者压入到单一的一个被观察者中。
基本使用：

```java
Observable.just("A", "B", "C")
    .flatMap(a -> {
        return Observable.intervalRange(1, 3, 0, 1, TimeUnit.SECONDS)
                .map(b -> '(' + a + ", " + b + ')');
    })
    .blockingSubscribe(System.out::println);

// prints (not necessarily in this order):
// (A, 1)
// (C, 1)
// (B, 1)
// (A, 2)
// (C, 2)
// (B, 2)
// (A, 3)
// (C, 3)
// (B, 3)
```
源代码：

```java
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
public final <R> Observable<R> flatMap(Function<? super T, ? extends ObservableSource<? extends R>> mapper) {
    return flatMap(mapper, false);
}
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
public final <R> Observable<R> flatMap(Function<? super T, ? extends ObservableSource<? extends R>> mapper, boolean delayErrors) {
    return flatMap(mapper, delayErrors, Integer.MAX_VALUE);
}
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
public final <R> Observable<R> flatMap(Function<? super T, ? extends ObservableSource<? extends R>> mapper, boolean delayErrors, int maxConcurrency) {
    return flatMap(mapper, delayErrors, maxConcurrency, bufferSize());
}
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
public final <R> Observable<R> flatMap(Function<? super T, ? extends ObservableSource<? extends R>> mapper,
        boolean delayErrors, int maxConcurrency, int bufferSize) {
    ObjectHelper.requireNonNull(mapper, "mapper is null");
    ObjectHelper.verifyPositive(maxConcurrency, "maxConcurrency");
    ObjectHelper.verifyPositive(bufferSize, "bufferSize");
    if (this instanceof ScalarCallable) {
        @SuppressWarnings("unchecked")
        T v = ((ScalarCallable<T>)this).call();
        if (v == null) {
            return empty();
        }
        return ObservableScalarXMap.scalarXMap(v, mapper);
    }
    return RxJavaPlugins.onAssembly(new ObservableFlatMap<T, R>(this, mapper, delayErrors, maxConcurrency, bufferSize));
}
```
再来看到ObservableFlatMap:

```java
@Override
public void subscribeActual(Observer<? super U> t) {

    if (ObservableScalarXMap.tryScalarXMapSubscribe(source, t, mapper)) {
        return;
    }

    source.subscribe(new MergeObserver<T, U>(t, mapper, delayErrors, maxConcurrency, bufferSize));
}
```
最终订阅的是MergeObserver，我们看下onNext()方法：

```java
@Override
public void onNext(T t) {
    // safeguard against misbehaving sources
    if (done) {
        return;
    }
    ObservableSource<? extends U> p;
    try {
        p = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper returned a null ObservableSource");
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        upstream.dispose();
        onError(e);
        return;
    }

    if (maxConcurrency != Integer.MAX_VALUE) {
        synchronized (this) {
            if (wip == maxConcurrency) {
                sources.offer(p);
                return;
            }
            wip++;
        }
    }

    subscribeInner(p);
}
```
这边的mapper还是我们传入的Function接口的实现。得到的新的Observable对象存在了局部变量p中，接着判断最大并发数量是否是Integer.MAX_VALUE，这里因为我们并没有指定，所以是默认值Integer.MAX_VALUE，所以直接进入到subscribeInner()方法：

```java
void subscribeInner(ObservableSource<? extends U> p) {
    for (;;) {
        if (p instanceof Callable) {
            if (tryEmitScalar(((Callable<? extends U>)p)) && maxConcurrency != Integer.MAX_VALUE) {
                boolean empty = false;
                synchronized (this) {
                    p = sources.poll();
                    if (p == null) {
                        wip--;
                        empty = true;
                    }
                }
                if (empty) {
                    drain();
                    break;
                }
            } else {
                break;
            }
        } else {
            InnerObserver<T, U> inner = new InnerObserver<T, U>(this, uniqueId++);
            if (addInner(inner)) {
                p.subscribe(inner);
            }
            break;
        }
    }
}
```
这边判断了我们的Observable是否实现了Callable接口，一般情况下，我们这里以false执行，也就是创建了一个InnerObserver，然后添加到数组中:

```java
boolean addInner(InnerObserver<T, U> inner) {
    for (;;) {
        InnerObserver<?, ?>[] a = observers.get();
        if (a == CANCELLED) {
            inner.dispose();
            return false;
        }
        int n = a.length;
        InnerObserver<?, ?>[] b = new InnerObserver[n + 1];
        System.arraycopy(a, 0, b, 0, n);
        b[n] = inner;
        if (observers.compareAndSet(a, b)) {
            return true;
        }
    }
}
```
这里就是把observers原子性地增加一个元素，就是我们新创建地InnerObserver。接着回到前一个方法，就是完成了对这个InnerObserver的订阅工作，接下来，我们看看这个InnerObserver
的onNext()方法：

```java
@Override
public void onNext(U t) {
    if (fusionMode == QueueDisposable.NONE) {
        parent.tryEmit(t, this);
    } else {
        parent.drain();
    }
}
```
这边会执行MergeObserver中的drain()方法：

```java
void drain() {
    if (getAndIncrement() == 0) {
         drainLoop();
     }
 }

 void drainLoop() {
     final Observer<? super U> child = this.downstream;
     int missed = 1;
     for (;;) {
         if (checkTerminate()) {
             return;
         }
         SimplePlainQueue<U> svq = queue;

         if (svq != null) {
             for (;;) {
                 if (checkTerminate()) {
                     return;
                 }

                 U o = svq.poll();

                 if (o == null) {
                     break;
                 }

                 child.onNext(o);
             }
         }

         boolean d = done;
         svq = queue;
         InnerObserver<?, ?>[] inner = observers.get();
         int n = inner.length;

         int nSources = 0;
         if (maxConcurrency != Integer.MAX_VALUE) {
             synchronized (this) {
                 nSources = sources.size();
             }
         }

         if (d && (svq == null || svq.isEmpty()) && n == 0 && nSources == 0) {
             Throwable ex = errors.terminate();
             if (ex != ExceptionHelper.TERMINATED) {
                 if (ex == null) {
                     child.onComplete();
                 } else {
                     child.onError(ex);
                 }
             }
             return;
         }

         int innerCompleted = 0;
         if (n != 0) {
             long startId = lastId;
             int index = lastIndex;

             if (n <= index || inner[index].id != startId) {
                 if (n <= index) {
                     index = 0;
                 }
                 int j = index;
                 for (int i = 0; i < n; i++) {
                     if (inner[j].id == startId) {
                         break;
                     }
                     j++;
                     if (j == n) {
                         j = 0;
                     }
                 }
                 index = j;
                 lastIndex = j;
                 lastId = inner[j].id;
             }

             int j = index;
             sourceLoop:
             for (int i = 0; i < n; i++) {
                 if (checkTerminate()) {
                     return;
                 }

                 @SuppressWarnings("unchecked")
                 InnerObserver<T, U> is = (InnerObserver<T, U>)inner[j];
                 SimpleQueue<U> q = is.queue;
                 if (q != null) {
                     for (;;) {
                         U o;
                         try {
                             o = q.poll();
                         } catch (Throwable ex) {
                             Exceptions.throwIfFatal(ex);
                             is.dispose();
                             errors.addThrowable(ex);
                             if (checkTerminate()) {
                                 return;
                             }
                             removeInner(is);
                             innerCompleted++;
                             j++;
                             if (j == n) {
                                 j = 0;
                             }
                             continue sourceLoop;
                         }
                         if (o == null) {
                             break;
                         }

                         child.onNext(o);

                         if (checkTerminate()) {
                             return;
                         }
                     }
                 }

                 boolean innerDone = is.done;
                 SimpleQueue<U> innerQueue = is.queue;
                 if (innerDone && (innerQueue == null || innerQueue.isEmpty())) {
                     removeInner(is);
                     if (checkTerminate()) {
                         return;
                     }
                     innerCompleted++;
                 }

                 j++;
                 if (j == n) {
                     j = 0;
                 }
             }
             lastIndex = j;
             lastId = inner[j].id;
         }

         if (innerCompleted != 0) {
             if (maxConcurrency != Integer.MAX_VALUE) {
                 while (innerCompleted-- != 0) {
                     ObservableSource<? extends U> p;
                     synchronized (this) {
                         p = sources.poll();
                         if (p == null) {
                             wip--;
                             continue;
                         }
                     }
                     subscribeInner(p);
                 }
             }
             continue;
         }
         missed = addAndGet(-missed);
         if (missed == 0) {
             break;
         }
     }
 }
```
这里主要是算法代码，基本逻辑就是把所有的InnerObserver的队列中的消息全部发送给我们最终的观察者，这里需要注意，由于并发的原因，我们添加数据是不能保证顺序的，所以在观察者中的输出也是不能保证顺序的。
**结束语**
由于篇幅的原因，今天我们就先介绍这些操作符。大家可以看到，像flatMap这种操作符，在基本执行流程上是和其他操作符没有什么区别的，但是它是一个比较复杂的操作符，代码涉及的数据结构算法的代码相对之前分析的其他操作符和执行流程来说要多很多，大家感兴趣的可以自己去看源码再深入研究。
