---
type: Note
---
# Java 线程池最佳实践：从「会用」到「精通」

> **一句话总结**：线程池不是用 `Executors` 一键生成的魔法黑盒，而是由 `ThreadPoolExecutor` 七大参数构成的精密调度器。本文从"线程很贵"这个起点讲起，带你吃透核心参数、任务提交流程、四种拒绝策略、参数调优、优雅停机、异常处理与线程池隔离，划出"严禁 Executors""必须用有界队列""给线程起名""ThreadLocal 记得清理"等高压线，最后探讨虚拟线程时代线程池的定位与取舍。

> **阅读路线**：本篇延续本知识库"前端工程师 → Java 全栈"的进阶体系，前置阅读 [破茧成蝶：前端工程师的Java线程全栈进阶指南](./破茧成蝶-前端工程师的java线程全栈进阶指南.md)。如果你已经会写 `ExecutorService.submit(...)`，本文帮你把"会用"升级为"懂原理、会排障、能调优"。

***

## 一、 为什么需要线程池：线程是昂贵的资源

很多前端转全栈的同学，第一次写多线程代码时的直觉是：

```java
new Thread(() -> {
    // 干点活
}).start();
```

这个写法在 demo 里没问题，但在生产环境里是灾难。**因为线程的创建和销毁成本极高**：

1. **创建成本**：`new Thread()` 底层要调用操作系统 API，申请内核级资源、分配栈内存（JVM 默认线程栈 `-Xss` 可达 1MB）、注册到调度器。JVM 自己都无法用纯 Java 写出"创建线程"的代码，必须交给 native 方法。
2. **切换成本**：CPU 在线程间切换时，需要保存/恢复上下文（寄存器、程序计数器、栈指针），这叫**上下文切换（Context Switch）**。线程越多，切换越频繁，花在"搬进搬出"上的时间占比越高，有效吞吐反而下降。
3. **数量瓶颈**：一个进程可创建的线程数是有限度的（受内存、系统限制），无脑 `new Thread` 一旦任务高峰到来，操作系统直接崩溃或 OOM。

**触类旁通**：前端里你不会为每个请求 `new Web Worker()`，因为 Worker 同样昂贵；你复用它们，本质上是同一个道理。

### 线程池解决什么问题

线程池的核心思想是**"线程复用 + 数量管控"**。想象一家银行：

| 线程池 | 银行柜台 |
|---|---|
| 核心线程数 `corePoolSize` | 常驻的固定窗口数 |
| 阻塞队列 `workQueue` | 等候区的座位数 |
| 最大线程数 `maximumPoolSize` | 高峰期可临时加开的窗口数 |
| 空闲存活时间 `keepAliveTime` | 临时窗口多久没客户就撤 |
| 拒绝策略 `handler` | 大厅满员、排队也满了之后怎么办 |

线程池带来三大收益：

- **复用**：线程执行完任务不销毁，回池子待命，避免反复创建/销毁的开销；
- **限流（背压）**：用"有界队列 + 最大线程数"给系统一个上限，流量暴增时不会被瞬间打垮，而是排队/拒绝/降级；
- **生命周期管理**：统一创建、统一回收、优雅停机，避免线程泄漏。

***

## 二、 解剖 ThreadPoolExecutor：七大核心参数

`ThreadPoolExecutor` 是线程池的正主，`Executors` 那些便捷方法不过是它的花式封装。它的构造方法有 7 个参数：

```java
new ThreadPoolExecutor(
    int corePoolSize,      // ① 核心线程数
    int maximumPoolSize,   // ② 最大线程数
    long keepAliveTime,    // ③ 非核心线程空闲存活时间
    TimeUnit unit,         // ④ keepAliveTime 的时间单位
    BlockingQueue<Runnable> workQueue, // ⑤ 任务队列
    ThreadFactory threadFactory,       // ⑥ 线程工厂
    RejectedExecutionHandler handler   // ⑦ 拒绝策略
);
```

逐一拆解：

