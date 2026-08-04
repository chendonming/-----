---
type: Note
---
# Java 线程池最佳实践（源码篇）：读透 ThreadPoolExecutor，从原理到排障

> **一句话总结**：网上所有线程池最佳实践——"禁止 Executors""必须用有界队列""submit 会吞异常""ThreadLocal 记得 remove"——本质都是源码里某一个分支的必然结果。本文不满足于"记住红线"，而是带你逐行读 `ThreadPoolExecutor` 的 `execute`、`Worker`、`getTask`、拒绝策略、`FutureTask`，搞懂每条规则**为什么**成立，再用三个真实事故复盘把"源码知识"变成"排障直觉"。

> **阅读路线**：本知识库已有两篇线程池文章——[《线程池炼金术：Java 线程池最佳实践避坑指南》](./从前端到全栈-Java线程池最佳实践.md)（最佳实践全景，前端视角）与[《Java 线程池最佳实践：从「会用」到「精通」》](./Java线程池最佳实践-从会用到精通.md)（完整七参数体系）。这两篇回答"该怎么做"，本文回答"**为什么该这么做**"——适合已会写 `ThreadPoolExecutor`、想要更深一层、或正在为线上线程池事故头疼的读者。

***

## 一、 为什么读源码：最佳实践是"果"，源码才是"因"

先问一个问题：为什么《阿里巴巴 Java 开发手册》要**强制**不用 `Executors`？

如果只记结论，你记住的是"因为有无界队列"。但如果你读过源码，你会看到更完整的画面：

```java
// Executors.newFixedThreadPool 的真实实现
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
            0L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<Runnable>());   // ← 默认容量 Integer.MAX_VALUE
}
```

```java
// LinkedBlockingQueue 无参构造的真实实现
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);   // ← 所谓的"无界"，其实是容量 = 21 亿
}
```

看到这一层，你就理解了：**"无界队列"不是一种抽象概念，而是一个具体到 `Integer.MAX_VALUE` 的容量**。队列能装 21 亿个任务，在内存面前约等于无限。最佳实践说"不要用无界队列"，本质上是在说"不要让内存替你的架构师兜底"。

这就是本文的路线：**每一条最佳实践，都在源码里有一个精确的对应点**。把对应点找到，规则就变成推导，记不住也不会用错。

***

## 二、 总览：ThreadPoolExecutor 的三张「设计蓝图」

读 `ThreadPoolExecutor` 源码前，先建立三张地图，否则容易迷路。

### 蓝图一：`ctl` —— 一个 int 同时管"状态"和"人数"

线程池的状态和线程数必须**原子地**一起变更（比如"关停"和"线程数归零"必须是一套动作），所以 JDK 用一个 `AtomicInteger` 把它们编码进同一个 int：

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

private static final int COUNT_BITS = Integer.SIZE - 3;    // 29
private static final int CAPACITY   = (1 << COUNT_BITS) - 1; // 536_870_911

