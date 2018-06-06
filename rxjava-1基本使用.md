#### create

创建一个简单的Observable
```
Observable.create(new Observable.OnSubscribe<String>() {
        @Override
        public void call(Subscriber<? super String> subscriber) {
            //当Observable.subscribe被调用时，执行
        }
});
```

OnSubscribe：发射T类型的数据，给Action处理，Action的参数为Subscriber，实际上的逻辑就是：OnSubscribe发送数据给Subscriber来处理。
```
public interface Function {
}

public interface Action extends Function {
}

public interface Action1<T> extends Action {
    void call(T var1);
}

public interface OnSubscribe<T> extends Action1<Subscriber<? super T>> {
}
```

create实际上是一个工厂方法：
```
public static <T> Observable<T> create(OnSubscribe<T> f) {
    //hook根据注册的plugin来处理Observable生命周期事件。
    return new Observable<T>(hook.onCreate(f));
}
```

#### 线程

create方法创建一个Observable并且subscribe，默认都是同步在当前线程上执行的
```
Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        for (int i = 0; i < 3; i++) {
            System.out.println("Observable Thread "+Thread.currentThread().getName());
            subscriber.onNext("" + i);
        }
    }
})
.subscribe(new Action1<String>() {
    @Override
    public void call(String s) {
        System.out.println("Subscribe Thread "+Thread.currentThread().getName());
    }
});
```
运行结果：
```
Observable Thread main
Subscribe Thread main
Observable Thread main
Subscribe Thread main
Observable Thread main
Subscribe Thread main
```
注意这里是一个线程同步执行，先调用OnSubscribe，每发射一个Data就会在Subscribe里触发处理。


##### 使用observeOn，指定Subscriber异步执行的线程

```
Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        for (int i = 0; i < 3; i++) {
            System.out.println("Observable Thread "+Thread.currentThread().getName());
            subscriber.onNext("" + i);
        }
    }
})
.observeOn(Schedulers.computation())
.subscribe(new Action1<String>() {
    @Override
    public void call(String s) {
        System.out.println("Subscribe Thread "+Thread.currentThread().getName());
    }
});
```
运行结果：
```
Observable Thread main
Observable Thread main
Observable Thread main
Subscribe Thread RxComputationScheduler-1
Subscribe Thread RxComputationScheduler-1
Subscribe Thread RxComputationScheduler-1
```
可以看到observeOn，会指定subsciribe的Action处理的线程。

##### 使用subscribeOn，指定Observable在指定线程池里去执行
```
Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        for (int i = 0; i < 3; i++) {
            System.out.println("Observable Thread "+Thread.currentThread().getName());
            subscriber.onNext("" + i);
        }
    }
})
.subscribeOn(Schedulers.computation())
.subscribe(new Action1<String>() {
    @Override
    public void call(String s) {
        System.out.println("Subscribe Thread "+Thread.currentThread().getName());
    }
});
```
结果：
```
Observable Thread RxComputationScheduler-1
Subscribe Thread RxComputationScheduler-1
Observable Thread RxComputationScheduler-1
Subscribe Thread RxComputationScheduler-1
Observable Thread RxComputationScheduler-1
Subscribe Thread RxComputationScheduler-1
```
注意：指定了subscribeOn，但是没有指定observeOn，观察者和被观察者仍然是同步执行的，即发射一条Data，立即在当前线程调用subscribe处理逻辑。只指定subscribeOn，和不指定类似，只是一个是当前线程处理，一个是将OnSubscribe和subscribe一起放到指定线程池里去执行。


##### 同时使用subscribeOn和observeOn
```
Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        for (int i = 0; i < 3; i++) {
            System.out.println("Observable Thread "+Thread.currentThread().getName());
            subscriber.onNext("" + i);
        }
    }
})
.subscribeOn(Schedulers.io())
.observeOn(Schedulers.computation())
.subscribe(new Action1<String>() {
    @Override
    public void call(String s) {
        System.out.println("Subscribe Thread "+Thread.currentThread().getName());
    }
});
```
运行：
```
Observable Thread RxIoScheduler-2
Observable Thread RxIoScheduler-2
Observable Thread RxIoScheduler-2
Subscribe Thread RxComputationScheduler-1
Subscribe Thread RxComputationScheduler-1
Subscribe Thread RxComputationScheduler-1
```
这是最典型的使用，RxJava设计的目的就是处理异步处理，特别是耗时的网络操作，这里将OnSubscribe放到io线程池里去处理，在computation来处理结果。