### ① corePoolSize —— 核心线程数
正常情况下一直存活的工作线程数量。核心线程默认**懒加载**：刚创建线程池时一个线程都没有，任务来了才逐个创建；也可以调用 `prestartAllCoreThreads()` 提前全部启动（常用于对首请求延迟敏感的服务）。

### ② maximumPoolSize —— 最大线程数
线程池允许的最大工作线程数，是"核心线程 + 临时线程"的总上限。临时线程（非核心线程）空闲超过 `keepAliveTime` 会被回收，但核心线程默认不会（除非开启 `allowCoreThreadTimeOut(true)`）。

### ③④ keepAliveTime + TimeUnit —— 非核心线程空闲存活时间
临时线程在没有任务可做时，最多存活多久。注意它**只对超过核心线程数的临时线程生效**。这个参数决定系统在低峰期能否把多余线程回收，释放资源。

### ⑤ workQueue —— 任务队列
核心线程全部忙碌时，新任务会先进入这个队列排队等待。队列选择直接决定线程池的行为（详见第五节），**一句话：生产环境请用有界队列**。

### ⑥ threadFactory —— 线程工厂
负责创建新线程的工厂。**必须自定义并给线程起名字**（默认叫 `pool-1-thread-1`，一旦线上出问题，线程 dump 你根本分不清是哪个业务池子的线程）。详见第九节。

### ⑦ handler —— 拒绝策略
当"线程数已达 `maximumPoolSize` 且队列也满了"，新任务无处安放，交给拒绝策略处理。详见第四节。

> **严谨提示**：`corePoolSize` 是"保底存活"，`maximumPoolSize` 是"极限扩容"，队列是"中间的缓冲地带"。三者构成一条完整的任务流水线，缺一不可。

***

## 三、 任务提交流程：线程池的「扩容逻辑」

理解线程池，最关键的是记住它的**任务提交流程**——面试必考，排障必用。用 JDK 源码（`execute` 方法）提炼出的逻辑如下：

1. **核心未满，直接上**：如果当前工作线程数 < `corePoolSize`，创建一个新核心线程来执行这个任务；
2. **核心已满，先排队**：否则尝试把任务放入 `workQueue`。入队成功就等空闲线程取走；
3. **队列已满，临时扩容**：如果队列也满了，且当前线程数 < `maximumPoolSize`，创建一个**非核心线程**来执行；
4. **全线告急，拒绝**：如果线程数已达 `maximumPoolSize` 且队列已满，交给拒绝策略。

伪代码直译：

```java
public void execute(Runnable command) {
    // ① 核心线程没满 → 新建核心线程执行
    if (workerCountOf(ctl.get()) < corePoolSize) {
        if (addWorker(command, true)) return;
    }
    // ② 核心线程满了 → 尝试入队（入队后还会二次检查线程池状态）
    if (isRunning(ctl.get()) && workQueue.offer(command)) {
        // ... 二次检查，若线程池已关闭则移除任务并拒绝
    }
    // ③ 队列满了 → 尝试创建非核心线程
    else if (!addWorker(command, false)) {
        // ④ 线程数已达 maximumPoolSize → 拒绝
        reject(command);
    }
}
```

**核心洞察：线程池是"先排队，后扩容"的。** 当核心线程都在忙、但队列还没满时，新任务会**乖乖排队**，而不是立刻再开新线程。只有队列也装不下了，才会动用 `maximumPoolSize` 的"预备队"。这个设计是有意的——临时线程是稀缺且昂贵的资源，能用队列缓冲就先缓冲。

这也解释了为什么 `maximumPoolSize` 与 `workQueue` 的搭配如此关键：**队列越大，越难触发扩容，任务堆积越深（延迟越高）；队列越小，越容易扩容和拒绝（吞吐峰值更高但更激进）**。

***

## 四、 拒绝策略：最后一道防线

当线程池彻底"装不下"时，`RejectedExecutionHandler` 登场。JDK 内置四种策略：