// 高 3 位 → 运行状态；低 29 位 → 工作线程数
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
```

### 蓝图二：五个状态 —— 线程池是一台「状态机」

线程池的一生只有五个状态，且**只能沿一个方向前进，不能回头**：

| 状态 | 值（高 3 位） | 行为 | 如何进入 |
|---|---|---|---|
| `RUNNING` | -1 | 接收新任务，处理队列 | 创建时的初始状态 |
| `SHUTDOWN` | 0 | **不接收**新任务，存量任务继续 | `shutdown()` |
| `STOP` | 1 | 不接收新任务、**中断所有**线程、丢弃队列 | `shutdownNow()` |
| `TIDYING` | 2 | 线程数归零，即将执行 `terminated()` 钩子 | 条件自动迁移 |
| `TERMINATED` | 3 | 最终状态 | `terminated()` 执行完毕 |

> **触类旁通**：这很像前端 Promise 的状态机——`Pending → Fulfilled/Rejected`，同样是单向不可逆。线程池的 `awaitTermination()` 之所以能"等"，就是在等 `TIDYING → TERMINATED` 这一步走完。

### 蓝图三：`Worker` —— 真正干活的"工人"

线程池不直接管理 `Thread`，而是用一个 `Worker` 包装线程：

```java
private final class Worker extends AbstractQueuedSynchronizer
        implements Runnable {
    final Thread thread;    // 真正执行任务的线程
    Runnable firstTask;     // 第一个任务；null 表示"去队列里取"
}
```

注意一个精妙的设计：**`Worker` 继承自 AQS（`AbstractQueuedSynchronizer`）**。它不是为了实现队列，而是为了把"独占锁"能力复用到 `Worker` 上——运行任务的 worker 会 `lock()`，空闲的 worker 处于未锁状态。这样线程池就能用 `tryLock()` **区分"正在干活的线程"和"空闲的线程"**（第五节会看到 `shutdown` 靠它只打断空闲线程）。

***

## 三、 `execute()` 精读：扩容决策树 + 二次检查（recheck）

这是线程池最核心的方法，也是网上"四步提交流程"的原始出处。但网上版本往往**漏掉了最精妙的一步**——`recheck`。逐行读：

```java
public void execute(Runnable command) {
    if (command == null) throw new NullPointerException();
    int c = ctl.get();

    // ① 线程数 < corePoolSize：直接开核心线程执行
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true)) return;
        c = ctl.get();                 // 开线程失败（比如池已关闭），重读状态
    }

    // ② 池在运行 && 入队成功
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();       // ★ 二次检查：入队后状态可能已变
        if (!isRunning(recheck) && remove(command))   // 池刚被关闭 → 撤单
            reject(command);
        else if (workerCountOf(recheck) == 0)         // 池里一个线程都没了 → 补一个
            addWorker(null, false);
    }

    // ③ 队列满了 → 尝试开非核心线程
    else if (!addWorker(command, false)) {
        // ④ 线程数已达 maximumPoolSize → 拒绝
        reject(command);
    }
}
```

把决策树画出来：

```
新任务 execute(command)
  │
  ├─ ① 线程数 < corePoolSize ? ──是──▶ addWorker(command, core) 直接干
  │
  ├─ ② 池在运行 && 队列入队成功 ? ──是──▶ 【recheck 二次检查】
  │       ├─ 池已非运行 → remove 撤单 + reject
  │       └─ 线程数为 0   → addWorker(null, false) 补个"取队列"的线程
  │
  ├─ ③ 队列满，线程数 < maximumPoolSize ? ──是──▶ addWorker(command, 非核心)
  │
  └─ ④ 全满 ──▶ reject(command) 交给拒绝策略
