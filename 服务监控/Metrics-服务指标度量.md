### 简介
[Metrics](https://github.com/dropwizard/metrics)作为一款监控指标的度量类库，提供了很多模块可以为第三方库或者应用提供辅助统计信息。Metrics内部提供了Gauge、Counter、Meter、Histogram、Timer等度量工具类以及Health Check功能。

### Maven配置
metrics-core为metrics核心库，定义了各种指标项，需要在pom.xml引用。
```
<dependencies>
    <dependency>
        <groupId>io.dropwizard.metrics</groupId>
        <artifactId>metrics-core</artifactId>
        <version>3.1.0</version>
    </dependency>
</dependencies>
```
### MetricRegistry
MetricRegistry类是核心容器，内部使用ConcurrentHashMap维护所有监控指标项。
指标注册核心代码：
```java
    public <T extends Metric> T register(String name, T metric) throws IllegalArgumentException {
        if (metric instanceof MetricSet) {
            registerAll(name, (MetricSet) metric);
        } else {
            final Metric existing = metrics.putIfAbsent(name, metric);
            if (existing == null) {
                onMetricAdded(name, metric);
            } else {
                throw new IllegalArgumentException("A metric named " + name + " already exists");
            }
        }
        return metric;
    }
```
每个指标项都需要有个独一无二的名字，MetricRegistry类提供了名字生成的方式。除了可以根据类名来生成名字外，也支持自定义名字。其本质是字符串的拼接。
```java
    public static String name(String name, String... names) {
        final StringBuilder builder = new StringBuilder();
        append(builder, name);
        if (names != null) {
            for (String s : names) {
                append(builder, s);
            }
        }
        return builder.toString();
    }

    public static String name(Class<?> klass, String... names) {
        return name(klass.getName(), names);
    }

    private static void append(StringBuilder builder, String part) {
        if (part != null && !part.isEmpty()) {
            if (builder.length() > 0) {
                builder.append('.');
            }
            builder.append(part);
        }
    }
```
### Metrics数据展示
Metrics提供了Reporter接口，用于展示内部的数据指标信息。metrics-core中主要实现了ConsoleReporter、CsvReporter 、Slf4jReporter、JmxReporter。在本文例子中使用ConsoleReporter展示内部指标。
> 对于使用**Falcon**监控系统的公司，可以参照ConsoleReporter实现自定义的Reporter，这样Metrics就可以无缝集成到公司的监控系统上。


### Metrics度量指标
#### Gauge
Gauge主要记录指标的瞬时值，如服务当前**Jvm**使用情况等；
```java
public class JvmGaugeTest {

    public static void main(String[] args) throws Exception {
        MetricRegistry registry = new MetricRegistry();

        ConsoleReporter reporter = ConsoleReporter.forRegistry(registry).build();
        reporter.start(1, TimeUnit.SECONDS);

        MemoryMXBean memoryMXBean = ManagementFactory.getMemoryMXBean();
        registry.register("jvm.total.used",
                new Gauge<Long>() {

                    @Override
                    public Long getValue() {
                        return memoryMXBean.getHeapMemoryUsage().getUsed()
                                + memoryMXBean.getNonHeapMemoryUsage().getUsed();
                    }
                });

        while (true) {
            Thread.sleep(1000);
        }
    }
}
```
代码运行结果如下：
```
-- Gauges ----------------------------------------------------------------------
jvm.total.used
             value = 16314496
```
#### Counter
Counter是计数器，可以对Counter进行增加和减少操作，维护累计的指标。
```java
public class CounterTest {

    private static Queue<String> queue = new LinkedBlockingQueue<String>();
    private static Counter pendingJobs;
    private static Random random = new Random();

    public static void addJob(String job) {
        pendingJobs.inc();
        queue.offer(job);
    }

    public static String takeJob() {
        pendingJobs.dec();
        return queue.poll();
    }

    public static void main(String[] args) throws InterruptedException {
        MetricRegistry registry = new MetricRegistry();

        ConsoleReporter reporter = ConsoleReporter.forRegistry(registry).build();
        reporter.start(1, TimeUnit.SECONDS);

        pendingJobs = registry.counter("pending.jobs.size");
        for (int num = 1; ; num++) {
            Thread.sleep(100);
            if (random.nextDouble() > 0.8) {
                takeJob();
            } else {
                addJob("Job-" + num);
            }
        }
    }
}
```
代码运行结果如下：
```
-- Counters --------------------------------------------------------------------
pending.jobs.size
             count = 19
```
#### Meter
Meter度量事件发生的频率，统计最近1分钟、5分钟、15分钟的速率。
```java
public class MeterTest {

    private static Random random = new Random();

    public static void request(Meter meter, int times) {
        for (int i = 0; i < times; i++) {
            meter.mark();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        MetricRegistry registry = new MetricRegistry();

        ConsoleReporter reporter = ConsoleReporter.forRegistry(registry).build();
        reporter.start(1, TimeUnit.SECONDS);

        Meter meterTps = registry.meter("request.tps");
        while (true) {
            request(meterTps, random.nextInt(10));
            Thread.sleep(1000);
        }
    }

}
```
代码运行结果如下：
```
-- Meters ----------------------------------------------------------------------
request.tps
             count = 115
         mean rate = 5.00 events/second
     1-minute rate = 7.04 events/second
     5-minute rate = 7.63 events/second
    15-minute rate = 7.74 events/second
```
Meter参考UNIX系统关于平均负荷load average来设计的，其中使用到了[EMA](https://en.wikipedia.org/wiki/Moving_average#Exponential_moving_average) 指数移动平均算法。越近期的数据加权影响力越重。

![](https://ws2.sinaimg.cn/large/006tKfTcly1g0g66xi96uj30o009zn18.jpg)
#### Histogram
Histogram统计数据分布情况，统计最小值、最大值、平均值、中位数、75分位、90分位、95分位、99分位、99.9分位等数据。
```java
public class HistogramTest {

    private static Random random = new Random();

    public static void main(String[] args) throws Exception {
        MetricRegistry registry = new MetricRegistry();

        ConsoleReporter reporter = ConsoleReporter.forRegistry(registry).build();
        reporter.start(1, TimeUnit.SECONDS);

        Histogram histogram = new Histogram(new UniformReservoir());
        registry.register("request.histogram", histogram);

        while (true) {
            Thread.sleep(1000);
            histogram.update(random.nextInt(100));
        }
    }

}
```
代码运行结果如下：
```
-- Histograms ------------------------------------------------------------------
request.histogram
             count = 18
               min = 10
               max = 98
              mean = 53.28
            stddev = 29.48
            median = 44.50
              75% <= 83.50
              95% <= 98.00
              98% <= 98.00
              99% <= 98.00
            99.9% <= 98.00
```
##### 数据抽样
Histogram需要统计数据分布，其内部必须抽样维护数据信息。内置的数据抽样有以下几种实现：
- ExponentiallyDecayingReservoir：基于指数级别的抽样算法，根据更新时间与开始时间的差值转化为权重值，权重越大数据被保留的几率越大。
- UniformReservoir：随机抽样，随着更新次数的增加，数据被抽样的概率会减少。
- SlidingWindowReservoir：滑动窗口抽样，总是保留最新的统计数据。
- SlidingTimeWindowReservoir：滑动时间窗口抽样，总是保留最近时间段的统计数据。

#### 注意事项
若使用ExponentiallyDecayingReservoir和SlidingTimeWindowReservoir，需要注意容量，底层并不会限制容量大小。若服务流量大，可能会占用很多内存。
### Timer
Timer是Histogram和Meter的结合，Histogram统计耗时分布，Meter统计QPS；
``` java
public class TimerTest {

    public static Random random = new Random();

    private static void request() throws InterruptedException {
        Thread.sleep(random.nextInt(1000));
    }

    public static void main(String[] args) throws Exception {
        MetricRegistry registry = new MetricRegistry();

        ConsoleReporter reporter = ConsoleReporter.forRegistry(registry).build();
        reporter.start(1, TimeUnit.SECONDS);

        Timer timer = registry.timer("request.latency");
        Timer.Context ctx;
        while (true) {
            ctx = timer.time();
            request();
            ctx.stop();
        }
    }
}
```
代码运行结果如下：
```
-- Timers ----------------------------------------------------------------------
request.latency
             count = 22
         mean rate = 2.00 calls/second
     1-minute rate = 1.98 calls/second
     5-minute rate = 2.00 calls/second
    15-minute rate = 2.00 calls/second
               min = 148.93 milliseconds
               max = 865.65 milliseconds
              mean = 491.89 milliseconds
            stddev = 219.59 milliseconds
            median = 465.60 milliseconds
              75% <= 671.65 milliseconds
              95% <= 850.28 milliseconds
              98% <= 865.65 milliseconds
              99% <= 865.65 milliseconds
            99.9% <= 865.65 milliseconds
```
### 经验总结
当我们需要上报服务瞬时指标时会使用Guage，如Jvm的使用情况。当我们需要统计数据分布时会使用Histogram，如接口的响应耗时分布。当我们需要统计频率时会使用Meter，如某个接口的请求频率。当我们既需要统计频率也需要统计分布时会使用Timer对象，如某个接口的请求频率及耗时情况。

### 相关资料
[Metrics Core](https://metrics.dropwizard.io/3.1.0/manual/core/)