@[TOC](RxJava源码阅读理解系列（二）)
# RXJava订阅事件取消的源码分析
我们知道，在RxJava中需要取消一个事件的订阅需要用到Disposeable对象，就像这样：

```kotlin
Observable.create<Int> { emitter ->
 	emitter.onNext(1)
    emitter.onNext(2)
    emitter.onNext(3)
}.subscribe(object : Observer<Int> {
    override fun onComplete() {
    }

    override fun onSubscribe(d: Disposable) {
        disposable = d
    }

    override fun onNext(t: Int) {
        LogUtils.e(t)
        if (t == 1) disposable?.dispose()
    }

    override fun onError(e: Throwable) {
    }

})
```
像上面的代码这样，我们打印出1以后，事件流被取消了订阅，2和3就不打印了。我们来看看RxJava是怎么做的吧。
```java
@Override
protected void subscribeActual(Observer<? super T> observer) {
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);
    observer.onSubscribe(parent);

    try {
        source.subscribe(parent);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        parent.onError(ex);
    }
}
```
回到我们订阅的代码，这里的onSubscribe方法中传递了我们用到的Disposable参数，可以看到是一个发射器对象，它是一个实现了Disposable的接口：

```java
static final class CreateEmitter<T>
    extends AtomicReference<Disposable>
    implements ObservableEmitter<T>, Disposable {
    /*暂时不看实现*/
}
```
好，既然我们订阅时穿进去的就是这个实现类，那么就可以研究下它内部取消订阅的时候都做了什么事：

```java
@Override
public void dispose() {
    DisposableHelper.dispose(this);
}
```
再来看下DisposableHelper的dispose方法

```java
public static boolean dispose(AtomicReference<Disposable> field) {
    Disposable current = field.get();
    Disposable d = DISPOSED;
    if (current != d) {
        current = field.getAndSet(d);
        if (current != d) {
            if (current != null) {
                current.dispose();
            }
            return true;
        }
    }
    return false;
}
```
AtomicReference是线程安全的对象引用，这里是一个线程安全的Disposable对象，也就是我们的发射器。看到这里还不知道DisposeHelper是个啥，我们看下：

```java
public enum DisposableHelper implements Disposable {
	/**
	 * The singleton instance representing a terminal, disposed state, don't leak it.
	 */
	DISPOSED
	;
	/*省略*/
	@Override
    public boolean isDisposed() {
        return true;
    }
}
```
原来这是个枚举类，那我们基本可以猜测出来，这里的DISPOSED就是一个单例，用来表示已经调用过dispose()方法的Disposable对象，为什么这么说？我们可以看到它重写了isDisposed()方法，返回值为true，其实官方的注释也已经很好的证明了我们的猜测。
这样我们就可以回过来分析上面的dispose()方法了，他首先对比传入进来的disposable对象是不是这个枚举单例，如果不是那就把它的Disposable引用指向这个单例，并且返回成功，如果已经是这个单例了，那就不管他。那么我们的CreateEmitter就是这个已经dispose掉的DISPOSED啦。
那可能你问会我们这样做怎么就能做到取消订阅的效果了呢？别急，我们接着看事件流的生产过程：

```java
@Override
public void onNext(T t) {
    if (t == null) {
        onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
        return;
    }
    if (!isDisposed()) {
        observer.onNext(t);
    }
}
```
这样就恍然大悟啦，生产者在生产数据的时候，首先会看是否已经被取消掉。因为刚才我们已经分析过，单例的DISPOSED对象已经重写了isDisposed()方法，是永远返回true的，所以说只要我们的生产者也就是CreateEmitter中的Disposable对象引用被指向了这个单例对象，那就实现了事件的取消。

**结束语**
通过单例结合AtomicReference解决线程安全问题是一个非常优雅的做法。是不是感觉又学到了新的姿势，美滋滋。今天的分析就是这么多啦，下回见～

