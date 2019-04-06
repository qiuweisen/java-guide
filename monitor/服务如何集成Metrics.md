### 1、概述
Metrics的基本介绍可以参考之前的文章：[Metrics-服务指标度量](https://juejin.im/post/5c3fdfa3e51d4567680e201c)。

本文简单介绍下如何将Metrics监控集成到我们的项目中。
> 本文所使用的metrics-core为3.1.0版本。
```
        <dependency>
            <groupId>io.dropwizard.metrics</groupId>
            <artifactId>metrics-core</artifactId>
            <version>3.1.0</version>
        </dependency>
```
### 2、场景
我们的主要的监控需求有以下方面：
- 机器指标

内存、线程、硬盘、服务GC情况等基本信息是我们关心的核心指标。我们可以考虑通过<font color=green>**Gauge**</font>指标项把这些机器指标做统一收集。

- 服务接口请求频率及耗时

请求频率及耗时是我们服务接口性能的核心指标，我们可以考虑通过<font color=green>**Timer**</font>指标项来采集相关信息。
- 服务内部基本数据

在某些场景下我们将内部Service统计到的瞬时指标上报，如Web Filter里面统计当前正在处理的请求数等。我们也可以使用<font color=green>**Gauge**</font>指标项来收集。

### 3、方案
针对于以上场景，我们虽然可以通过写代码的方式创建和注册相应的服务指标，可是在使用上却不太友好。如何更方便灵活地将Metrics指标统计集成到我们的项目中呢？
#### 3.1、MetricSet自动注册，收集机器指标
- (1) 预先定义好MetricSet；

指标集合MetricSet可参看metrics-jvm库的**MemoryUsageGaugeSet**来定义，MemoryUsageGaugeSet定义了内存使用情况的基本指标，如下所示。
```java
/**
 * A set of gauges for JVM memory usage, including stats on heap vs. non-heap memory, plus
 * GC-specific memory pools.
 */
public class MemoryUsageGaugeSet implements MetricSet {
    private static final Pattern WHITESPACE = Pattern.compile("[\\s]+");

    private final MemoryMXBean mxBean;
    private final List<MemoryPoolMXBean> memoryPools;

    public MemoryUsageGaugeSet() {
        this(ManagementFactory.getMemoryMXBean(),
             ManagementFactory.getMemoryPoolMXBeans());
    }

    public MemoryUsageGaugeSet(MemoryMXBean mxBean,
                               Collection<MemoryPoolMXBean> memoryPools) {
        this.mxBean = mxBean;
        this.memoryPools = new ArrayList<MemoryPoolMXBean>(memoryPools);
    }

    @Override
    public Map<String, Metric> getMetrics() {
        final Map<String, Metric> gauges = new HashMap<String, Metric>();

        gauges.put("total.init", new Gauge<Long>() {
            @Override
            public Long getValue() {
                return mxBean.getHeapMemoryUsage().getInit() +
                        mxBean.getNonHeapMemoryUsage().getInit();
            }
        });

        gauges.put("total.used", new Gauge<Long>() {
            @Override
            public Long getValue() {
                return mxBean.getHeapMemoryUsage().getUsed() +
                        mxBean.getNonHeapMemoryUsage().getUsed();
            }
        });

        gauges.put("total.max", new Gauge<Long>() {
            @Override
            public Long getValue() {
                return mxBean.getHeapMemoryUsage().getMax() +
                        mxBean.getNonHeapMemoryUsage().getMax();
            }
        });

        gauges.put("total.committed", new Gauge<Long>() {
            @Override
            public Long getValue() {
                return mxBean.getHeapMemoryUsage().getCommitted() +
                        mxBean.getNonHeapMemoryUsage().getCommitted();
            }
        });


        gauges.put("heap.init", new Gauge<Long>() {
            @Override
            public Long getValue() {
                return mxBean.getHeapMemoryUsage().getInit();
            }
        });

        gauges.put("heap.used", new Gauge<Long>() {
            @Override
            public Long getValue() {
                return mxBean.getHeapMemoryUsage().getUsed();
            }
        });

        gauges.put("heap.max", new Gauge<Long>() {
            @Override
            public Long getValue() {
                return mxBean.getHeapMemoryUsage().getMax();
            }
        });

        gauges.put("heap.committed", new Gauge<Long>() {
            @Override
            public Long getValue() {
                return mxBean.getHeapMemoryUsage().getCommitted();
            }
        });

        gauges.put("heap.usage", new RatioGauge() {
            @Override
            protected Ratio getRatio() {
                final MemoryUsage usage = mxBean.getHeapMemoryUsage();
                return Ratio.of(usage.getUsed(), usage.getMax());
            }
        });

        gauges.put("non-heap.init", new Gauge<Long>() {
            @Override
            public Long getValue() {
                return mxBean.getNonHeapMemoryUsage().getInit();
            }
        });

        gauges.put("non-heap.used", new Gauge<Long>() {
            @Override
            public Long getValue() {
                return mxBean.getNonHeapMemoryUsage().getUsed();
            }
        });

        gauges.put("non-heap.max", new Gauge<Long>() {
            @Override
            public Long getValue() {
                return mxBean.getNonHeapMemoryUsage().getMax();
            }
        });

        gauges.put("non-heap.committed", new Gauge<Long>() {
            @Override
            public Long getValue() {
                return mxBean.getNonHeapMemoryUsage().getCommitted();
            }
        });

        gauges.put("non-heap.usage", new RatioGauge() {
            @Override
            protected Ratio getRatio() {
                final MemoryUsage usage = mxBean.getNonHeapMemoryUsage();
                return Ratio.of(usage.getUsed(), usage.getMax());
            }
        });

        for (final MemoryPoolMXBean pool : memoryPools) {
            gauges.put(name("pools",
                            WHITESPACE.matcher(pool.getName()).replaceAll("-"),
                            "usage"),
                       new RatioGauge() {
                           @Override
                           protected Ratio getRatio() {
                               final long max = pool.getUsage().getMax() == -1 ?
                                       pool.getUsage().getCommitted() :
                                       pool.getUsage().getMax();
                               return Ratio.of(pool.getUsage().getUsed(), max);
                           }
                       });
        }

        return Collections.unmodifiableMap(gauges);
    }
}
```
- (2) 通过**BeanPostProcessor**处理器自动注册MetricSet对象Bean；

```java
public class UserDefinedMetricBeanPostProcessor implements BeanPostProcessor {

    private final Logger LOG = LoggerFactory.getLogger(getClass());

    private final MetricRegistry metrics = MetricBeans.getRegistry();

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {

        if (bean instanceof MetricSet) {
            MetricSet metricSet = (MetricSet) bean;
            if (!canRegister(beanName)) {
                return bean;
            }
            String metricName;
            if (isJvmCollector(beanName)) {
                metricName = Config.getProjectPrefix() + "." + beanName;
            } else {
                //根据规则生成Metric的名字
                metricName = Util.forMetricBean(bean.getClass(), beanName);
            }
            try {
                metrics.register(metricName, metricSet);
                LOG.debug("Registered metric named {} in registry. class: {}.", metricName, metricSet);
            } catch (IllegalArgumentException ex) {
                LOG.warn("Error injecting metric for field. bean named {}.", metricName, ex);
            }

        }
        return bean;
    }

    private boolean isJvmCollector(String beanName) {
        return beanName.indexOf("jvm") != -1;
    }

    private boolean canRegister(String beanName) {
        return !isJvmCollector(beanName) || Config.canJvmCollectorStart();
    }
}
```
- (3) 在spring xml文件或通过spring注解定义bean对象；

```
    <!--定义Jvm监控对象-->
    <bean id="jvm.memory" class="com.codahale.metrics.jvm.MemoryUsageGaugeSet"/>
    <!--自动添加用户定义的监控对象Metric-->
    <bean class="com.test.metrics.collector.UserDefinedMetricBeanPostProcessor"/>
```

可以根据需要定制MetricSet集合，实现服务指标的自动注册及上报。

#### 3.2、结合注解实现成员变量自动注册
我们可以结合注解实现成员变量的自动注册。在BeanPostProcessor可以获取到成员变量的注解，若是我们的目标注解，可以通过反射的方式获取到变量信息进行自动注册。

下面以Gauged注解为例说明，Gauged注解可以让成员变量自动注册并上报。
- (1)  Gauged注解定义；
```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.METHOD, ElementType.FIELD, ElementType.ANNOTATION_TYPE })
public @interface Gauged {
    String name() default "";
}
```
- (2) 使用**BeanPostProcessor**解析Gauge注解并注册；

核心代码如下所示：
```java
    protected void withField(final Object bean, String beanName, Class<?> targetClass, final Field field) {
        ReflectionUtils.makeAccessible(field);

        final Gauged annotation = field.getAnnotation(Gauged.class);
        final String metricName = Util.forGauge(targetClass, field, annotation);

        metrics.register(metricName, new Gauge<Object>() {
            @Override
            public Object getValue() {
                return ReflectionUtils.getField(field, bean);
            }
        });

        LOG.debug("Created gauge {} for field {}.{}", metricName, targetClass.getCanonicalName(), field.getName());
    }
```
- (3) 在spring.xml文件定义对应**BeanPostProcessor**即可使用。

基本使用如下：
```
@Component
public class GaugeUsage {

    @Gauged(name = "gaugeField")
    private int gaugedField = 999;
    
}
```

#### 3.3、结合注解实现方法切面的拦截统计
基于Spring AOP可以实现接口调用的耗时统计。

下面以Timed注解为例，Timed注解可以统计接口方法耗时情况。
- (1) Timed注解定义；
``` java
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.TYPE, ElementType.CONSTRUCTOR, ElementType.METHOD, ElementType.ANNOTATION_TYPE })
public @interface Timed {
    String name() default "";
}
```
- (2) Timed注解切面定义；
```java
@Component
@Aspect
public class MetricAspect {
    @Around("@annotation(timed)")
    public Object processTimerAnnotation(ProceedingJoinPoint joinPoint, Timed timed) throws Throwable {
        Class clazz = joinPoint.getTarget().getClass();
        Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
        String metricName = Util.forTimedMethod(clazz, method, timed);
        Timer timer = MetricBeans.timer(metricName);
        final Timer.Context context = timer.time();
        try {
            return joinPoint.proceed();
        } finally {
            context.stop();
        }
    }
}
```
- (3) 在spring.xml文件定义MetricAspect即可实现带Timed注解接口的请求频率和耗时统计。

基本示例如下：
```java
@Component
public class TimedUsage {

    //@Timed注解会让监控组件创建Timer对象，统计该方法的执行次数和执行时间等指标
    @Timed(name = "simple-timed-method")
    public void timedMethod() {
        for (int i = 0; i < 1000; i++) {
        }
    }
}
```
### 4、总结
当前我们主要通过<font color=green>**BenPostProcessor**</font>和<font color=green>**Spring AOP**</font>对类实例进行拦截，从而实现服务指标的自动注册和收集。

### 其它文章
[订单服务的设计思考](https://juejin.im/post/5c1e4d48f265da61120500e3)

[会员自动续费该如何实现](https://juejin.im/post/5c3d7102e51d454518502814)