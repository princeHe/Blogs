@[TOC](RxJava源码阅读理解系列（六）)
# 背压
在前两篇中，我们分析了几个常用的操作符，其他的操作符实现原理也都是大同小异，就不再多做分析了。今天我们开始讲RxJava2中新增的背压。
什么是背压？我们看下官方文档的解释：
Backpressure is when in an Flowable processing pipeline, some asynchronous stages can't process the values fast enough and need a way to tell the upstream producer to slow down.
翻译过来就是说：背压是指在可流处理管道中，一些异步级不能足够快地处理这些值，需要一种方法来通知上游生产者放慢速度。简单的说就是生产者生产数据的速度超过了消费者处理数据的速度，导致越来越多的数据来不及处理会被积压。
话不多说，直接上代码：

```java
Flowable.create((FlowableOnSubscribe<Integer>) emitter -> {
    for (int i = 0; i < 1_000_000; i++) {
        emitter.onNext(i);
    }
    emitter.request(1000);
}, BackpressureStrategy.BUFFER)
        .subscribe(new Subscriber<Integer>() {
            @Override
            public void onSubscribe(Subscription s) {
                
            }

            @Override
            public void onNext(Integer integer) {
                try {
                    Thread.sleep(100);
                    System.out.println(String.valueOf(integer));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            @Override
            public void onError(Throwable t) {

            }

            @Override
            public void onComplete() {

            }
        });
```
这里我们看到create方法的第二个参数BackpressureStrategy背压策略
```java
public enum BackpressureStrategy {
    /**
     * OnNext events are written without any buffering or dropping.
     * Downstream has to deal with any overflow.
     * <p>Useful when one applies one of the custom-parameter onBackpressureXXX operators.
     */
    MISSING,
    /**
     * Signals a MissingBackpressureException in case the downstream can't keep up.
     */
    ERROR,
    /**
     * Buffers <em>all</em> onNext values until the downstream consumes it.
     */
    BUFFER,
    /**
     * Drops the most recent onNext value if the downstream can't keep up.
     */
    DROP,
    /**
     * Keeps only the latest onNext value, overwriting any previous value if the
     * downstream can't keep up.
     */
    LATEST
}
```
我们看到这个策略枚举类：

 - MISSING：从命名上看，上游什么都不做。注释中也解释的很明白：OnNext事件的编写没有任何缓冲或删除。下游必须处理任何溢流。
 - ERROR：如果下游来不及处理数据，就抛出一个MissingBackpressureException异常。
 - BUFFER：把来不及消费的数据存入缓冲区。
 - DROP：如果下游来不及处理数据，就把最近收到的数据丢弃。
 - LATEST：只保留最新的，如果下数据游无法跟上，则覆盖以前的任何值。

看完了策略，我们继续回到代码的执行流程中：

```java
@CheckReturnValue
@NonNull
@BackpressureSupport(BackpressureKind.SPECIAL)
@SchedulerSupport(SchedulerSupport.NONE)
public static <T> Flowable<T> create(FlowableOnSubscribe<T> source, BackpressureStrategy mode) {
    ObjectHelper.requireNonNull(source, "source is null");
    ObjectHelper.requireNonNull(mode, "mode is null");
    return RxJavaPlugins.onAssembly(new FlowableCreate<T>(source, mode));
}
```
这边创建了一个FlowableCreate，我们看下他的订阅方法：
```java
@Override
public void subscribeActual(Subscriber<? super T> t) {
    BaseEmitter<T> emitter;

    switch (backpressure) {
    case MISSING: {
        emitter = new MissingEmitter<T>(t);
        break;
    }
    case ERROR: {
        emitter = new ErrorAsyncEmitter<T>(t);
        break;
    }
    case DROP: {
        emitter = new DropAsyncEmitter<T>(t);
        break;
    }
    case LATEST: {
        emitter = new LatestAsyncEmitter<T>(t);
        break;
    }
    default: {
        emitter = new BufferAsyncEmitter<T>(t, bufferSize());
        break;
    }
    }

    t.onSubscribe(emitter);
    try {
        source.subscribe(emitter);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        emitter.onError(ex);
    }
}
```
这里我们以最经典的策略BUFFER缓冲区来进行分析，这里会新建一个BufferAsyncEmitter，另一个参数bufferSize()就是我们设置的缓冲区大小，默认情况下是128。我们看看它内部onNext()的执行逻辑：

```java
@Override
public void onNext(T t) {
    if (done || isCancelled()) {
        return;
    }

    if (t == null) {
        onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
        return;
    }
    queue.offer(t);
    drain();
}
```
我们看到在onNext()中首先把需要发射的数据存到了队列中，接着调用了drain()方法：