| 策略 | 行为 | 代价 | 适用场景 |
|---|---|---|---|
| `AbortPolicy`（默认） | 直接抛 `RejectedExecutionException` | 调用方要自己捕获，否则异常向上传播 | 失败必须被感知的任务 |
| `CallerRunsPolicy` | 由**提交任务的调用方线程**自己执行这个任务 | 调用方线程被占用，无法继续提交新任务 → 天然**背压** | 全链路削峰兜底，**最推荐** |
| `DiscardPolicy` | 静默丢弃新任务，不抛异常 | 任务无声消失，难以发现 | 允许丢弃的埋点/统计类任务 |
| `DiscardOldestPolicy` | 丢弃队列中**最老**的任务，再尝试提交当前任务 | 老任务丢失 | 追求"最新"数据的任务（如实时价格推送） |

### 为什么 `CallerRunsPolicy` 最推荐

它的巧妙之处在于**把压力回传给调用方**。假设线程池已被打满，此时调用方线程自己动手执行任务——它忙着干活，就腾不出手来提交新任务，提交速度自然被掐住，系统像一个"弹簧"一样自我调节。前端世界里，这就像 Event Loop 的主线程被长任务占住后，后面的任务自然排队——用"阻塞"换取"不丢"。

自定义策略也很简单，实现 `RejectedExecutionHandler` 接口即可，常见的做法是"记录告警 + 降级处理"：

```java
public class AlarmRejectedHandler implements RejectedExecutionHandler {
    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        log.error("线程池已满！核心={} 最大={} 队列={}",
                e.getPoolSize(), e.getMaximumPoolSize(), e.getQueue().size());
        // 降级：写入 MQ/告警、或走兜底逻辑
    }
}
```

***

## 五、 高压红线：为什么严禁用 Executors

《阿里巴巴Java开发手册》有一条著名的强制规定，原话大意：

> 线程池不允许使用 `Executors` 去创建，而是通过 `ThreadPoolExecutor` 的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。

为什么？因为 `Executors` 的便捷方法都踩了"无界"的坑：

| 工厂方法 | 底层实现 | 隐患 |
|---|---|---|
| `newFixedThreadPool(n)` | `LinkedBlockingQueue`（**无界**） | 任务无限堆积占满内存 → **OOM** |
| `newSingleThreadExecutor()` | 同 `FixedThreadPool`（1 线程 + 无界队列） | 同上 |
| `newCachedThreadPool()` | `SynchronousQueue` + `maximumPoolSize = Integer.MAX_VALUE` | 线程无限创建 → **OOM** |
| `newScheduledThreadPool(n)` | `DelayedWorkQueue`（无界） | 同上 |

- **FixedThreadPool 的坑**：队列无界，意味着永远触达不到拒绝策略。核心线程忙不过来时，任务全堆进队列。如果生产者速度长期大于消费速度，队列里的任务越积越多，最终堆满堆内存 → OOM，而且**进程挂之前毫无征兆**，你只会看到 GC 频繁、Full GC 不停。
- **CachedThreadPool 的坑**：`maximumPoolSize` 是 `Integer.MAX_VALUE`，一个请求高峰就能创建成千上万个线程。线程本身吃内存、吃 CPU（调度），创建速度远超系统承受能力时，直接 OOM 或把机器打瘫。

**正确的姿势是手动 `new ThreadPoolExecutor`，明确每一个参数**：

```java
private static final ThreadPoolExecutor ORDER_POOL = new ThreadPoolExecutor(
        8,                                  // 核心线程数
        16,                                 // 最大线程数（高峰临时扩容上限）
        60L, TimeUnit.SECONDS,              // 非核心线程空闲 60s 回收
        new ArrayBlockingQueue<>(2000),     // ★ 有界队列：挡住任务无限堆积
        new NamedThreadFactory("order-async"), // ★ 自定义线程工厂：命名
        new ThreadPoolExecutor.CallerRunsPolicy() // ★ 拒绝策略：调用者执行（背压）
);
```

> **一句话**：`Executors` 帮你省的是"三行代码"，让你埋的是"OOM 的雷"。参数写清楚，本身就是一种防御性设计。

***

## 六、 参数调优：公式只是起点，压测才是终点

设置线程数没有银弹，但业界有几条经验公式可以当**起点**。

### 1. 按任务类型区分