```

### 为什么必须"二次检查"？

因为 `execute` 是**多线程并发调用的**，入队成功的那一刻，线程池可能已经发生了两件"意外"：

1. **池刚被 `shutdown()`**：一个任务刚入队，另一个线程就调了 `shutdown()`，状态变成 `SHUTDOWN`。此时这个任务即使排进队列也没人执行了，所以 `recheck` 发现"不在运行"就把它从队列 `remove` 掉并 `reject`。
2. **池里线程恰好全挂了**：核心线程因为 `beforeExecute`/`afterExecute` 抛异常等原因死亡，队列里排着任务却没人消费。所以 `recheck` 发现 `workerCount == 0`，就补一个 `firstTask = null` 的 worker，专门去队列里捞任务。

> **最佳实践 × 源码**：这一步解释了为什么"**先入队、后开非核心线程**"——JDK 用队列做缓冲，队列都满了才舍得开临时线程。理解了 `recheck`，你才真正明白为什么线程池是"**先排队、后扩容**"。

***

## 四、 `Worker` 的一生：`runWorker` 与 `getTask`

### `runWorker`：一次任务的"完整生命周期"

每个 `Worker` 启动后进入 `runWorker`，核心是一个 `while` 循环——**一个线程反复从队列取任务执行，而不是干完一个就死**：

```java
final void runWorker(Worker w) {
    Runnable task = w.firstTask;   // 先干"第一个任务"（可能是 null）
    w.firstTask = null;
    boolean completedAbruptly = true;   // 是否"异常退出"
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();                    // AQS 锁：标记"我在执行任务"
            try {
                beforeExecute(w.thread, task);   // 钩子①：执行前
                task.run();                      // 真正跑业务
                afterExecute(task, null);        // 钩子②：执行后
            } finally {
                task = null;             // 释放引用，防止长任务滞留
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly); // 退出：回收 + 决定是否补位
    }
}
```

两个被许多开发者忽略的细节，都在这里：

- **`beforeExecute` / `afterExecute` 是模板方法，可以重写**——这就是"统一捕获 `submit` 异常"的官方入口（第六节展开）。
- **`afterExecute` 一旦抛异常，任务会被记为"异常退出"**（`completedAbruptly = true`），当前 worker 直接走 `processWorkerExit`。这意味着：**别在 `afterExecute` 里干重活**，一个异常就能让整个 worker 倒下。

### `getTask`：线程为什么"有生有死"

`getTask()` 从队列取任务，但它同时负责**线程的回收决策**——这正是 `keepAliveTime` 和 `allowCoreThreadTimeOut` 的源头：

```java
private Runnable getTask() {
    boolean timedOut = false;
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 池已关停（SHUTDOWN 且队列空 / STOP 及以上）→ 线程退出
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;               // 返回 null → runWorker 循环退出 → 线程结束
        }

        int wc = workerCountOf(c);
        // ★ 判断"当前线程是不是需要被超时回收的临时线程"
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        // 超时了 且（线程不止一个 或 队列已空）→ 回收这个线程
        if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            // ★ timed → poll(keepAliveTime)：等一会儿，等不到就超时
            //   非 timed → take()：无限等，永远不超时（核心线程）
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null) return r;
            timedOut = true;           // poll 超时 → 记一笔，下一轮就退出
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

**读到这里，两条规则彻底显形：**

| 源码事实 | 对应规则 |
|---|---|
| `timed = allowCoreThreadTimeOut \|\| wc > corePoolSize` | `keepAliveTime` **只对非核心线程生效**；想让核心线程也超时，必须显式 `allowCoreThreadTimeOut(true)` |
| 核心线程走 `take()`（无限等待） | 核心线程默认**永不死**——这就是"常驻"的源码含义 |
| `wc > maximumPoolSize` 也触发退出 | `setMaximumPoolSize()` 调小后，多余的线程会在这轮被回收 |

### `processWorkerExit`：线程死了，池子会"补位"

一个 worker 退出时，线程池不是简单减员，而是会判断要不要**补一个新线程**：

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // ... 移除 worker、tryTerminate() ...

    if (runStateLessThan(ctl.get(), STOP)) {   // 池还活着
        if (!completedAbruptly) {
            // 正常退出：保证池里至少保留 corePoolSize 个线程
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && !workQueue.isEmpty()) min = 1;
            if (workerCountOf(ctl.get()) >= min) return;
        }
        addWorker(null, false);    // ★ 异常退出 / 不足额 → 补一个 worker
    }
}
```

> **最佳实践 × 源码**：网上说"`execute` 提交的任务抛异常，该线程销毁、线程池补一个新线程"——这个"补新线程"就发生在这里。所以异常线程**不会**导致池子慢慢变空，但会让"老线程带着异常死掉、新线程重新开始"，如果你把线程数做成"养兵"资源，这种反复死亡/补位本身就是一种损耗。

***

## 五、 拒绝策略源码：四个"几行代码"的策略

拒绝策略在 JDK 里不过是 `ThreadPoolExecutor` 的四个内部静态类，每个都短到几行——但每一行都有讲究：

```java
// 默认策略：直接抛异常
public static class AbortPolicy implements RejectedExecutionHandler {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException(
            "Task " + r.toString() + " rejected from " + e.toString());
    }
}

