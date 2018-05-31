
#### Spring Cloud 微服务jvm配置

    -server
    -Dfile.encoding=UTF-8
    -Xss1m                                      //spring和rxjava调用堆栈层数很多，将xss设置大一些
    -Xms
    -Xmx
    
    #新生代空间
    -XX:NewRatio=1                              //
    -XX:NewSize                                 //新生代空间
    -XX:SurvivorRatio                           //伊甸区和幸存区占比
    
    #持久代
    -XX:PermSize
    -XX:MaxPermSize
    
##### GC回收算法

    #按序回收
    -XX:+UseSerialGC                            //Young区和old区都使用serial 垃圾回收算法
    
    #Parallel
    -XX:+UseParallelGC                          //Young区：使用Parallel scavenge 回收算法
    -XX:ParallelGCThreads                       //与UseParallelGC匹配的参数，配置并行收集线程数，一般与core数一样
    -XX:+UseParallelOldGC                       //Old区：可以使用单线程的或者Parallel 垃圾回收算法
    
    #CMS
    #针对年老代的回收，减少Full GC时间和频率
    #用两次短暂停来替代串行标记整理算法的长暂停
    #初始标记(CMS-initial-mark) -> 并发标记(CMS-concurrent-mark) -> 
    #重新标记(CMS-remark) -> 并发清除(CMS-concurrent-sweep) ->并发重设状态等待下次CMS的触发(CMS-concurrent-reset)
    
    -XX:+UseConcMarkSweepGC                     //Old区只能用Concurrent Mark Sweep
    -XX:+UseParNewGC                            //指定对年轻代使用parallel回收，新版本里，开启了UseConcMarkSweepGC，会自动开启
    -XX:+CMSParallelRemarkEnabled               //减少第二次暂停的时间
    -XX:+CMSScavengeBeforeRemark                //如果第二次remark还是过长，这个参数可以再remark之后开启一次minor gc
    -XX:+UseCMSInitiatingOccupancyOnly          //
    -XX:CMSInitiatingOccupancyFraction=75       //在年老带沾满75%时开始进行CMS收集
    -XX:+UseCMSCompactAtFullCollection          //CMS不会整理堆碎片，这个参数可以开启碎片整理，可能会影响性能，需要实践
    -XX:+UseCMSInitiatingOccupancyOnly          //
    -XX:+CMSClassUnloadingEnabled
    -XX:+CMSIncrementalMode
    -XX:+ExplicitGCInvokesConcurrent
    -XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses
    -XX:+DisableExplicitGC                      //禁掉System.gc
    -XX:+ExplicitGCInvokesConcurrent            //设置了改参数，就不要设置DisableExplicitGC
    
    #开启CMS回收Perm选项
    -XX:+CMSPermGenSweepingEnabled
    -XX:+CMSClassUnloadingEnabled
    
    #G1
    -XX:+UseG1GC                                //没有young/old区
    -XX:+UnlockExperimentalVMOptions
    
    -XX:MaxTenuringThreshold=6                  //存活多少次进入幸存区
    
    
    #gc日志
    -verbose:gc 
    -XX:+PrintGCTimeStamps 
    -XX:+PrintGCDetails 
    -Xloggc:/home/test/logs/gc.log
    
    
    -XX:+AlwaysPreTouch                         //
    -XX:AutoBoxCacheMax=20000                   //自动装箱缓存数量
    -XX:+HeapDumpOnOutOfMemoryError             //
     
    
##### 经典设置

    -server
    -Dfile.encoding=UTF-8
    -Xss1m
    -XX:-UseConcMarkSweepGC
    -XX:+UseCMSCompactAtFullCollection
    -XX:CMSInitiatingOccupancyFraction=80
    -XX:+CMSParallelRemarkEnabled
    -XX:SoftRefLRUPolicyMSPerMB=0
    -verbose:gc 
    -XX:+PrintGCTimeStamps
    -XX:+PrintGCDetails 
    -Xloggc:/mnt/logs/gc.log
    
##### 
    