- **CPU 密集型**（大量计算，几乎没有等待）：JDK 官方 javadoc 建议 `核心线程数 ≈ Ncpu + 1`。因为 CPU 密集型任务几乎不阻塞，线程超过核数反而因上下文切换而变慢。
- **IO 密集型**（大量等待：网络、磁盘、RPC）：《Java Concurrency in Practice》（Brian Goetz）给出的经典公式为 `线程数 = Ncpu × (1 + W/C)`，其中 `W` 是等待时间，`C` 是计算时间。工程上常先用 `2 × Ncpu` 起步再压测。

### 2. 队列容量怎么定

队列大小需要结合**任务产生速度**和**可容忍的积压延迟**来估算：

- 公式：`队列容量 ≈ 单任务处理耗时 × 每秒任务量 × 允许积压秒数`；
- 经验：`ArrayBlockingQueue` 容量常取 `corePoolSize` 的 10~20 倍作为起点；
- 原则：**宁小勿大**。队列太大 → 任务堆积无人知、延迟飙升；队列太小 → 过早触发扩容和拒绝，线程频繁创建/回收。

### 3. 动态调优：让线程池"热更新"

`ThreadPoolExecutor` 提供了两个 setter，可以在运行时动态调整参数，非常适合**线上灰度调参**：

```java
// 提升核心线程数（扩容）
orderPool.setCorePoolSize(12);
// 修改最大线程数
orderPool.setMaximumPoolSize(24);
// 让核心线程空闲超时后也可回收（低峰期省资源）
orderPool.allowCoreThreadTimeOut(true);
```

再配合监控（见第九节），就能做到"先压测预估 → 上线观察 → 动态微调"，而不是拍脑袋定死。

> **最终原则**：公式给的是"合理的起点"，**真正的答案永远来自压测**。结合压测时的吞吐量、P99 延迟、线程活跃度、队列积压四个指标，迭代调整。

***

## 七、 优雅停机：shutdown / shutdownNow / awaitTermination

线上发布、重启时，最怕的是**线程池里还有任务在跑，进程就被强杀了**。优雅停机是线程池的"体面退场"。

两个关闭方法的对比：

| 方法 | 行为 | 是否等队列中任务 | 返回值 |
|---|---|---|---|
| `shutdown()` | 停止接收**新任务**，已提交的任务（含队列中的）继续执行完 | 是 | 无 |
| `shutdownNow()` | 尝试中断正在执行的任务，**清空队列**并返回未执行的任务列表 | 否 | `List<Runnable>`（被丢弃的任务） |

**推荐姿势**：先 `shutdown()` 温婉地关，等一段超时；等不完再 `shutdownNow()` 强制收尾：

```java
// ① 不再接收新任务，存量任务继续执行
executor.shutdown();
// ② 等待存量任务执行完（带超时，避免无限阻塞）
if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
    // ③ 超时未完成 → 强制中断，并拿到被丢弃的任务列表
    List<Runnable> dropped = executor.shutdownNow();
    log.warn("强制关闭线程池，丢弃 {} 个未执行任务", dropped.size());
    // ④ 中断后仍需等待任务响应 interrupt
    executor.awaitTermination(10, TimeUnit.SECONDS);
}
```

两点注意：

- **`shutdownNow()` 只是发中断信号**，任务如果不响应 `InterruptedException` 或忽略中断标志，依旧停不下来，所以④还要再等一轮；
- **web 应用慎用 `shutdown()` 后马上释放 Spring 容器**：要确保线程池关闭逻辑排在所有"往池里提交任务"的代码之后，否则会出现 `RejectedExecutionException`。Spring 的 `@PreDestroy` 或 `DisposableBean` 是常见的托管位置。

***

## 八、 异常处理：execute 与 submit 的「冰火两重天」

线程池里的异常处理是**重灾区**，因为 `execute` 和 `submit` 对异常的处理截然不同：

