@[TOC](RxJava源码阅读理解系列（三）)
# RxJava的线程切换
在Android开发中，我们经常需要用到主线程与子线程之间的切换操作，Android原生的实现方式是使用Handler。在RxJava中我们可以很方便的实现这样的操作：subscribeOn(Scheduler scheduler)指定订阅的线程，observeOn(Scheduler scheduler)指定观察的线程。
那我们现在就进入源码一探究竟，我们找到入口函数：

```java
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> subscribeOn(Scheduler scheduler) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}
```
和往常一样是个Hook方法，这里返回了一个ObservableSubscribeOn对象，它也是Observable
的实现类，我们进到ObservableSubscribeOn中：


```java
@Override
public void subscribeActual(final Observer<? super T> observer) {
    final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);

    observer.onSubscribe(parent);

    parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
}
```
我们看到，前两行和之前基本执行流程分析的一样，是一个Observer订阅了一个Observable。
继续往下看，setDisposable(Disposable disposable)方法

```java
void setDisposable(Disposable d) {
	DisposableHelper.setOnce(this, d);
}

public static boolean setOnce(AtomicReference<Disposable> field, Disposable d) {
    ObjectHelper.requireNonNull(d, "d is null");
    if (!field.compareAndSet(null, d)) {
        d.dispose();
        if (field.get() != DISPOSED) {
            reportDisposableSet();
        }
        return false;
    }
    return true;
}
```
看起来只是做了一次CAS操作，把参数中的disposable对象替换到SubscribeOnObserver所持有的原子引用的Disposable对象中去。
那我们看下替换进来的的disposable：

```java
/**
 *Schedules the given task on this Scheduler without any time delay.
 */
@NonNull
public Disposable scheduleDirect(@NonNull Runnable run) {
    return scheduleDirect(run, 0L, TimeUnit.NANOSECONDS);
}
```
通过代码和注释，我们看到在这里是要立即执行调度器上的任务，这是一个什么任务呢？

```java
final class SubscribeTask implements Runnable {
    private final SubscribeOnObserver<T> parent;

    SubscribeTask(SubscribeOnObserver<T> parent) {
        this.parent = parent;
    }

    @Override
    public void run() {
        source.subscribe(parent);
    }
}
```
这里就是一个订阅任务，我们再接着往下看上面方法的具体实现：

```java
@NonNull
public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
    final Worker w = createWorker();

    final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

    DisposeTask task = new DisposeTask(decoratedRun, w);

    w.schedule(task, delay, unit);

    return task;
}
```
好像每一行都没怎么见过，我们一行一行看：
- 首先第一行，创建一个Worker：

	```java
	@NonNull
	public abstract Worker createWorker();
	```
	这里是个抽象方法，实现需要去看具体Scheduler的实现，我们拿IoScheduler来看：
	
	```java
	@NonNull
	@Override
	public Worker createWorker() {
	    return new EventLoopWorker(pool.get());
	}
	```
	是一个EventLoopWorker，从名字上看，是事件轮询worker。
- 再来看第二行：装饰的runnable，其实，也是通过Function接口把这个runnable替换掉，默认就是返回原来的runnable。
- 第三行，创建了一个disposeTask。
- 第四行，执行worker的schedule方法，这应该是关键了，我们还是以IoScheduler看下实现：
	```java
	@NonNull
    @Override
    public Disposable schedule(@NonNull Runnable action, long delayTime, @NonNull TimeUnit unit) {
        if (tasks.isDisposed()) {
            // don't schedule, we are unsubscribed
            return EmptyDisposable.INSTANCE;
        }

        return threadWorker.scheduleActual(action, delayTime, unit, tasks);
    }
    @NonNull
    public ScheduledRunnable scheduleActual(final Runnable run, long delayTime, @NonNull TimeUnit unit, @Nullable DisposableContainer parent) {
        Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

        ScheduledRunnable sr = new ScheduledRunnable(decoratedRun, parent);

        if (parent != null) {
            if (!parent.add(sr)) {
                return sr;
            }
        }

        Future<?> f;
        try {
            if (delayTime <= 0) {
                f = executor.submit((Callable<Object>)sr);
            } else {
                f = executor.schedule((Callable<Object>)sr, delayTime, unit);
            }
            sr.setFuture(f);
        } catch (RejectedExecutionException ex) {
            if (parent != null) {
                parent.remove(sr);
            }
            RxJavaPlugins.onError(ex);
        }

        return sr;
    }
	```
看到这里大家应该都恍然大悟了，这里就是把之前我们分析到的Runnable丢到了线程池里去执行，Runnable中跑的就是Observable的subscribe方法，所以subscribe后所有的操作就都是在这个线程池里执行啦，这样切换订阅线程的分析就完成了。
那么接下来我们就要分析切换观察线程了，同样，我们从入口函数observeOn()开始:

```java
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> observeOn(Scheduler scheduler) {
    return observeOn(scheduler, false, bufferSize());
}
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    ObjectHelper.verifyPositive(bufferSize, "bufferSize");
    return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
}
```
😂和往常一样是个Hook函数，来看ObservableObserveOn的subscribeActual()函数

```java
@Override
protected void subscribeActual(Observer<? super T> observer) {
    if (scheduler instanceof TrampolineScheduler) {
        source.subscribe(observer);
    } else {
        Scheduler.Worker w = scheduler.createWorker();

        source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
    }
}
```
我们以非TrampolineScheduler为例吧，这里是重新订阅了ObserveOnObserver，这个Observer实现中的onNext()方法如下：

```java
@Override
public void onNext(T t) {
    if (done) {
        return;
    }

    if (sourceMode != QueueDisposable.ASYNC) {
        queue.offer(t);
    }
    schedule();
}
```
关键应该就是这里的schedule()方法了，我们继续往下看

```java
void schedule() {
    if (getAndIncrement() == 0) {
        worker.schedule(this);
    }
}
```
看到这里是不是觉得似曾相识？没错，和刚才分析订阅线程切换的实现一样，这里就不再分析了，想要丢到哪个线程池就丢到哪个线程池，就是这么easy。
**结束语**
好啦，今天我们就基本上把线程切换的逻辑实现搞清楚了，其实我们分析大型框架源码的时候，抛开细枝末节的一些健壮性代码，只关注他的基本脉络，应该还是很清晰的～
最近在考虑一个问题，我是否需要补充一些执行流程图和UML设计图上来让大家更好的理解这个源码分析系列呢？有需要的同学可以留言，或者有建议也可以提～




	

	


