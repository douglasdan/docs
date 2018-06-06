#### map作用

按照RxJava wiki上的介绍：
    
    Returns an Observable that applies a specified function to each item emitted by the source Observable and emits the results of these function applications.

返回一个Observable对象，将原Observable发射出的对象做一次转换再发射给subscriber。

![avatar](https://raw.github.com/wiki/ReactiveX/RxJava/images/rx-operators/map.png)

与函数式编程里的概念一致，map是一个操作符。

#### 
例子：
```
Observable.from(new String[]{"This", "is", "RxJava"})
    .map(new Func1<String, String>() {
        @Override
        public String call(String s) {
            return s.toUpperCase();
        }
    })
    .toList()
    .map(new Func1<List<String>, List<String>>() {
        @Override
        public List<String> call(List<String> strings) {
            Collections.reverse(strings);
            return strings;
        }
    })
    .observeOn(Schedulers.computation())
    .subscribeOn(Schedulers.io())
    .subscribe(new Action1<List<String>>() {
        @Override
        public void call(List<String> s) {
            System.out.println(s.toString());
        }
    });
```
Observable.from(String[])，将数据包装转化成一个Observable<String>

第一次map，使用：Func1<T, R>，将原类型T转成R
```
new Func1<String, String>() {       
    @Override
    public String call(String s) {
        return s.toUpperCase();
    }
}
```

toList，将Observable<String>转成Observable<List<T>>，实际是通过个内部的Subscriber，做了一层包装，内部生成一个producer来向原来的subscriber发射数据
```
public Subscriber<? super T> call(final Subscriber<? super List<T>> o) {
    final SingleDelayedProducer<List<T>> producer = new SingleDelayedProducer(o);
    Subscriber<T> result = new Subscriber<T>() {
        boolean completed;
        List<T> list = new LinkedList();

        public void onStart() {
            this.request(9223372036854775807L);
        }

        public void onCompleted() {
            //只有收到onCompleted之后，才会向实际的subscriber发射数据，而且只会一次
            if (!this.completed) {
                this.completed = true;

                ArrayList result;
                try {
                    result = new ArrayList(this.list);
                } catch (Throwable var3) {
                    Exceptions.throwOrReport(var3, this);
                    return;
                }

                this.list = null;
                //producer setvalue即会发射
                producer.setValue(result);
            }

        }

        public void onError(Throwable e) {
            o.onError(e);
        }

        public void onNext(T value) {
            if (!this.completed) {
                this.list.add(value);
            }

        }
    };
    o.add(result);
    o.setProducer(producer);
    return result;
}

public final class SingleDelayedProducer<T> extends AtomicInteger implements Producer {
    ...
    public void setValue(T value) {
        while(true) {
            int s = this.get();
            if (s == 0) {
                this.value = value;
                if (!this.compareAndSet(0, 1)) {
                    continue;
                }
            } else if (s == 2 && this.compareAndSet(2, 3)) {
                emit(this.child, value);
            }

            return;
        }
    }
}
```

实际例子，结合耗时IO操作
```
Observable.just(
        "http://www.baidu.com/",
        "http://www.google.com/",
        "https://www.bing.com/")
        .map(new Func1<String, String>() {
            @Override
            public String call(String s) {
                try {
                    return s + " : " + getIPByUrl(s);
                } catch (MalformedURLException e) {
                    e.printStackTrace();
                } catch (UnknownHostException e) {
                    e.printStackTrace();
                }
                return null;
            }
        })
        .subscribeOn(Schedulers.io())
        .observeOn(Schedulers.computation())
        .subscribe(new Action1<String>() {
            @Override
            public void call(String s) {
                System.out.println(s);
            }
        });

        Thread.sleep(1000);
```

#### flatmap

flat是平的一次，flat map的意思是将复杂对象转化成需要的对象

在RxJava中，将多个Observable emit 的data一起发射给subscriber，可以理解为合并Observable的意思

![avatar](https://raw.github.com/wiki/ReactiveX/RxJava/images/rx-operators/flatMap.png)

```
private static String getIPByUrl(String str) throws MalformedURLException, UnknownHostException {
    URL urls = new URL(str);
    String host = urls.getHost();
    String address = InetAddress.getByName(host).toString();
    int b = address.indexOf("/");
    return address.substring(b + 1);

}

private static Observable<String> createIpObservable(final String url) {
    return Observable.create(new Observable.OnSubscribe<String>() {
        @Override
        public void call(Subscriber<? super String> subscriber) {
            try {
                String ip = getIPByUrl(url);
                subscriber.onNext(ip);

                System.out.println(Thread.currentThread().getName()+" Object ="+ this+"  "+url+" : " +ip);
            } catch (MalformedURLException e) {
                e.printStackTrace();
                subscriber.onNext(null);
            } catch (UnknownHostException e) {
                e.printStackTrace();
                subscriber.onNext(null);
            }
            subscriber.onCompleted();
        }
    });
}
```

```
Observable.just(
    "http://www.baidu.com/",
    "http://www.google.com/",
    "https://www.bing.com/")
    .flatMap(new Func1<String, Observable<String>>() {
        @Override
        public Observable<String> call(String s) {
            return createIpObservable(s);
        }
    })
    .subscribeOn(Schedulers.io())
    .observeOn(Schedulers.computation())
    .subscribe(new Action1<String>() {
        @Override
        public void call(String s) {
            System.out.println(Thread.currentThread().getName()+" consume "+s);
        }
    }, new Action1<Throwable>() {
        @Override
        public void call(Throwable throwable) {
            System.out.println(Thread.currentThread().getName()+" throwable "+throwable.getMessage());
        }
    });
```

实际使用结合对象：

```
//p1,p2,p3为person对象实例，一个Person有多个Cat
//本例子将多个person的多个cat合并发射给subscriber
Observable.just(p1,p2,p3).flatMap(new Func1<Person, Observable<Cat>>() {
  @Override public Observable<Cat> call(Person person) {
    return Observable.from(person.getCats());
  }
}).subscribe(new Action1<Cat>() {
  @Override public void call(Cat cat) {
    System.out.println("输出结果：" + cat.getName());
  }
});
```

#### flatMapIterable

和flatmap类似，只是将Observable，换成了Iterable对象
```
Observable.just(1,2,3).flatMapIterable(new Func1<Integer, List<Integer>>() {
  @Override public List<Integer> call(Integer integer) {
    List<Integer> list = new ArrayList<Integer>();
    list.add(integer);
    return list;
  }
}).subscribe(new Action1<Integer>() {
  @Override public void call(Integer integer) {
    System.out.println("输出结果->"+integer);
  }
});
}
```   
方法：
```
public final <R> Observable<R> flatMapIterable(Func1<? super T, ? extends Iterable<? extends R>> collectionSelector) 
```

测试flatmap发射顺序：
```
Observable.just(1,2,3,4,5,6).flatMap(new Func1<Integer, Observable<Integer>>() {
    @Override
    public Observable<Integer> call(Integer integer) {
        int time=1;
        if(integer%2==0)
            time=2;
        System.out.println("Producer "+Thread.currentThread().getName()+" produce "+integer);
        return Observable.just(integer).delay(time,TimeUnit.SECONDS);
    }
})
.observeOn(Schedulers.computation())
.subscribe(new Action1<Integer>() {
    @Override
    public void call(Integer integer) {
        System.out.println("Consumer "+Thread.currentThread().getName()+" consume "+String.valueOf(integer));
    }
});
```
结果：
```
Producer main produce 1
Producer main produce 2
Producer main produce 3
Producer main produce 4
Producer main produce 5
Producer main produce 6
Consumer RxComputationScheduler-1 consume 1
Consumer RxComputationScheduler-1 consume 3
Consumer RxComputationScheduler-1 consume 5
Consumer RxComputationScheduler-1 consume 2
Consumer RxComputationScheduler-1 consume 6
Consumer RxComputationScheduler-1 consume 4
```

delay方法：
```
public final Observable<T> delay(long delay, TimeUnit unit) {
    return this.delay(delay, unit, Schedulers.computation());
}
```
OperatorDelay操作符：
```
public final class OperatorDelay<T> implements Operator<T, T> {
    final long delay;
    final TimeUnit unit;
    final Scheduler scheduler;

    public OperatorDelay(long delay, TimeUnit unit, Scheduler scheduler) {
        this.delay = delay;
        this.unit = unit;
        this.scheduler = scheduler;
    }

    public Subscriber<? super T> call(final Subscriber<? super T> child) {
        final Worker worker = this.scheduler.createWorker();
        child.add(worker);
        return new Subscriber<T>(child) {
            boolean done;

            public void onCompleted() {
                //schedule delay指定time去发射onCompleted
                worker.schedule(new Action0() {
                    public void call() {
                        if (!done) {
                            done = true;
                            child.onCompleted();
                        }

                    }
                }, OperatorDelay.this.delay, OperatorDelay.this.unit);
            }

            public void onError(final Throwable e) {
                //onError是会直接发射，不会delay，即错误是立即抛出的，onNext，onCompleted都认为正常
                worker.schedule(new Action0() {
                    public void call() {
                        if (!done) {
                            done = true;
                            child.onError(e);
                            worker.unsubscribe();
                        }

                    }
                });
            }

            public void onNext(final T t) {
                //schedule delay指定time去发射onNext
                worker.schedule(new Action0() {
                    public void call() {
                        if (!done) {
                            child.onNext(t);
                        }

                    }
                }, OperatorDelay.this.delay, OperatorDelay.this.unit);
            }
        };
    }
}
```

#### concatMap

按次序连接而不是合并那些生成的Observables

```
Observable.just(1,2,3,4,5,6).concatMap(new Func1<Integer, Observable<Integer>>() {
    @Override
    public Observable<Integer> call(Integer integer) {
        int time=1;
        if(integer%2==0)
            time=2;
        return Observable.just(integer).delay(time,TimeUnit.SECONDS);
    }
}).observeOn(Schedulers.computation()).subscribe(new Action1<Integer>() {
    @Override
    public void call(Integer integer) {
        System.out.println(Thread.currentThread().getName()+" consume "+integer);
    }
});
```

运行结果：
```
RxComputationScheduler-1 consume 1
RxComputationScheduler-1 consume 2
RxComputationScheduler-1 consume 3
RxComputationScheduler-1 consume 4
RxComputationScheduler-1 consume 5
RxComputationScheduler-1 consume 6
```

#### switchMap

```
Observable.just(1,2,3,4,5,6).switchMap(new Func1<Integer, Observable<Integer>>() {
    @Override
    public Observable<Integer> call(Integer integer) {
        int time=1;
        if(integer%2==0)
            time=2;
        return Observable.just(integer).delay(time,TimeUnit.SECONDS);
    }
}).observeOn(Schedulers.computation()).subscribe(new Action1<Integer>() {
    @Override
    public void call(Integer integer) {
        System.out.println(Thread.currentThread().getName()+" consume "+integer);
    }
});
```
结果：
```
RxComputationScheduler-1 consume 6
```