| 提交方式 | 异常去哪了 | 风险 |
|---|---|---|
| `execute(Runnable)` | 交给工作线程的 `UncaughtExceptionHandler`（默认打印到 stderr），该线程销毁、线程池**补一个新线程** | 异常"泄漏"到系统日志，无法在业务层感知和记录 |
| `submit(Runnable/Callable)` | 被封装进 `FutureTask`，**异常被吞进 `Future` 里**；你不调 `Future.get()` 就永远不知道它失败了 | **静默失败，最危险**，看起来"成功了"实际全挂了 |

**submit 的静默失败**是线上事故的头号嫌疑人：任务全挂了，日志一片安静，只有监控里"结果数为 0"才暴露问题。

### 解法一：统一用钩子方法 `afterExecute` 捕获

`ThreadPoolExecutor` 提供了 `afterExecute(Runnable r, Throwable t)` 钩子，**推荐重写它统一记录异常**。注意 JDK javadoc 明确说明：通过 `submit` 提交的任务，异常被 `FutureTask` 捕获，`t` 参数是 `null`，需要手动从 `Future` 里取：

```java
@Override
protected void afterExecute(Runnable r, Throwable t) {
    super.afterExecute(r, t);
    // submit() 的任务异常被 FutureTask 吞掉，t 为 null，需要手动取出
    if (t == null && r instanceof Future<?>) {
        try {
            ((Future<?>) r).get();
        } catch (CancellationException ce) {
            t = ce;
        } catch (ExecutionException ee) {
            t = ee.getCause(); // 真正的业务异常
        } catch (InterruptedException ie) {
            Thread.currentThread().interrupt();
        }
    }
    if (t != null) {
        log.error("线程池任务执行异常", t);
    }
}
```

### 解法二：全局兜底 `UncaughtExceptionHandler`

配合自定义线程工厂，给每个线程设置兜底处理器，捕获 `execute` 提交的任务抛出的异常：

```java
Thread t = new Thread(r, prefix + "-" + seq.getAndIncrement());
t.setUncaughtExceptionHandler((thread, e) ->
        log.error("线程 {} 意外终止", thread.getName(), e));
```

> **一句话**：用 `submit` 必须 `get()`（哪怕只是拿异常）；不想管 `Future`，就统一走 `afterExecute` 钩子 + `UncaughtExceptionHandler` 双保险。

***

## 九、 给线程起名字 + 监控告警

### 为什么线程命名是"硬需求"

默认线程名叫 `pool-1-thread-1`。想象一个线上故障：服务突然 CPU 飙升，你 dump 线程栈，看到 50 个 `pool-X-thread-Y` 卡在某个方法里——**你根本不知道这是哪个业务的线程池**。排查全靠猜。

自定义 `ThreadFactory`，名字带上业务语义，故障时一眼定位：

```java
public class NamedThreadFactory implements ThreadFactory {
    private final AtomicInteger seq = new AtomicInteger(1);
    private final String prefix;

    public NamedThreadFactory(String prefix) { this.prefix = prefix; }

    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r, prefix + "-" + seq.getAndIncrement());
        t.setUncaughtExceptionHandler(...); // 见第八节
        return t;
    }
}
```

（生产上可以直接用 Guava 的 `ThreadFactoryBuilder` 或 Hutool 的 `ThreadFactoryBuilder`，一行搞定命名 + 守护线程标志。）

### 监控哪些指标

`ThreadPoolExecutor` 本身暴露了一组自带指标：

| 方法 | 含义 | 排查价值 |
|---|---|---|
| `getPoolSize()` | 当前线程数 | 是否还在扩容/收缩 |
| `getActiveCount()` | 正在执行任务的线程数 | 与 poolSize 接近 → 线程都满了 |
| `getLargestPoolSize()` | 历史最大线程数 | 判断是否曾经触发过扩容 |
| `getTaskCount()` | 历史累计提交任务数 | 与 completed 对比算积压 |
| `getCompletedTaskCount()` | 已完成任务数 | 结合 TaskCount 看丢没丢任务 |
| `getQueue().size()` | 队列中排队任务数 | **积压预警的核心指标** |
| `getRejectedExecutionCount()`（自定义） | 拒绝次数 | 拒绝策略触发了多少回 |

把这些指标接入 Micrometer + Prometheus，核心告警就两条：

