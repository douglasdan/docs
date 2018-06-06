
#### Hystrix Properties

Hystrix配置提供了灵活的配置：
```
private static HystrixProperty<Boolean> getProperty(String propertyPrefix,
                                                        HystrixCommandKey key,
                                                        String instanceProperty,
                                                        Boolean builderOverrideValue,
                                                        Boolean defaultValue) {
        return forBoolean()
                .add(propertyPrefix + ".command." + key.name() + "." + instanceProperty, builderOverrideValue)
                .add(propertyPrefix + ".command.default." + instanceProperty, defaultValue)
                .build();
    }
```

	prefix：	指定了前缀，默认是hystrix
	key：		hystrix.command.cmdkey，针对cmdkey配置
	default：	hystrix.command.default这个
	默认值：	代码中默认值

想要临时修改默认值，可以通过ConfigurationManager.getConfigInstance().setProperty来设置全局默认值，之后通过clearProperty来reset。

```
ConfigurationManager.getConfigInstance().setProperty("hystrix.command.default.circuitBreaker.forceClosed", true);
ConfigurationManager.getConfigInstance().clearProperty("hystrix.command.default.circuitBreaker.forceClosed");
```

#### Hystrix CircuitBreaker
方法：
```
boolean allowRequest();
boolean isOpen();
void markSuccess();
```

CircuitBreaker内部实现HystrixCircuitBreakerImpl其实是通过properties和metrics来判断是否需要断开的。

```
default_circuitBreakerRequestVolumeThreshold = 20;		//在服务流量不大的情况下，可以调小此值
default_circuitBreakerSleepWindowInMilliseconds = 5000;
default_circuitBreakerErrorThresholdPercentage = 50;
```

```
//判断或更新断路器开闭状态
public boolean isOpen() {
	if (this.circuitOpen.get()) {
		return true;
	} else {
		HealthCounts health = this.metrics.getHealthCounts();
		if (health.getTotalRequests() < (long)(Integer)this.properties.circuitBreakerRequestVolumeThreshold().get()) {
			return false;
		} else if (health.getErrorPercentage() < (Integer)this.properties.circuitBreakerErrorThresholdPercentage().get()) {
			return false;
		} else if (this.circuitOpen.compareAndSet(false, true)) {
			//到这里，必须符合条件：
			//1.请求次数大于等于circuitBreakerRequestVolumeThreshold
			//2.错误次数大于等于circuitBreakerErrorThresholdPercentage
			//记录下断开时间，在测试时会用到
			this.circuitOpenedOrLastTestedTime.set(System.currentTimeMillis());
			return true;
		} else {
			return true;
		}
	}
}

public boolean allowRequest() {
	if ((Boolean)this.properties.circuitBreakerForceOpen().get()) { 	//强制断开
		return false;
	} else if ((Boolean)this.properties.circuitBreakerForceClosed().get()) {	//强制闭合
		this.isOpen();
		return true;
	} else {
	
		//判断当前是否是断开，或者可以进行短路测试：allowSingleTest
		//可能存在allowRequest=true，isOpen=true的情况，这种情况为了在断路器打开之后，进行服务测试
		return !this.isOpen() || this.allowSingleTest();	
	}
}

public boolean allowSingleTest() {
	//获取上次断开时间
	long timeCircuitOpenedOrWasLastTested = circuitOpenedOrLastTestedTime.get();
	
	//当前是断开状态，且当前时间距离上次断开时间大于circuitBreakerSleepWindowInMilliseconds
	if (circuitOpen.get() 
		&& System.currentTimeMillis() > timeCircuitOpenedOrWasLastTested + properties.circuitBreakerSleepWindowInMilliseconds().get()) {
		
		//重新设置timeCircuitOpenedOrWasLastTested，如果这次测试失败，那么下次还是要等circuitBreakerSleepWindowInMilliseconds时间才会再测试一次
		if (circuitOpenedOrLastTestedTime.compareAndSet(timeCircuitOpenedOrWasLastTested, System.currentTimeMillis())) {
			//如果返回为true，则表示可以允许测试
			//如果返回为false，则表示其他线程抢先一步进行了测试
			return true;
		}
	}
	return false;
}

public void markSuccess() {
	if (circuitOpen.get()) {
		if (circuitOpen.compareAndSet(true, false)) {
			//强制连通断路器
			//metrics.resetStream，只会影响healthCountsStream
			metrics.resetStream();
		}
	}
}

#HystrixCommandMetrics
synchronized void resetStream() {
	healthCountsStream.unsubscribe();
	HealthCountsStream.removeByKey(key);
	healthCountsStream = HealthCountsStream.getInstance(key, properties);
}
```

如果执行超时：HystrixTimeoutException，会立即出发断路器打开。

断路器打开时，请求会立即失败，不会进入metrics。

关闭断路器，指定使用：NoOpCircuitBreaker

```
public boolean allowRequest() {
	return true;
}

public boolean isOpen() {
	return false;
}

public void markSuccess() {

}
```

#### hystrix command命令执行

##### 线程隔离
在outer command里面，创建并调用inner command，提交执行的threadpool默认是根据CommandGroupKey来划分。

outer command和inner command执行的线程和在哪调用没有关系，都会提交到ThreadPool里去执行。

如果将使用andThreadPoolPropertiesDefaults(HystrixThreadPoolProperties.defaultSetter().withCoreSize(1))设置线程池coresize=1，

如果Outer Command和Inner Command的GroupKey一样，那么inner commmand执行时将会抛出异常：RejectedExecutionException。

##### 信号量隔离
同上面例子，














