# JProfile

> JProfiler是怎么将这些数据收集并展现的?

![](./jprofile/001.png)

**分析的数据主要来自于下面两部分：**
1)  一部分来自于jvm的分析接口**JVMTI**(JVM Tool Interface) , JDK必须>=1.6(<font color=#048E54>例如: 对象的生命周期，thread的生命周期等信息</font>)
2)  一部分来自于instruments classes(可理解为class的重写,增加JProfiler相关统计功能 <font color=#048E54>例如：方法执行时间，次数，方法栈等信息</font>)

```
1. 用户在JProfiler GUI中下达监控的指令(一般就是点击某个按钮)
2. JProfiler GUI JVM 通过socket(默认端口8849)，发送指令给被分析的jvm中的JProfile Agent。
3. JProfiler Agent(如果不清楚Agent请看文章第三部分"启动模式") 收到指令后，将该指令转换成相关需要监听的事件或者指令,来注册到JVMTI上或者直接让JVMTI去执行某功能(例如dump jvm内存)
4. JVMTI 根据注册的事件，来收集当前jvm的相关信息。 例如: 线程的生命周期; jvm的生命周期;classes的生命周期;对象实例的生命周期;堆内存的实时信息等等
5. JProfiler Agent将采集好的信息保存到**内存**中，按照一定规则统计好(如果发送所有数据JProfiler GUI，会对被分析的应用网络产生比较大的影响)
6. 返回给JProfiler GUI Socket.
7. JProfiler GUI Socket 将收到的信息返回 JProfiler GUI Render
8. JProfiler GUI Render 渲染成最终的展示效果
```

## 数据采集方式和启动模式

### JProfier采集方式分为两种：Sampling(样本采集)和Instrumentation

- Sampling: 类似于样本统计, 每隔一定时间(5ms)将每个线程栈中方法栈中的信息统计出来。优点是对应用影响小(即使你不配置任何Filter, Filter可参考文章第四部分)，缺点是一些数据/特性不能提供(例如:方法的调用次数)

- Instrumentation: 在class加载之前，JProfier把相关功能代码写入到需要分析的class中，对正在运行的jvm有一定影响。优点: 功能强大，但如果需要分析的class多，那么对应用影响较大，一般配合Filter一起使用。所以一般JRE class和framework的class是在Filter中通常会过滤掉。

注: JProfiler本身没有指出数据的采集类型，这里的采集类型是针对方法调用的采集类型 。因为JProfiler的绝大多数核心功能都依赖方法调用采集的数据, 所以可以直接认为是JProfiler的数据采集类型。

### 启动模式:

- Attach mode
可直接将本机正在运行的jvm加载JProfiler Agent. 优点是很方便，缺点是一些特性不能支持。如果选择Instrumentation数据采集方式，那么需要花一些额外时间来重写需要分析的class。

- Profile at startup
在被分析的jvm启动时，将指定的JProfiler Agent手动加载到该jvm。JProfiler GUI 将收集信息类型和策略等配置信息通过socket发送给JProfiler Agent，收到这些信息后该jvm才会启动。
在被分析的jvm 的启动参数增加下面内容：

语法: -agentpath:[path to jprofilerti library]

【注】: 语法不清楚没关系，JProfiler提供了帮助向导.
![](./jprofile/002.png)

- Prepare for profiling:
和Profile at startup的主要区别：被分析的jvm不需要收到JProfiler GUI 的相关配置信息就可以启动。

- Offline profiling
一般用于适用于不能直接调试线上的场景。Offline profiling需要将信息采集内容和策略(一些Trigger, Trigger请参考文章第五部分)打包成一个配置文件(config.xml)，在线上启动该jvm 加载 JProfiler Agent时，加载该xml。那么JProfiler Agent会根据Trigger的类型会生成不同的信息。例如: heap dump; thread dump; method call record等
语法:
-agentpath:/home/2080/jprofiler8/bin/Linux-x64/libjprofilerti.so=offline,id=151,config=/home/2080/config.xml

【注】: config.xml中的每一个被分析的jvm的采集信息都有一个id来标识。

下面是使用了离线模式，并使用了每隔一秒dump heap 的Trigger：
![](./jprofile/003.png)

## JProfiler核心概念