```java
void drain() {
    if (wip.getAndIncrement() != 0) {
        return;
    }

    int missed = 1;
    final Subscriber<? super T> a = downstream;
    final SpscLinkedArrayQueue<T> q = queue;

    for (;;) {
        long r = get();
        long e = 0L;

        while (e != r) {
            if (isCancelled()) {
                q.clear();
                return;
            }

            boolean d = done;

            T o = q.poll();

            boolean empty = o == null;

            if (d && empty) {
                Throwable ex = error;
                if (ex != null) {
                    error(ex);
                } else {
                    complete();
                }
                return;
            }

            if (empty) {
                break;
            }

            a.onNext(o);

            e++;
        }

        if (e == r) {
            if (isCancelled()) {
                q.clear();
                return;
            }

            boolean d = done;

            boolean empty = q.isEmpty();

            if (d && empty) {
                Throwable ex = error;
                if (ex != null) {
                    error(ex);
                } else {
                    complete();
                }
                return;
            }
        }

        if (e != 0) {
            BackpressureHelper.produced(this, e);
        }

        missed = wip.addAndGet(-missed);
        if (missed == 0) {
            break;
        }
    }
}
```
在while的循环条件中比较了e和这个emitter自身的值，e是外层循环中的一个计数器，用来计算这个外层循环中发送消息的次数，r是这个emitter自身的值，默认情况下是0，在父类BaseEmitter中有一个request方法可以设置它的值：

```java
@Override
public final void request(long n) {
    if (SubscriptionHelper.validate(n)) {
        BackpressureHelper.add(this, n);
        onRequested();
    }
}
```
这个方法是在Subscription接口中定义的，我们看下这个值的作用：

```java
/**
 * No events will be sent by a {@link Publisher} until demand is signaled via this method.
 * <p>
 * It can be called however often and whenever needed—but the outstanding cumulative demand must never exceed Long.MAX_VALUE.
 * An outstanding cumulative demand of Long.MAX_VALUE may be treated by the {@link Publisher} as "effectively unbounded".
 * <p>
 * Whatever has been requested can be sent by the {@link Publisher} so only signal demand for what can be safely handled.
 * <p>
 * A {@link Publisher} can send less than is requested if the stream ends but
 * then must emit either {@link Subscriber#onError(Throwable)} or {@link Subscriber#onComplete()}.
 * 
 * @param n the strictly positive number of elements to requests to the upstream {@link Publisher}
 */
public void request(long n);
```
好了，这里的文档说的很清楚了，只有调用了这个方法上游才会发送事件。结合我们上面看到的while循环中的判断代码，也就对应上了，因为默认情况下r=0,e=0，所以是不会进入循环去发送事件的，当我们调用了request后，我们再来继续分析这段循环代码：如果事件发送完成了或者发送的消息数量达到了我们设置的request的值，就退出while循环；接着判断是否已经全部消费完了上游事件，如果消费完了就回调消费者的onComplete()方法；接着看缓冲区中是否还有数据，如果还有数据，就进入下面的代码：

```java
/**
 * Atomically subtract the given number (positive, not validated) from the target field unless it contains Long.MAX_VALUE.
 * @param requested the target field holding the current requested amount
 * @param n the produced element count, positive (not validated)
 * @return the new amount
 */
public static long produced(AtomicLong requested, long n) {
    for (;;) {
        long current = requested.get();
        if (current == Long.MAX_VALUE) {
            return Long.MAX_VALUE;
        }
        long update = current - n;
        if (update < 0L) {
            RxJavaPlugins.onError(new IllegalStateException("More produced than requested: " + update));
            update = 0L;
        }
        if (requested.compareAndSet(current, update)) {
            return update;
        }
    }
}
```
这段代码就是把emitter的值做一个CAS操作改为原先的值减去已经发送事件的数量。结果也就是剩余的发送次数。
再接下来，把wip值恢复为0并退出外层循环，直到下次的emitter发送onNext()事件。
整个流程分析下来，我们对于缓冲策略的背压逻辑应该就清楚了，由于采用的是队列的数据结构，因此来不及消费的事件会从缓冲区队列的头部被挤出去而能够保存较新的事件。
**结束语**
对缓冲策略的背压分析的就差不多了，我们可以看到，所有的策略都是依赖于数据结构和算法的，有兴趣的同学可以研究研究其他被压策略的代码，对于数据结构和算法，有兴趣的同学也可以花点时间研究研究，毕竟要写出高质量的代码，这些底层的技能还是要掌握的。

