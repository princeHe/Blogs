@[TOC](RxJava源码阅读理解系列（五）)
# 操作符
今天我们继续来阅读RxJava中常用的操作符的源码。
## concatMap
concatMap和flatMap基本功能类似，区别在于concatMap接收到的数据源顺序是有序的而flatMap是无序的。
因为在flatMap中，订阅操作订阅MergeObserver的时候并发地创建了多个InnerObserver并存储在了ArrayDeque这个非线程安全的数据结构中，而concatMap中并没有这个操作，只有一个SourceObserver中用队列存储了上游的数据源，因此能保证数据的顺序。整个操作流程和flatMap如出一辙，就不多做分析了。
## groupBy
![groupBy](https://img-blog.csdnimg.cn/20190328112110582.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3NTMyMTQw,size_16,color_FFFFFF,t_70)
groupBy的官方文档说明是这样：把一个被观察者分成多个被观察者，每个被观察者发射原始被观察者中发射数据的一个子集。
我们来看一下用法：

```java
Observable<String> animals = Observable.just(
    "Tiger", "Elephant", "Cat", "Chameleon", "Frog", "Fish", "Turtle", "Flamingo");

animals.groupBy(animal -> animal.charAt(0), String::toUpperCase)
    .concatMapSingle(Observable::toList)
    .subscribe(System.out::println);

// prints:
// [TIGER, TURTLE]
// [ELEPHANT]
// [CAT, CHAMELEON]
// [FROG, FISH, FLAMINGO]
```
我们可以看到上面的例子，我们成功地把所有首字母一样的数据源归为了一类。接下来上代码：

```java
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
public final <K, V> Observable<GroupedObservable<K, V>> groupBy(Function<? super T, ? extends K> keySelector,
        Function<? super T, ? extends V> valueSelector) {
    return groupBy(keySelector, valueSelector, false, bufferSize());
}

@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
public final <K, V> Observable<GroupedObservable<K, V>> groupBy(Function<? super T, ? extends K> keySelector,
        Function<? super T, ? extends V> valueSelector,
        boolean delayError, int bufferSize) {
    ObjectHelper.requireNonNull(keySelector, "keySelector is null");
    ObjectHelper.requireNonNull(valueSelector, "valueSelector is null");
    ObjectHelper.verifyPositive(bufferSize, "bufferSize");

    return RxJavaPlugins.onAssembly(new ObservableGroupBy<T, K, V>(this, keySelector, valueSelector, bufferSize, delayError));
}
```
我们看到ObservableGroupBy的subscribeActual

```java
@Override
public void subscribeActual(Observer<? super GroupedObservable<K, V>> t) {
    source.subscribe(new GroupByObserver<T, K, V>(t, keySelector, valueSelector, bufferSize, delayError));
}
```
继续看GroupByObserver的onNext():

```java
@Override
public void onNext(T t) {
    K key;
    try {
        key = keySelector.apply(t);
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        upstream.dispose();
        onError(e);
        return;
    }

    Object mapKey = key != null ? key : NULL_KEY;
    GroupedUnicast<K, V> group = groups.get(mapKey);
    if (group == null) {
        // if the main has been cancelled, stop creating groups
        // and skip this value
        if (cancelled.get()) {
            return;
        }

        group = GroupedUnicast.createWith(key, bufferSize, this, delayError);
        groups.put(mapKey, group);

        getAndIncrement();

        downstream.onNext(group);
    }

    V v;
    try {
        v = ObjectHelper.requireNonNull(valueSelector.apply(t), "The value supplied is null");
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        upstream.dispose();
        onError(e);
        return;
    }

    group.onNext(v);
}
```
这里就是核心的处理逻辑了：
首先通过keySelector生成key，在我们上面的例子中，是取出字符串的首字母，所以这里的key是所有字符串的首字母。这里我们知道生成的组是保存在了ConcurrentHashMap中：
```java
this.groups = new ConcurrentHashMap<Object, GroupedUnicast<K, V>>();
```
接着我们从这个map中根据生成的key取分组后的Observable，也就是这里的GroupedUnicast，如果还没有，我们就根据这个key取创建一个并把他们保存到map中，接着把这个GroupedUnicast发到我们的观察者中。
紧接着通过valueSelector转换了我们的数据源，并把转换后的数据源传入了GroupedUnicast的onNext()方法：

```java
public void onNext(T t) {
    state.onNext(t);
}
```
这里调用了state的onNext()方法，这里的state是我们在创建GroupedUnicast的时候创建的，从名字上看他应该是保存了GroupedUnicast的状态，我们看到他的onNext()方法：
```java
public void onNext(T t) {
    queue.offer(t);
    drain();
}
```
我们看到这边先把数据丢到了队列中，然后又是调用了这个熟悉的drain()方法：

```java
void drain() {
    if (getAndIncrement() != 0) {
        return;
    }
    int missed = 1;

    final SpscLinkedArrayQueue<T> q = queue;
    final boolean delayError = this.delayError;
    Observer<? super T> a = actual.get();
    for (;;) {
        if (a != null) {
            for (;;) {
                boolean d = done;
                T v = q.poll();
                boolean empty = v == null;

                if (checkTerminated(d, empty, a, delayError)) {
                    return;
                }

                if (empty) {
                    break;
                }

                a.onNext(v);
            }
        }

        missed = addAndGet(-missed);
        if (missed == 0) {
            break;
        }
        if (a == null) {
            a = actual.get();
        }
    }
}
```
这里也都是一些算法的实现，主要的工作就是取出队列中所有的数据并都发送到观察者中。
所以最终的逻辑总结下来就是根据key把所有数据分成多个组，每个组被包装成了一个Observable，外层的Observable发射出这些包装后的Observable，包装过的Observable再发射出这个组中的所有数据。
**结束语**
今天的分析就先到这里，我们今天又分析了一个比较复杂的操作符。我们应该可以发现，复杂一些的操作符对数据结构和算法还是需要有一定的基础的，所以建议大家有空的时候多研究研究常用的数据结构和算法，这样才能设计出强大的软件。
