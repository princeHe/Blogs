@[TOC](RxJava源码阅读理解系列（一）)

# RxJava的事件流执行源码的分析
## RxJava的基本介绍
[官方文档](https://github.com/ReactiveX/RxJava/wiki)

## RxJava的基本用法
我们首先来看一下RxJava的基本用法：
```kotlin
Observable.create<Int> { emitter ->
	emitter.onNext(1)
	emitter.onNext(2)
	emitter.onNext(3)
}.subscribe { data -> 
	LogUtils.e(data)
}
```
首先我们调用Observable的静态的create方法创建一个Observable对象，然后再去订阅一个Observer对象，我们的Observer对象中就可以收到Observable对象中Emitter对象发出来的消息了。一个标准的生产者消费者模型。

## 基本流程源码分析
好了，我们进到源码去一探究竟，看看RxJava是怎么把我们发射器中发射的内容输送到观察者中去的。

```java
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
	ObjectHelper.requireNonNull(source, "source is null");
	return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```
我们看到，这个方法只做了一个入参的非空校验，实际工作丢给了RxJavaPlugin，那我们继续往下看：

```java
@NonNull
public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
	Function<? super Observable, ? extends Observable> f = onObservableAssembly;
	if (f != null) {
		return apply(f, source);
	}
	return source;
}
```
emmm？这里的Function是啥？去看一下Function的定义:

```java
public interface Function<T, R> {
    /**
     * Apply some calculation to the input value and return some other value.
     * @param t the input value
     * @return the output value
     * @throws Exception on error
     */
    R apply(@NonNull T t) throws Exception;
}
```
哦哦，原来是这样，就是把T转换成R，那我们看看这里这个Function的具体实现，我们找到onObservableAssembly:

```java
@Nullable
static volatile Function<? super Observable, ? extends Observable> onObservableAssembly;
```
哎，好像只有声明，全局搜索一下看看有没有具体实现，好像没有，那好吧，可能这是RxJavaPlugins提供的暴露给调用者的一个替换Observable的接口吧。。默认情况都是空的，就直接返回我们调用create方法时传入的ObservableCreate。
那么我们再来看这个ObservableCreate，它是Observable的一个子类，关键代码：

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
在我们调用Observable的subscribe(Observer)的时候最终会执行到这个方法中，我们首先创建一个用于发射数据的Emitter，然后我们的ObservableOnSubscribe去订阅这个发射器，，也就会执行我们在订阅回调方法中Emitter的onNext()方法了。
接下来就是最后一步，看下这个Emitter实现类CreateEmitter的具体实现，上代码：

```java
 CreateEmitter(Observer<? super T> observer) {
     this.observer = observer;
 }

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
可以看到，在CreateEmitter的onNext()方法中如果数据不是空并且没有取消订阅的情况下，就会执行Observer的onNext()方法啦，上游Emitter发射的数据就传递到了下游Observer中来了。

**结束语**
今天对于RxJava事件流传递的源码分析就结束了，其实很好理解。有兴趣的同学可以自己阅读下源码，个人觉得对我们阅读源码的能力、抽象设计的思维都是很有帮助的。
接下来我还会继续分享阅读RxJava更高级功能的源码分析的内容，包括取消事件订阅、各种操作符，线程调度，背压等等。