##### 一个Producer多个Consumer
```
Observable<String> producer = Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        for (int i = 0; i < 3; i++) {
            System.out.println("Observable Thread "+Thread.currentThread().getName());
            subscriber.onNext("" + i);
        }
    }
});

producer
.subscribeOn(Schedulers.io())
.observeOn(Schedulers.computation())
.subscribe(new Action1<String>() {
    @Override
    public void call(String s) {
        System.out.println("Subscribe1 Thread "+Thread.currentThread().getName());
    }
});

producer
.subscribeOn(Schedulers.io())
.observeOn(Schedulers.computation())
.subscribe(new Action1<String>() {
    @Override
    public void call(String s) {
        System.out.println("Subscribe2 Thread "+Thread.currentThread().getName());
    }
});
```
运行：
```
Observable Thread RxIoScheduler-2
Observable Thread RxIoScheduler-2
Observable Thread RxIoScheduler-2
Subscribe1 Thread RxComputationScheduler-1
Subscribe1 Thread RxComputationScheduler-1
Subscribe1 Thread RxComputationScheduler-1
Observable Thread RxIoScheduler-3
Observable Thread RxIoScheduler-3
Observable Thread RxIoScheduler-3
Subscribe2 Thread RxComputationScheduler-2
Subscribe2 Thread RxComputationScheduler-2
Subscribe2 Thread RxComputationScheduler-2
```
使用create的producer会运行多次OnSubscribe


##### 测试onError,onCompleted
同步执行：
```
Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        for (int i = 0; i < 3; i++) {
            if (i == 2) {
                subscriber.onError(new RuntimeException("测试OnError"));
            }
            System.out.println("Emit onNext ");
            subscriber.onNext("" + i);
        }
        System.out.println("Emit onCompleted ");
        subscriber.onCompleted();
    }
})
.subscribe(new Action1<String>() {
    @Override
    public void call(String s) {
        System.out.println("Subscriber onNext "+s);
    }
}, new Action1<Throwable>() {
    @Override
    public void call(Throwable throwable) {
        System.out.println("Subscriber onError "+throwable.getMessage());
    }
}, new Action0() {
    @Override
    public void call() {
        System.out.println("Subscriber onComplete");
    }
});
```
结果：
```
Emit onNext 
Subscriber onNext 0
Emit onNext 
Subscriber onNext 1
Subscriber onError 测试OnError
Emit onNext 
Emit onCompleted 
```
可以看到onError之后，OnSubscribe仍然会Emit Data，但是Subscriber已经不会处理，其实已经unsubscribe了。

将subscribe和observe分开在不同的线程池里运行：
```
Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        for (int i = 0; i < 3; i++) {
            if (i == 2) {
                System.out.println(Thread.currentThread().getName()+" Emit onError ");
                subscriber.onError(new RuntimeException("测试OnError"));
            }
            System.out.println(Thread.currentThread().getName()+" Emit onNext ");
            subscriber.onNext("" + i);
        }
        System.out.println(Thread.currentThread().getName()+" Emit onCompleted ");
        subscriber.onCompleted();
    }
})
.subscribeOn(Schedulers.io())
.observeOn(Schedulers.computation())
.subscribe(new Action1<String>() {
    @Override
    public void call(String s) {
        System.out.println(Thread.currentThread().getName()+" Subscriber onNext "+s);
    }
}, new Action1<Throwable>() {
    @Override
    public void call(Throwable throwable) {
        System.out.println(Thread.currentThread().getName()+" Subscriber onError "+throwable.getMessage());
    }
}, new Action0() {
    @Override
    public void call() {
        System.out.println(Thread.currentThread().getName()+" Subscriber onComplete");
    }
});
```
结果：
```
RxIoScheduler-2 Emit onNext 
RxIoScheduler-2 Emit onNext 
RxIoScheduler-2 Emit onError 
RxComputationScheduler-1 Subscriber onNext 0
RxIoScheduler-2 Emit onNext 
RxIoScheduler-2 Emit onCompleted 
RxComputationScheduler-1 Subscriber onError 测试OnError
```


