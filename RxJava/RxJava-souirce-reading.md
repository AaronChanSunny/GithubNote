# RxJava 源码解读笔记

## 概览

基于 `RxJava:1.2.1`，主要从以下四个方面解读：

- 被观察者如何发射数据
- 观察者如何接收到数据
- 如何加工数据
- 调度的原理

## 被观察者如何发射数据

创建 Observable 通常有 3 种方式：just、from、create。

### just

创建一个 Observable 最简单的方式，使用 `just`：
```
Observable
        .just("Hello world")
        .subscribe(word -> {
            System.out.println("Got word " + word);
        });
```

相关源码：

```
// Observable.java
public static <T> Observable<T> just(final T value) {
    return ScalarSynchronousObservable.create(value);        // 1
}

// ScalarSynchronousObservable.java
public static <T> ScalarSynchronousObservable<T> create(T t) {
    return new ScalarSynchronousObservable<T>(t);            // 2
}

protected ScalarSynchronousObservable(final T t) {
    super(RxJavaHooks.onCreate(new JustOnSubscribe<T>(t)));  // 3
    this.t = t;
}
```

这里先不管RxJavaHooks，可以简单理解成 RxJava 做的一层包装，并不影响代码的实际逻辑。看 RxJavaHooks.onCreate() 源码：

```
// RxJavaHooks.java
public static <T> Observable.OnSubscribe<T> onCreate(Observable.OnSubscribe<T> onSubscribe) {
    Func1<Observable.OnSubscribe, Observable.OnSubscribe> f = onObservableCreate;
    if (f != null) {
        return f.call(onSubscribe);
    }
    return onSubscribe;
}
```

返回的还是 `OnSubscribe`，因此从代码逻辑看 `super(RxJavaHooks.onCreate(new JustOnSubscribe<T>(t)))` 和 `super(new JustOnSubscribe<T>(t))` 其实是等价的。这里相当于调用了 `Observable` 的构造方法：

```
// Observable.java
protected Observable(OnSubscribe<T> f) {
    this.onSubscribe = f;
}
```

需要明确一点，Observable 构造方法传入的是一个 OnSubscribe 接口，这是一个回调接口。只有当 Observable#subscribe 时，OnSubscribe#call 方法才会被调用。这一点从 subscribe 部分再作详细说明。

### from

当需要发送的数据项不止一个，而是一个集合，可以使用 `from` 创建 Observable：

```
Observable
        .from(Arrays.asList("Hello", "World"))
        .subscribe(word -> {
            System.out.println("Got word " + word);
        });
```

相关源码：

```
// Observable.java
public static <T> Observable<T> from(Iterable<? extends T> iterable) {
    return create(new OnSubscribeFromIterable<T>(iterable));          // 1
}

// OnSubscribeFromIterable.java
public OnSubscribeFromIterable(Iterable<? extends T> iterable) {      // 2
    if (iterable == null) {
        throw new NullPointerException("iterable must not be null");
    }
    this.is = iterable;
}
```

`OnSubscribeFromIterable` 是 `OnSubscribe` 的一个实现类。

### create

创建 Observable 最基本的方法，create：

```
// Observable.java
public static <T> Observable<T> create(OnSubscribe<T> f) {
    return new Observable<T>(RxJavaHooks.onCreate(f));   // 1
}

protected Observable(OnSubscribe<T> f) {
    this.onSubscribe = f;                                // 2
}
```

## 订阅被观察者

 前面提到过，创建 Observable 时传入的 `OnSubscribe` 接口是一个回调接口。那么，这个什么时候会调用这个回调接口呢？答案是 Observable#subscribe，即 Observable 被订阅的时候。
 
 Observable 提供了多种订阅方式供开发者选择，但是无论使用哪种方式，最终都会走到 `subscribe(Subscriber<? super T> subscriber, Observable<T> observable)`。这里，以 `subscribe(final Action1<? super T> onNext)` 作为例子：
 
 ```
 // Observable.java
 public final Subscription subscribe(final Action1<? super T> onNext) {
    if (onNext == null) {
        throw new IllegalArgumentException("onNext can not be null");
    }
    Action1<Throwable> onError = InternalObservableUtils.ERROR_NOT_IMPLEMENTED;   // 1
    Action0 onCompleted = Actions.empty();                                        // 2
    return subscribe(new ActionSubscriber<T>(onNext, onError, onCompleted));      // 3
}

public final Subscription subscribe(Subscriber<? super T> subscriber) {
    return Observable.subscribe(subscriber, this);                                 // 4
}

// ActionSubscriber.java
public ActionSubscriber(Action1<? super T> onNext, Action1<Throwable> onError, Action0 onCompleted) {
    this.onNext = onNext;
    this.onError = onError;
    this.onCompleted = onCompleted;
}

@Override
public void onNext(T t) {
    onNext.call(t);          // 5
}
@Override
public void onError(Throwable e) {
    onError.call(e);         // 6
}
@Override
public void onCompleted() {
    onCompleted.call();      // 7
}
 ```
 
 首先要明确，Observable#subscribe 方法传入的是一个 `Subscriber`，`Subscriber` 是个什么东西呢？
 
 ```
 public abstract class Subscriber<T> implements Observer<T>, Subscription {
    // 代码省略
 }
 ```
 
 其实，Observable#subscribe 方法传入的就是一个观察者。通过 Observable#subscribe 方法，被观察者和观察者建立起联系。
 
 Observable#subscribe 核心代码：
 
 ```
 static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
    // 省略参数校验代码
    
    // new Subscriber so onStart it
    subscriber.onStart();                                    // 1
    /*
     * See https://github.com/ReactiveX/RxJava/issues/216 for discussion on "Guideline 6.4: Protect calls
     * to user code from within an Observer"
     */
    // if not already wrapped
    if (!(subscriber instanceof SafeSubscriber)) {
        // assign to `observer` so we return the protected version
        subscriber = new SafeSubscriber<T>(subscriber);       // 2
    }
    // The code below is exactly the same an unsafeSubscribe but not used because it would
    // add a significant depth to already huge call stacks.
    try {
        // allow the hook to intercept and/or decorate
        RxJavaHooks.onObservableStart(observable, observable.onSubscribe).call(subscriber);      // 3
        return RxJavaHooks.onObservableReturn(subscriber);
    } catch (Throwable e) {
        // 省略异常捕获代码
    }
}
 ```

1. 调用 Subscriber#onStart 方法。这里需要注意，onStart 方法所在线程就是当前线程。也就是说，Schedules 并不会影响 onStart 所运行的线程；
2. `SafeSubscriber` 是保证一个 `Subscriber` 遵循 [The Observable Contract](http://reactivex.io/documentation/contract.html)；
3. 跳过 RxJavaHooks 逻辑，相当于 observable.onSubscribe#call(subscriber)。

回想一下 Observable 是如何被创建的：

```
 Observable
         .create(new Observable.OnSubscribe<String>() {
             @Override
             public void call(Subscriber<? super String> subscriber) {
                 subscriber.onNext("Hello world");        // 1
                 subscriber.onCompleted();
             }
         });
```

因此，当 observable.onSubscribe#call(subscriber) 被执行时，相当于就开始执行位置 1 的代码逻辑，数据就开始从被观察者发送到观察者了。
