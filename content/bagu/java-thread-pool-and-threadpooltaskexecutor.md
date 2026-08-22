---
title: 'Java 线程池'
date: 2026-08-21
draft: false
categories: ['Java']
tags: ['并发编程', 'Spring']
---

## 为什么要用线程池

线程的创建和销毁是有代价的（内核态切换、内存分配），高并发场景下频繁创建线程会拖垮系统。线程池的核心价值就三句话：

- **降低资源消耗**：复用已创建的线程，减少创建/销毁的开销
- **提高响应速度**：任务到达时无需等待线程创建，直接复用空闲线程
- **提高可管理性**：统一分配、调优和监控线程，避免无限制创建导致 OOM

## ThreadPoolExecutor 七大参数

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```

| 参数              | 含义                                                              |
| ----------------- | ----------------------------------------------------------------- |
| `corePoolSize`    | 核心线程数，即使空闲也会保留（除非开启 `allowCoreThreadTimeOut`） |
| `maximumPoolSize` | 线程池允许的最大线程数                                            |
| `keepAliveTime`   | 非核心线程空闲时的存活时间                                        |
| `unit`            | `keepAliveTime` 的时间单位                                        |
| `workQueue`       | 任务队列，存放等待执行的任务                                      |
| `threadFactory`   | 线程工厂，自定义线程名、优先级等                                  |
| `handler`         | 拒绝策略，队列满且线程达到上限时的处理方式                        |

## 执行流程（面试必背）

提交一个任务到线程池，处理顺序如下：

1. 线程数 < `corePoolSize`：直接创建**核心线程**执行任务
2. 线程数 ≥ `corePoolSize`：任务进入**队列**排队
3. 队列满了，线程数 < `maximumPoolSize`：创建**非核心线程**执行任务
4. 队列满了，线程数 = `maximumPoolSize`：触发**拒绝策略**

{{< alert icon="lightbulb" >}}
注意顺序是「队列优先于非核心线程」——很多人以为是先扩容到最大线程数再入队，这是最常见的记忆错误。
{{< /alert >}}

### 一个形象的比喻

把线程池想象成一家餐厅：

- **核心线程** = 正式员工，没客人也发工资养着
- **非核心线程** = 临时工，忙不过来才招，闲下来就辞退
- **队列** = 排队取号区
- **拒绝策略** = 号发完了直接不接客

## 四种拒绝策略

| 策略                  | 行为                                                   |
| --------------------- | ------------------------------------------------------ |
| `AbortPolicy`（默认） | 抛出 `RejectedExecutionException`                      |
| `CallerRunsPolicy`    | 由提交任务的线程自己执行，起到反压（backpressure）效果 |
| `DiscardPolicy`       | 静默丢弃任务，不抛异常                                 |
| `DiscardOldestPolicy` | 丢弃队列中最老的任务，然后重试提交                     |

生产环境的建议：**不要用默认的 AbortPolicy 裸奔**。要么用 `CallerRunsPolicy` 让调用方减速，要么自定义 handler 记录日志、落库、发告警。

## 常见阻塞队列的选择

| 队列                    | 特点                                                                   |
| ----------------------- | ---------------------------------------------------------------------- |
| `SynchronousQueue`      | 不存储元素，直接交接；配合无界最大线程数（对应 `newCachedThreadPool`） |
| `LinkedBlockingQueue`   | 默认无界（`Integer.MAX_VALUE`），可能堆积任务导致 OOM                  |
| `ArrayBlockingQueue`    | 有界，容量固定，需要预设大小                                           |
| `PriorityBlockingQueue` | 支持优先级排序                                                         |

## 线程数怎么设置

没有银弹，只有经验公式：

- **CPU 密集型**（计算为主）：`N + 1`，N 为 CPU 核心数。多出来的 1 防止缺页中断等暂停时 CPU 空闲
- **IO 密集型**（等待网络/磁盘为主）：`2N` 或按公式 `N × (1 + W/C)`（W 为等待时间，C 为计算时间）

更靠谱的方式是**压测**：上线前用真实流量模型压一压，观察吞吐和延迟再调整。动态调参可以参考美团那篇《Java 线程池实现原理及其在美团业务中的实践》（setCorePoolSize 等方法支持运行时修改）。

## 为什么不推荐 Executors 工厂方法

《阿里巴巴 Java 开发手册》强制规定不用 `Executors` 创建线程池：

| 方法                                             | 问题                                                     |
| ------------------------------------------------ | -------------------------------------------------------- |
| `newFixedThreadPool` / `newSingleThreadExecutor` | 队列是 `LinkedBlockingQueue` 无界，任务堆积 → **OOM**    |
| `newCachedThreadPool` / `newScheduledThreadPool` | 最大线程数为 `Integer.MAX_VALUE`，线程无限创建 → **OOM** |

正确姿势是手动 `new ThreadPoolExecutor(...)`，显式指定每个参数，做到心里有数。

## Spring 的 ThreadPoolTaskExecutor

Spring 没有重新发明轮子，`ThreadPoolTaskExecutor` 是对 JDK `ThreadPoolExecutor` 的**包装**（组合关系，内部持有一个 `ThreadPoolExecutor` 实例），主要增强了：

- **统一的配置方式**：以 Bean 的形式声明，参数可配置化
- **生命周期管理**：实现 `InitializingBean`/`DisposableBean`，容器启动时初始化、关闭时优雅 shutdown
- `TaskDecorator`：装饰任务，比如传递 MDC 日志上下文、捕获异常
- 与 Spring 的异步体系（`@Async`、`TaskExecutor` 接口）无缝集成

### 基本用法

```java
@Configuration
public class AsyncConfig {