// 最推荐：让提交任务的线程自己跑
public static class CallerRunsPolicy implements RejectedExecutionHandler {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {   // ★ 池已关停就不跑，否则任务会无限滞留
            r.run();             // ★ 提交者线程同步执行 → 天然背压
        }
    }
}

// 静默丢弃
public static class DiscardPolicy implements RejectedExecutionHandler {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    }
}

// 丢最老，再提交新的
public static class DiscardOldestPolicy implements RejectedExecutionHandler {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            e.getQueue().poll();   // 丢队头最老任务
            e.execute(r);          // 再走一遍提交流程
        }
    }
}
```

三个源码细节值得品：

1. **`CallerRunsPolicy` 和 `DiscardOldestPolicy` 都带 `!e.isShutdown()` 守卫**——池都关了还执行任务，等于把"拒绝"变成"无限滞留"，这是有意的防御。
2. **`CallerRunsPolicy` 的背压是"隐式"的**：`r.run()` 同步执行，提交者线程被占用，自然无法继续提交新任务——不需要任何限流框架，压力自动回传。
3. **`DiscardOldestPolicy` 内部又调用了 `execute(r)`**——注意这是递归重入 `execute`，而不是直接执行。所以如果它再次触发拒绝，会再走一次拒绝策略。

> **最佳实践 × 源码**：为什么"自定义拒绝策略"时官方示例都建议**打日志 + 计数 + 告警**？因为 `AbortPolicy` 抛异常只能让"当前提交方"知道，而很多提交方（MQ 消费者、定时任务）未必会处理；把拒绝次数接进监控，才让"线程池被打满"这件事**对运维可见**。

***

## 六、 最佳实践 × 源码对照：每条红线的"所以然"

这一节把知识库前两篇文章里的核心规则，逐一和源码对上号。

### 1. 为什么 `submit` 的异常会"静默消失"

```java
// ExecutorService.submit：把任务包成 FutureTask，再交给 execute
public <T> Future<T> submit(Callable<T> task) {
    ...
    RunnableFuture<T> ftask = newTaskFor(task);  // new FutureTask<>(task)
    execute(ftask);                              // 交给 execute 的是 FutureTask
    return ftask;
}

// FutureTask.run：异常不冒泡，而是被"存"起来
public void run() {
    ...
    try {
        Callable<V> c = callable;
        ...
        result = c.call();          // 业务抛异常
        ...
    } catch (Throwable ex) {
        setException(ex);           // ★ 异常只写入 state + outcome，线程不受影响
    }
}