1. **队列积压超过阈值**（如容量 80%）→ 消费者跟不上了；
2. **拒绝次数 > 0** → 线程池被打满，开始丢任务/背压。

> **前端触类旁通**：这就好比给异步任务加上 Performance Timing 和错误上报——没有监控的异步代码，等于把系统状态扔进黑箱。

***

## 十、 线程池隔离：防止"一粒老鼠屎坏一锅汤"

一个应用里往往同时跑着多条业务线：下单、导报表、发短信。如果所有业务共用一个线程池，那么**慢任务会拖垮所有业务**：

```java
// ❌ 错误示范：所有业务共用一个池
//    报表导出这种耗时任务一旦占满线程，会把下单请求也堵死
ThreadPoolExecutor sharedPool = new ThreadPoolExecutor(8, 16, 60L, TimeUnit.SECONDS, ...);

// ✅ 正确示范：按业务拆分，各自为政（Bulkheading / 故障隔离）
ThreadPoolExecutor orderPool   = new ThreadPoolExecutor(8,  16, 60L, TimeUnit.SECONDS, ...);
ThreadPoolExecutor reportPool  = new ThreadPoolExecutor(4,   8, 60L, TimeUnit.SECONDS, ...);
ThreadPoolExecutor smsPool     = new ThreadPoolExecutor(2,   4, 60L, TimeUnit.SECONDS, ...);
```

**隔离的价值**：

- 报表任务把 `reportPool` 打满，`orderPool` 里的下单丝毫无损；
- 每个池的参数（线程数、队列、拒绝策略）可以**按业务特性独立调优**——报表可以队列大一点、线程少一点，下单可以更激进地扩容；
- 出问题时，故障被"圈"在单个池内，不再殃及池鱼。

**触类旁通**：这就像前端把不同模块用 Error Boundary 分开——A 组件崩了，整个页面不至于白屏。线程池隔离的本质是**故障隔离（Bulkheading）**，把"共享失败域"拆成"独立失败域"。

> 判断标准：**并发特性差异大、或相互不能容忍阻塞的业务，一定要拆池**。反之，一群低风险、低并发的短任务共用一个池问题也不大，但务必监控（见第九节）。

***

## 十一、 细节魔鬼：ThreadLocal、线程池嵌套、无界队列

### 1. ThreadLocal：线程复用的"记忆残留"

线程池的线程是**复用的**，而 `ThreadLocal` 是**跟线程绑定**的——这俩组合会出大问题：

```java
// ❌ 危险写法
public void handle(String userId) {
    UserContext.set(userId);       // 写入 ThreadLocal
    // ... 业务逻辑
    // 没有 remove()！线程被池子回收复用，userId 残留
}

// ✅ 正确写法：finally 中清理
public void handle(String userId) {
    try {
        UserContext.set(userId);
        // ... 业务逻辑
    } finally {
        UserContext.remove();      // 必须清理
    }
}
```

两个隐患：

- **数据串味**：下一个任务复用同一线程时，读到上一个任务的 `ThreadLocal`，张冠李戴（比如拿到别人的用户上下文）；
- **内存泄漏**：线程池线程是长生命周期对象，`ThreadLocalMap` 强引用 value，即使业务对象早该被 GC，也一直被线程池线程拽着不放。

> 如果要在任务里传递"调用链上下文"（如日志 MDC、traceId），用 `TaskDecorator`（Spring 的 `ThreadPoolTaskExecutor` 支持）在包装任务时**复制进、执行后清理**，不要依赖隐式残留。

### 2. 线程池嵌套：递归提交的隐患

在一个线程池任务里再向（自己或另一个）线程池提交任务，如果内部池也被打满，就会形成"生产者被自己堵死"的连锁反应。尤其配合 `CallerRunsPolicy`，父任务的线程被内部任务占用，层层叠加，最终可能把提交线程也拖垮。**尽量保持单层异步，明确谁在哪个池子里执行。**

### 3. 无界队列是"定时炸弹"

再强调一次第五节的核心：**有界队列 + 明确拒绝策略** 是生产环境底线。无界队列的"看似永远装得下"其实是在用内存换延迟，最终以 OOM 收场。