    @Bean("taskExecutor")
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(16);
        executor.setQueueCapacity(200);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("my-task-");
        // 队列满时由调用线程执行，形成反压
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        // 等待所有任务完成再关闭，保证优雅停机
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);
        executor.initialize(); // 关键：初始化内部 ThreadPoolExecutor
        return executor;
    }
}
```

配合 `@Async` 使用：

```java
@Async("taskExecutor")  // 不指定名字则找默认的或唯一的 TaskExecutor
public void doAsyncWork() {
    // 异步逻辑
}
```

{{< alert icon="triangle-exclamation" >}}
`@Async` 的常见坑：

1. 同类内部调用失效（绕过了代理）——需要拆到另一个 Bean
2. 方法必须是 `public`
3. 默认返回 `void` 或 `Future`，其他返回值拿不到结果
4. 忘记加 `@EnableAsync`
   {{< /alert >}}

### ThreadPoolTaskExecutor vs ThreadPoolExecutor 对比

| 维度     | `ThreadPoolExecutor`                     | `ThreadPoolTaskExecutor`                             |
| -------- | ---------------------------------------- | ---------------------------------------------------- |
| 归属     | JDK（`java.util.concurrent`）            | Spring（`spring-core`）                              |
| 关系     | 底层实现                                 | 包装类，内部委托给 ThreadPoolExecutor                |
| 生命周期 | 手动管理                                 | 跟随 Spring 容器，自动初始化与销毁                   |
| 配置方式 | 构造参数，一次性                         | Setter 注入，支持配置文件绑定                        |
| 优雅停机 | 手动 `shutdown()` + `awaitTermination()` | `setWaitForTasksToCompleteOnShutdown(true)` 一行搞定 |
| 适用场景 | 纯 Java 项目、精细控制                   | Spring / Spring Boot 项目（默认选择）                |

还有一个容易忽略的点：两者的拒绝时机**略有差异**。`ThreadPoolTaskExecutor` 提交任务时如果线程池已 shutdown，同样会走拒绝策略，但 Spring 在 `submit` 前会先检查 `task == null` 并抛 `IllegalArgumentException`，注意区分异常来源。

### Spring Boot 中的自动配置

Spring Boot 2.1+ 通过 `TaskExecutionAutoConfiguration` 自动配置了一个 `ThreadPoolTaskExecutor`（Bean 名 `applicationTaskExecutor`，线程名前缀 `task-`），可以通过配置文件调整：

```yaml
spring:
  task:
    execution:
      pool:
        core-size: 8
        max-size: 16
        queue-capacity: 200
        keep-alive: 60s
      thread-name-prefix: my-task-
```

如果自己定义了 `Executor`/`TaskExecutor` Bean，自动配置会**退让**（`@ConditionalOnMissingBean`），以你的为准。

## 状态与生命周期

`ThreadPoolExecutor` 用一个 `AtomicInteger ctl` 的高 3 位表示状态、低 29 位表示线程数：

| 状态         | 含义                                                   |
| ------------ | ------------------------------------------------------ |
| `RUNNING`    | 接受新任务，处理队列任务                               |
| `SHUTDOWN`   | 不接受新任务，处理完队列任务（`shutdown()` 触发）      |
| `STOP`       | 不接受新任务，中断进行中的任务（`shutdownNow()` 触发） |
| `TIDYING`    | 所有任务结束，线程数为 0                               |
| `TERMINATED` | `terminated()` 钩子执行完毕，彻底终止                  |

```java
executor.shutdown();                     // 温和：不再收新任务，处理完存量
executor.shutdownNow();                  // 激进：中断所有任务，返回未执行的任务列表
executor.awaitTermination(30, SECONDS);  // 阻塞等待终止
```

## 高频面试题速查

1. **核心线程可以被回收吗？** 可以，`allowCoreThreadTimeOut(true)` 后核心线程也会超时回收
2. **提交方式 `execute` vs `submit`？** `execute` 无返回值；`submit` 返回 `Future`，异常被封装进 Future，不调用 `get()` 就发现不了
3. **线程池中线程抛异常会怎样？** 该线程死亡，线程池补一个新线程；异常默认无感知，建议在任务内 try-catch 或用 `submit` + `Future.get()`
4. **如何监控线程池？** 定期采集 `getActiveCount`、`getCompletedTaskCount`、`getQueue().size()`、`getLargestPoolSize()` 等指标上报
5. **Spring 中动态修改线程池参数？** `ThreadPoolTaskExecutor` 也提供了 `setCorePoolSize` 等运行时修改方法（委托给内部的 `ThreadPoolExecutor`），可结合配置中心实现动态调参