// 只有 get() 才把异常"挖"出来
public V get() throws InterruptedException, ExecutionException {
    ...
    return report(s);               // 若 state == EXCEPTIONAL → throw new ExecutionException(cause)
}
```

**结论浮出水面**：`submit` 让 `execute` 执行的是 `FutureTask`，而 `FutureTask.run()` 用 `try-catch` 把异常存进了内部字段——**工作线程根本不会收到这个异常**。所以：

- 你 `submit` 后不 `get()` → 异常永远躺在那儿 → 线上表现为"任务失败但日志干净"；
- 官方为此在 `ThreadPoolExecutor` 的 javadoc 里明确建议：重写 `afterExecute`，当 `t == null && r instanceof Future<?>` 时，手动 `((Future<?>) r).get()` 把异常取出来记日志。

### 2. 为什么入队用 `offer` 而不是 `put`

`execute` 里入队用的是 `workQueue.offer(command)`。`BlockingQueue` 有 `put`（满了阻塞）和 `offer`（满了返回 false）——**为什么选 `offer`？**

因为线程池**必须立刻知道"队列满了"，才能进入第三步"开非核心线程"**。如果用 `put`，线程池会在入队时无限阻塞，`maximumPoolSize` 形同虚设，扩容逻辑永远走不到。**`offer` 的"满了就返回 false"，正是触发扩容的信号。**

> 反过来说，如果你在业务里用 `workQueue.put()` 手动塞任务，那就绕过了线程池的扩容与拒绝逻辑——队列塞满就挂起，属于常见误用。

### 3. 为什么"严禁 Executors"是无界的错

| `Executors` 方法 | 源码真相 | 后果 |
|---|---|---|
| `newFixedThreadPool(n)` | `LinkedBlockingQueue` 默认容量 `Integer.MAX_VALUE` | 队列"装得下"，`execute` 永远走不到拒绝 → 内存被任务撑爆 → **OOM** |
| `newCachedThreadPool()` | `maximumPoolSize = Integer.MAX_VALUE` + `SynchronousQueue` | 来一个任务开一个线程，线程数上不封顶 → 栈内存耗尽 → **OOM** |
| `newScheduledThreadPool(n)` | `DelayedWorkQueue`（无界） | 任务堆积 → **OOM** |

对照 `execute` 的决策树看：`FixedThreadPool` 因为队列"永远不满"，**第三步（开非核心线程）和第四步（拒绝）永远不会触发**，任务全堆进队列。等内存爆掉时，堆里是几千万个 `Runnable`。

### 4. 为什么 `ThreadLocal` 必须 `remove`

源码事实：`ThreadLocal` 的值存在**线程私有**的 `ThreadLocalMap` 里，而这个 map 是线程对象字段，**跟着线程走，不跟着任务走**。

线程池里线程是复用的——`getTask()` 循环让同一个 `Worker.thread` 执行成千上万个任务。如果任务 A 往 `ThreadLocal` 写了值不清理，任务 B 复用到同一线程时，`get()` 读到的就是 A 的残留。更糟的是：线程池线程是长生命周期对象，`ThreadLocalMap` 强引用 value，**业务对象该被 GC 时还被线程拽着 → 内存泄漏**。

```java
// ✅ 铁律：try-finally 里 remove
try {
    UserContext.set(userId);
    // ... 业务
} finally {
    UserContext.remove();
}
```

> **最佳实践 × 源码**：Spring 的 `@Async` / `ThreadPoolTaskExecutor` 用 `TaskDecorator` 在提交时复制 `MDC`/上下文、执行后清理，本质就是"在 `execute` 之前和 `afterExecute` 之后各做一次 `set`/`remove`"——和你手写 `try-finally` 是同一件事，只是托管化了。

### 5. 为什么"优雅停机"是三段式

对照状态机表格，`shutdown` 和 `shutdownNow` 的区别一目了然：

| 方法 | 状态迁移 | 源码关键动作 |
|---|---|---|
| `shutdown()` | `RUNNING → SHUTDOWN` | `interruptIdleWorkers()`：对每个 worker 用 **`tryLock()`** 尝试加锁，只有**空闲**（未在执行任务）的线程被 interrupt；正在干活的线程把存量任务跑完，循环里 `getTask()` 发现 `SHUTDOWN && 队列空` 后自然退出 |
| `shutdownNow()` | `RUNNING → STOP` | `interruptWorkers()`：**不加锁**，打断所有线程；`drainQueue()` 清空队列并返回未执行任务列表 |

所以推荐的"`shutdown() → awaitTermination(超时) → shutdownNow() → awaitTermination(再等一轮)`"本质上是在**顺着状态机走**：先温和地让存量任务跑完，跑不完再强制 STOP，最后等 `TERMINATED`。

***

## 七、 生产排障实战：三个真实事故复盘

源码是"纸上谈兵"，真正检验理解的是排障。三个高频事故，按"现场 → 排查 → 根因 → 修复"复盘。

### 事故一：凌晨 OOM 挂了，白天又挂

**现场**：某报表导出服务连续两天在凌晨任务高峰 OOM 崩溃，重启恢复，白天正常。

**排查**：
```bash
# ① 看堆：什么对象撑爆了内存
jmap -histo:live <pid> | head -20
#    结果：char[] 和 java.util.concurrent.LinkedBlockingQueue$Node 占据大头