##### subscription.unsubscribe

```
Subscription subscription = Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        for (int i = 0; i < 3; i++) {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName()+" Emit onNext ");
            subscriber.onNext("" + i);
        }
        System.out.println(Thread.currentThread().getName()+" Emit onCompleted ");
        subscriber.onCompleted();
    }
})
.subscribeOn(Schedulers.io())
.observeOn(Schedulers.computation())
.subscribe(new Action1<String>() {
    @Override
    public void call(String s) {
        System.out.println(Thread.currentThread().getName()+" Subscriber onNext "+s);
    }
}, new Action1<Throwable>() {
    @Override
    public void call(Throwable throwable) {
        System.out.println(Thread.currentThread().getName()+" Subscriber onError "+throwable.getMessage());
    }
}, new Action0() {
    @Override
    public void call() {
        System.out.println(Thread.currentThread().getName()+" Subscriber onComplete");
    }
});

Thread.sleep(2200);
subscription.unsubscribe();
```
结果：
```
RxIoScheduler-2 Emit onNext 
RxComputationScheduler-1 Subscriber onNext 0
RxIoScheduler-2 Emit onNext 
RxComputationScheduler-1 Subscriber onNext 1
```
注意：调用unsubscribe之后，OnSubscribe也会停止EmitData。

##### 异步调用observeOn原理

```
.observeOn(Schedulers.computation())
 =>
public final Observable<T> observeOn(Scheduler scheduler) {
    return this.observeOn(scheduler, RxRingBuffer.SIZE);
}
=>
public final Observable<T> observeOn(Scheduler scheduler, int bufferSize) {
    return this.observeOn(scheduler, false, bufferSize);
}
=> 
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
    return this instanceof ScalarSynchronousObservable ? ((ScalarSynchronousObservable)this).scalarScheduleOn(scheduler) : this.lift(new OperatorObserveOn(scheduler, delayError, bufferSize));
}
=> 通过lift转换成一个OnSubscribeLift
public final <R> Observable<R> lift(Observable.Operator<? extends R, ? super T> operator) {
    return create((Observable.OnSubscribe)(new OnSubscribeLift(this.onSubscribe, operator)));
}

public final class OnSubscribeLift<T, R> implements OnSubscribe<R> {
    final OnSubscribe<T> parent;
    final Operator<? extends R, ? super T> operator;

    public OnSubscribeLift(OnSubscribe<T> parent, Operator<? extends R, ? super T> operator) {
        this.parent = parent;
        this.operator = operator;
    }

    public void call(Subscriber<? super R> o) {
        try {
            //使用一个中间的Subscriber对象来处理
            Subscriber st = (Subscriber)RxJavaHooks.onObservableLift(this.operator).call(o);

            try {
                st.onStart();
                //将之前的parent OnSubscribe调用到中间的Subscriber
                this.parent.call(st);
            } catch (Throwable var4) {
                Exceptions.throwIfFatal(var4);
                st.onError(var4);
            }
        } catch (Throwable var5) {
            Exceptions.throwIfFatal(var5);
            o.onError(var5);
        }

    }
}

最后是调用OperatorObserveOn，它是一个Operator<R, T>

public interface Operator<R, T> extends Func1<Subscriber<? super R>, Subscriber<? super T>> {
}   

实际上最终异步的实现在：ObserveOnSubscriber，将onNext消息丢到queue里去，让subscriber异步获取执行。

public void onNext(T t) {
    if (!this.isUnsubscribed() && !this.finished) {
        if (!this.queue.offer(this.on.next(t))) {
            this.onError(new MissingBackpressureException());
        } else {
            this.schedule();
        }
    }
}

```

#### 总结

    1.Observable在被订阅时才会执行OnSubscribe#call方法
    2.onCompleted和onError都会取消执行，Subscription#unsubscribe
    3.如果调用了onError，那么再次调用onNext、onCompleted，都不会执行，因为已经unsubscribe。
    4.通过Subscription#unsubscribe可以取消执行。
    5.onNext,onCompleted,onError实际上会被包装成一个Subscriber，最后实际上调用是：Observable对象.call(subscriber)，将subscriber对象传递到OnSubscribe里，直接Emit Data给subscriber。