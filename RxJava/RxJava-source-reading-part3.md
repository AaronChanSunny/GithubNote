# RxJava 源码解读笔记（三）

## 1 线程调度

RxJava 中线程的切换非常方便，只需要调用：Observable#subscribeOn() 和 Observable#observeOn() 方法即可。

### 1.1 subscribeOn

简单的例子：

```
public void testSubscribeOn() {
    Observable
            .create(subscriber -> {
                subscriber.onNext("1");     // 1
                subscriber.onCompleted();
            })
            .subscribeOn(Schedulers.io())   // 2
            .subscribe(s -> {
                System.out.println("got " + s + " on " + Thread.currentThread().getName());
            });
}
```

1. 默认情况下，OnSubscribe#call 方法调用的线程和 Observable#subscribe 处在同一个线程；
2. 位置 2 的代码完成了上游 Observable 的执行线程。

相关源码：

```
// Observable.java
public final Observable<T> subscribeOn(Scheduler scheduler) {
    if (this instanceof ScalarSynchronousObservable) {
        return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
    }
    return create(new OperatorSubscribeOn<T>(this, scheduler));   // 1
}

// OperatorSubscribeOn.java
public OperatorSubscribeOn(Observable<T> source, Scheduler scheduler) {
    this.scheduler = scheduler;
    this.source = source;
}
@Override
public void call(final Subscriber<? super T> subscriber) {
    final Worker inner = scheduler.createWorker();               // 2
    subscriber.add(inner);
    inner.schedule(new Action0() {                              
        @Override
        public void call() {
            final Thread t = Thread.currentThread();
            Subscriber<T> s = new Subscriber<T>(subscriber) {
                @Override
                public void onNext(T t) {
                    subscriber.onNext(t);                       // 3
                }
                @Override
                public void onError(Throwable e) {
                    try {
                        subscriber.onError(e);
                    } finally {
                        inner.unsubscribe();
                    }
                }
                @Override
                public void onCompleted() {
                    try {
                        subscriber.onCompleted();
                    } finally {
                        inner.unsubscribe();
                    }
                }
                @Override
                public void setProducer(final Producer p) {
                    subscriber.setProducer(new Producer() {
                        @Override
                        public void request(final long n) {       // 4
                            if (t == Thread.currentThread()) {           
                                p.request(n);
                            } else {
                                inner.schedule(new Action0() {
                                    @Override
                                    public void call() {
                                        p.request(n);
                                    }
                                });
                            }
                        }
                    });
                }
            };
            source.unsafeSubscribe(s);
        }
    });
}
```

1. 同其他操作符一样，这里也是创建一个新的 Observable；
2. 被订阅时，创建 Worker 对象，Worker 实现了 `Subscription`，可以被 add，用于集体取消订阅；
3. 上游 subscriber 收到数据之后，转发个下游；
4. 线程切换的逻辑。如果上游所在线程不是 Io，进行线程切换；如果上游已经在 Io 线程，直接往下通知。

因此，通过 `subscribeOn` 操作符，就能保证上游的线程一定是在 Io 线程中执行。引出大家总结的一条结论：

> `subscribeOn` 影响它上面的调用执行时所在的线程。

如果多次调用 `subscribeOn` 的话，以第一个出现 `subscribeOn` 所指定的线程为准。

### 1.2 observeOn

简单例子：

```
public void testSubscribeOn() {
    Observable
            .create(subscriber -> {
                System.out.println("produce on " + Thread.currentThread().getName());
                subscriber.onNext("1");
                subscriber.onCompleted();
            })
            .subscribeOn(Schedulers.io())
            .observeOn(Schedulers.computation())
            .flatMap(s -> {
                System.out.println("got " + s + " on " + Thread.currentThread().getName());
                return Observable.just(s);
            })
            .subscribe(s -> {
                System.out.println("got " + s + " on " + Thread.currentThread().getName());
            });
}
```

相关源码：

```
// Observable.java
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
    if (this instanceof ScalarSynchronousObservable) {
        return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
    }
    return lift(new OperatorObserveOn<T>(scheduler, delayError, bufferSize));             // 1
}

// OperatorObserveOn.java
@Override
public Subscriber<? super T> call(Subscriber<? super T> child) {
    if (scheduler instanceof ImmediateScheduler) {                // 2
        // avoid overhead, execute directly
        return child;
    } else if (scheduler instanceof TrampolineScheduler) {
        // avoid overhead, execute directly
        return child;
    } else {
        ObserveOnSubscriber<T> parent = new ObserveOnSubscriber<T>(scheduler, child, delayError, bufferSize);
        parent.init();
        return parent;
    }
}


// OperatorObserveOn$ObserveOnSubscriber
@Override
public void onNext(final T t) {
    if (isUnsubscribed() || finished) {
        return;
    }
    if (!queue.offer(NotificationLite.next(t))) {
        onError(new MissingBackpressureException());
        return;
    }
    schedule();                       // 3
}
@Override
public void onCompleted() {
    if (isUnsubscribed() || finished) {
        return;
    }
    finished = true;
    schedule();
}
@Override
public void onError(final Throwable e) {
    if (isUnsubscribed() || finished) {
        RxJavaHooks.onError(e);
        return;
    }
    error = e;
    finished = true;
    schedule();
}
```

1. `observeOn` 通过 `lift` 操作符实现。`lift` 操作符负责接收上游传下来的数据，进行加工再转发；
2. 如果指定的 Scheduler 是 `ImmediateScheduler` 或者 `TrampolineScheduler`，则进行往下游转发；否则，在转发之前还需要做线程切换；
3. 负责线程切换的 Subscriber 类。