# ② 看线程：都在干什么
jstack <pid> > dump.txt
#    结果：几十个 "pool-1-thread-N" 卡在导出逻辑
```
顺着线程名发现是 `Executors.newFixedThreadPool(10)` 创建的池，队列里积压了**几十万条**待导出任务。

**根因**：`FixedThreadPool` + 无界 `LinkedBlockingQueue`。对照 `execute` 决策树：队列永远不满 → 永不扩容、永不拒绝 → 任务无限堆积直到 OOM。而**导出的核心线程全在忙**，`getPoolSize()` 恒等于 10，看似"没毛病"，实际上队列才是真正的炸弹。

**修复**：
```java
private static final ThreadPoolExecutor REPORT_POOL = new ThreadPoolExecutor(
        8, 16, 60L, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(5000),                    // 有界！
        new NamedThreadFactory("report"),                  // 命名，见事故三
        new ThreadPoolExecutor.CallerRunsPolicy()          // 背压兜底
);
```

> 复盘金句：**无界队列的 OOM，你永远不会在 OOM 前一天看到"拒绝告警"——因为它从不拒绝，它只是默默吃内存。**

### 事故二：对账结果大面积缺失，日志却一片干净

**现场**：数据对账系统某天有几千笔对账"凭空消失"，但应用日志里 grep 不到任何异常。

**排查**：
1. 监控里"对账完成数"当日骤降为 0——业务确实没跑，但没报错；
2. 翻代码，发现提交方式是 `executor.submit(() -> 对账(...))`，**返回值 `Future` 被丢弃**；
3. 手工触发一条对账，复现后 `jstack`，任务线程早已被回收。

**根因**：`submit` 把任务包成 `FutureTask`，异常被 `setException` 存起来不冒泡（第六节第 1 点）；没人 `get()`，异常就永远躺在那。**任务全挂了，日志零异常**——这是"静默失败"最阴险的地方。

**修复**：统一用 `afterExecute` 钩子兜底：
```java
@Override
protected void afterExecute(Runnable r, Throwable t) {
    super.afterExecute(r, t);
    if (t == null && r instanceof Future<?>) {   // submit 的异常在这里被吞了
        try {
            ((Future<?>) r).get();
        } catch (CancellationException ce) {
            t = ce;
        } catch (ExecutionException ee) {
            t = ee.getCause();                   // 真正的业务异常
        } catch (InterruptedException ie) {
            Thread.currentThread().interrupt();
        }
    }
    if (t != null) {
        log.error("线程池任务执行失败", t);       // 让失败"响起来"
    }
}
```

> 复盘金句：**不用 `Future` 就别 `submit`；用了 `Future` 就必须 `get()`（哪怕只为取异常）。**

### 事故三：客服系统里，用户 A 看到了用户 B 的订单

**现场**：客服后台偶发"串数据"——用户 A 的会话里出现用户 B 的订单。重放请求无法稳定复现，频率极低。

**排查**：
1. 日志里 `currentUserId` 在同一线程名下出现 A、B 交替——锁定是**线程复用**；
2. 代码：`handle()` 里 `UserContext.set(userId)` 后没有 `remove`；
3. 对照源码：`getTask()` 循环让同一个 `Worker.thread` 连续执行多个请求的任务，第一个请求的 `ThreadLocal` 残留被第二个请求读到。

**根因**：`ThreadLocal` 跟线程走、不跟任务走，线程池复用放大了残留效应（第六节第 4 点）。

**修复**：`finally { UserContext.remove(); }`；同时给线程池命名（`NamedThreadFactory`），让这类问题的线程栈**一眼可定位到业务池子**：
```java
public class NamedThreadFactory implements ThreadFactory {
    private final AtomicInteger seq = new AtomicInteger(1);
    private final String prefix;
    public NamedThreadFactory(String prefix) { this.prefix = prefix; }
    @Override public Thread newThread(Runnable r) {
        Thread t = new Thread(r, prefix + "-" + seq.getAndIncrement());
        t.setUncaughtExceptionHandler((th, e) -> log.error("线程[{}]未捕获异常", th.getName(), e));
        return t;
    }
}
```

> 复盘金句：**线程池里的 `ThreadLocal`，写进去那一刻就欠了债，`finally` 里的 `remove()` 才是还款。**

***

## 八、 排障工具箱：把源码知识变成手上家伙

| 工具 | 命令 | 用在哪 |
|---|---|---|
| `jstack` | `jstack <pid> \| grep -A 30 "report-"` | 按线程名前缀定位是哪个池子在忙/卡住 |
| `jmap` | `jmap -histo:live <pid>` | OOM 排查：看堆里是谁在膨胀（`LinkedBlockingQueue$Node`、`char[]`） |
| `jstat` | `jstat -gcutil <pid> 1s` | 观察 Full GC 频率，佐证"任务积压拖垮内存" |
| Arthas | `thread -n 3`（最忙线程） / `thread --state WAITING` | 线上不重启，直接看线程栈与状态 |
| 监控指标 | `getPoolSize` / `getActiveCount` / `getQueue().size()` / 拒绝计数 | 常态化的三条告警：**队列积压超阈值、拒绝次数>0、活跃线程持续=最大线程** |

一个定位思路供参考：**先看线程名 → 锁定池子 → 看队列积压 → 看拒绝计数 → 对照 `execute` 决策树判断卡在哪个分支**。这五步基本覆盖 80% 的线程池事故。

***

## 九、 虚拟线程来了，这套"最佳实践"还成立吗

简短回应（详细讨论见前两篇）：虚拟线程确实解决了"**线程贵**"这一半问题——`Executors.newVirtualThreadPerTaskExecutor()` 每个任务一个轻量线程，海量 I/O 等待不再是痛点。但线程池的另一半价值——**限流、排队、拒绝、监控、故障隔离**——在虚拟线程时代反而更需要"池"或等价物（`Semaphore`）来兜底。所以：

- **结论不变**：CPU 密集型、需要严格并发上限/背压的场景，`ThreadPoolExecutor` 依然是最佳工具；
- **读源码的价值不变**：虚拟线程只是换了调度实现，`execute` 里的"决策树思维"——资源有限、边界要明确——在任何并发模型下都是通用的。

***

## 结语

把"源码"和"最佳实践"对齐之后，你会得到一个可复用的检查单：

- [ ] **能讲清 `execute` 的四步决策树**，包括被很多人漏掉的 `recheck`；
- [ ] **能解释 `offer` 而非 `put`**——满队返 false 是扩容的触发信号；
- [ ] **能说出 `submit` 吞异常的源码路径**（`FutureTask.setException`），并用 `afterExecute` 兜底；
- [ ] **能画出五状态迁移图**，理解优雅停机本质是顺着状态机走；
- [ ] **能给每个 `ThreadLocal` 配一个 `finally { remove(); }`**；
- [ ] **能说出"严禁 Executors"的精确理由**：无界队列 = 拒绝分支永不触发 = OOM；
- [ ] **线上排障有自己的五步流程**：线程名 → 池子 → 队列 → 拒绝 → 决策树。

前两篇教会你"怎么用"，这一篇带你看到"为什么"。**真正的高手不是记住了所有规则，而是能从源码推导出规则——当新场景出现时，他不需要等一篇新博客，自己就能算出来。** 这就是读源码的复利。

> **延伸阅读**：[线程池炼金术：Java 线程池最佳实践避坑指南](./从前端到全栈-Java线程池最佳实践.md)｜[Java 线程池最佳实践：从「会用」到「精通」](./Java线程池最佳实践-从会用到精通.md)｜[破茧成蝶：前端工程师的Java线程全栈进阶指南](./破茧成蝶-前端工程师的java线程全栈进阶指南.md)