### 4. CompletableFuture 别默认走公共池

`CompletableFuture.supplyAsync(...)` 不带线程池参数时，默认使用 JVM 全局共享的 `ForkJoinPool.commonPool()`（线程数 ≈ `CPU核数 - 1`），全应用不可控。并发较高的异步编排，**请显式传入自定义线程池**：

```java
CompletableFuture.supplyAsync(() -> queryUserInfo(userId), orderPool);
```

否则一个应用里所有"偷懒"的 CompletableFuture 会互相挤占公共池，谁流量大谁遭殃。

### 5. 其他容易被忽略的细节

- **`submit` 一个非常耗时的任务占满核心线程** → 后续任务全进队列，延迟飙升。必要时给任务加超时（`Future.get(timeout)` 配合取消）。
- **线程池作为 static 单例**：不要在每次请求里 `new` 一个线程池，那等于变相 `new Thread`。
- **`shutdown()` 之后再 `submit`** 会立刻抛 `RejectedExecutionException`，提交方要做好兜底。

***

## 十二、 虚拟线程来了，线程池过时了吗？

Java 21 正式发布了**虚拟线程（Virtual Threads）**，这引出一个真问题：线程池会被淘汰吗？

回顾线程池的动机：**线程昂贵 → 所以要复用、要管控数量**。而虚拟线程是 JVM 调度的轻量级线程，创建成本比 OS 线程低几个数量级，一台机器轻松跑几十万甚至上百万个。官方文档的原话思路是：

> Virtual threads are lightweight, so there is no need to pool them. —— 虚拟线程足够轻量，**没必要池化**。

```java
// Java 21：每个任务一个虚拟线程，无需手动管线程数
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    tasks.forEach(task -> executor.submit(task));
}
```

**所以结论不是"线程池过时了"，而是"线程池的职责变了"**：

| 场景 | 用什么 |
|---|---|
| 海量 **IO 密集型**短任务（RPC、DB 访问、消息处理） | **虚拟线程**优先，每个任务一个，无需调参 |
| **CPU 密集型**任务、需要严格并发上限/背压的场景 | 仍需**传统线程池**（池子天然是并发限流器） |
| 需要精细控制排队、拒绝策略、监控告警的业务池 | 传统线程池依然适用，虚拟线程没有队列概念 |

一句话：**虚拟线程解决了"线程贵"的问题，线程池里另一半价值——"限流、排队、拒绝、监控"——依然不可替代**。很多团队现在是"虚拟线程跑业务 + 传统线程池做资源管控"的组合拳。

***

## 结语

把整篇文章浓缩成一张自检清单，写线程池前对照一遍：

- [ ] **手动 `new ThreadPoolExecutor`**，不使用 `Executors`；
- [ ] **队列用有界队列**（`ArrayBlockingQueue`/有界 `LinkedBlockingQueue`）；
- [ ] **明确拒绝策略**，兜底优先 `CallerRunsPolicy`；
- [ ] **自定义 `ThreadFactory`** 给线程起业务名字；
- [ ] **参数有据可依**：CPU/IO 公式起步，压测 + 动态调优收尾；
- [ ] **优雅停机**：`shutdown → awaitTermination → shutdownNow → awaitTermination`；
- [ ] **异常不漏**：`afterExecute` 钩子 + `UncaughtExceptionHandler` 双保险；
- [ ] **ThreadLocal 用必 `remove`**（或 TaskDecorator 托管上下文）；
- [ ] **指标进监控**：队列积压、拒绝次数、活跃线程数三条告警；
- [ ] **按业务拆池**：并发特性差异大的业务隔离，防止相互拖垮；
- [ ] **区分场景**：IO 密集型优先虚拟线程，CPU 密集型/需背压时用传统线程池。

从前端走向全栈，线程池是你交出的第一份"后端思维"答卷。前端的世界里，异步是"时间片里见缝插针"；后端的世界里，并发是"资源池里有舍有得"。理解线程池的每一个参数，本质上是在理解**如何在不确定性中为系统划定边界**——这正是全栈工程师最核心的能力。
