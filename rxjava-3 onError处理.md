#### onErrorResumeNext

onErrorResumeNext：当发生Error时，返回Observable，即，发生Error时：

    1.原Observable的subscription.unsubscribe
    2.返回一个对应Error的Observable，这个Observable向subscriber继续发射数据

```
Observable.create(new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> subscriber) {
        for (int i = 0; i < 6; i++) {
            if (i == 2) {
                subscriber.onError(new RuntimeException("测试OnError"));
            }
            System.out.println(Thread.currentThread().getName()+" Emit onNext "+i);
            subscriber.onNext(i);
        }
        System.out.println(Thread.currentThread().getName()+" Emit onCompleted ");
        subscriber.onCompleted();
    }
})
.onErrorResumeNext(new Func1<Throwable, Observable<? extends Integer>>() {
    @Override
    public Observable<? extends Integer> call(Throwable throwable) {
        System.out.println("onErrorResumeNext "+throwable.getMessage()+" 返回 -1");
        return Observable.just(-1);
    }
})
.subscribe(new Action1<Integer>() {
    @Override
    public void call(Integer s) {
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
main Emit onNext 0
main Subscriber onNext 0
main Emit onNext 1
main Subscriber onNext 1
main Subscriber onNext -1
main Subscriber onComplete
main Emit onNext 2
main Emit onNext 3
main Emit onNext 4
main Emit onNext 5
main Emit onCompleted 
```

Observable.just会默认发送onCompleted，使用OnSubscribe，可以自定义发送内容
```
.onErrorResumeNext(new Func1<Throwable, Observable<? extends Integer>>() {
    @Override
    public Observable<? extends Integer> call(Throwable throwable) {

        return Observable.create(new Observable.OnSubscribe<Integer>() {
            @Override
            public void call(Subscriber<? super Integer> subscriber) {
                for (int i=-1; i>= -3; i--) {
                    subscriber.onNext(i);
                }
            }
        });
    }
})
```
结果：
```
main Emit onNext 0
main Subscriber onNext 0
main Emit onNext 1
main Subscriber onNext 1
main Subscriber onNext -1
main Subscriber onNext -2
main Subscriber onNext -3
main Emit onNext 2
main Emit onNext 3
main Emit onNext 4
main Emit onNext 5
main Emit onCompleted 
```

#### onExceptionResumeNext

和onErrorResumeNext类似，当源Observable发生异常时调用，无法获取Throwable
```
Observable.create(new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> subscriber) {
        for (int i = 0; i < 6; i++) {
            if (i == 2) {
                throw new RuntimeException("测试OnError");
            }
            System.out.println(Thread.currentThread().getName()+" Emit onNext "+i);
            subscriber.onNext(i);
        }
        System.out.println(Thread.currentThread().getName()+" Emit onCompleted ");
        subscriber.onCompleted();
    }
})
.onExceptionResumeNext(Observable.just(-2))
```
结果：
```
main Emit onNext 0
main Subscriber onNext 0
main Emit onNext 1
main Subscriber onNext 1
main Subscriber onNext -2
main Subscriber onComplete
```

自定义Observable，并发送onCompleteds
```
Observable.create(new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> subscriber) {
        for (int i = 0; i < 6; i++) {
            if (i == 2) {
                throw new RuntimeException("测试OnError");
            }
            System.out.println(Thread.currentThread().getName()+" Emit onNext "+i);
            subscriber.onNext(i);
        }
        System.out.println(Thread.currentThread().getName()+" Emit onCompleted ");
        subscriber.onCompleted();
    }
})
.onExceptionResumeNext(Observable.create(new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> subscriber) {
        for (int i=-1; i>= -3; i--) {
            subscriber.onNext(i);
        }
    }
}))
```
结果：
```
main Emit onNext 0
main Subscriber onNext 0
main Emit onNext 1
main Subscriber onNext 1
main Subscriber onNext -1
main Subscriber onNext -2
main Subscriber onNext -3
```

#### onErrorReturn

和onExceptionResumeNext类似，只是Observable变成Integer Data，直接返回onNext数据

触发onError
```
Observable.create(new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> subscriber) {
        for (int i = 0; i < 6; i++) {
            if (i == 2) {
                subscriber.onError(new RuntimeException("测试OnError"));
            }
            System.out.println(Thread.currentThread().getName()+" Emit onNext "+i);
            subscriber.onNext(i);
        }
        System.out.println(Thread.currentThread().getName()+" Emit onCompleted ");
        subscriber.onCompleted();
    }
})
.onErrorReturn(new Func1<Throwable, Integer>() {
    @Override
    public Integer call(Throwable throwable) {
        return -3;
    }
})
```
结果：
```
main Emit onNext 0
main Subscriber onNext 0
main Emit onNext 1
main Subscriber onNext 1
main Subscriber onNext -3
main Subscriber onComplete
main Emit onNext 2
main Emit onNext 3
main Emit onNext 4
main Emit onNext 5
main Emit onCompleted 
```

抛出异常：
```
Observable.create(new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> subscriber) {
        for (int i = 0; i < 6; i++) {
            if (i == 2) {
                throw new RuntimeException("测试OnError");
            }
            System.out.println(Thread.currentThread().getName()+" Emit onNext "+i);
            subscriber.onNext(i);
        }
        System.out.println(Thread.currentThread().getName()+" Emit onCompleted ");
        subscriber.onCompleted();
    }
})
.onErrorReturn(new Func1<Throwable, Integer>() {
    @Override
    public Integer call(Throwable throwable) {
        return -3;
    }
})
```
结果：
```
main Emit onNext 0
main Subscriber onNext 0
main Emit onNext 1
main Subscriber onNext 1
main Subscriber onNext -3
main Subscriber onComplete
```

可以看到onError或者抛出异常都会触发onError

#### onExceptionResumeNext和onErrorReturn一起使用



```
Observable.create(new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> subscriber) {
        for (int i = 0; i < 6; i++) {
            if (i == 2) {
                subscriber.onError(new RuntimeException("测试OnError"));
            }
            System.out.println(Thread.currentThread().getName()+" Emit onNext "+i);
            subscriber.onNext(i);
        }
        System.out.println(Thread.currentThread().getName()+" Emit onCompleted ");
        subscriber.onCompleted();
    }
})
.onExceptionResumeNext(Observable.just(-2))
.onErrorReturn(new Func1<Throwable, Integer>() {
    @Override
    public Integer call(Throwable throwable) {
        return -3;
    }
})
```
结果：
```
main Emit onNext 0
main Subscriber onNext 0
main Emit onNext 1
main Subscriber onNext 1
main Subscriber onNext -2
main Subscriber onComplete
main Emit onNext 2
main Emit onNext 3
main Emit onNext 4
main Emit onNext 5
main Emit onCompleted 
```
可以看到只会按顺序触发第一个。
