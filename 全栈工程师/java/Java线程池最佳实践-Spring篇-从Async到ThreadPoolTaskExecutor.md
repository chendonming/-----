---
type: Note
---
# Java 线程池最佳实践（Spring 篇）：从 @Async 到 ThreadPoolTaskExecutor

> **一句话总结**：前面几篇讲的 `ThreadPoolExecutor` 是"裸"线程池的道与术，但真实工程里九成线程池都经由 Spring 托管——`@Async`、`ThreadPoolTaskExecutor`、`TaskDecorator`。Spring 把这些包装成"注解魔法"的同时，也埋下了代理失效、默认执行器无界、异常静默、事务不跨线程四颗雷。本文是线程池系列的"落地篇"，讲透 Spring 场景下怎么配、怎么用、怎么排障，把前几篇的每一条红线翻译成 Spring 工程里的正确姿势。

> **阅读路线**：本知识库线程池系列已完成三篇——[《从「会用」到「精通」》](./Java线程池最佳实践-从会用到精通.md)（七参数全景）、[《源码篇：从原理到排障》](./Java线程池最佳实践-源码篇-从原理到排障.md)（读 `ThreadPoolExecutor` 源码）、[《线程池炼金术》](./从前端到全栈-Java线程池最佳实践.md)（前端视角避坑）。那三篇回答"裸 JDK 线程池怎么用对"，本文回答"**进了 Spring 工程，这些最佳实践长什么样**"——适合已经会写 `ThreadPoolExecutor`、正在 Spring Boot 项目里用 `@Async` 或要接线程池的同学。

***

## 一、 为什么线程池的"最佳实践"还要单独讲一篇 Spring

前几篇的所有红线——**禁止无界队列、给线程命名、拒绝策略兜底、优雅停机、ThreadLocal 用完清理、按业务拆池**——在 Spring 工程里一条都不会消失，只是全部换了副面孔：

| 裸 JDK 的最佳实践 | 到了 Spring 变成 |
|---|---|
| 手动 `new ThreadPoolExecutor` | 声明 `ThreadPoolTaskExecutor` 的 `@Bean`（Spring Boot 还能用 yml 配） |
| 自定义 `ThreadFactory` 给线程命名 | `setThreadNamePrefix("order-async-")` |
| `submit` 的异常会被吞 | `@Async void` 方法的异常**默认只打一行日志**，更隐蔽 |
| `ThreadLocal` 用完 `remove` | `TaskDecorator` 帮你复制进、执行后清 |
| 三段式优雅停机 | `setWaitForTasksToCompleteOnShutdown(true)` 一个开关 |
| 按业务拆池 | 定义多个 `@Bean`，`@Async("orderExecutor")` 指定 |

Spring 的价值是把"基础设施"托管化了：你不再手写 `shutdown()` 三部曲，不再每次 `execute` 都记得包 try-finally。但**托管 ≠ 消失**——它只是把坑换了个位置。下面从最底层的原理讲起。

***

## 二、 地基：@EnableAsync 与"代理"

`@Async` 之所以"注解一下方法就异步"，靠的是 **AOP 动态代理**：

```
调用方 → [代理对象] → 拦截到 @Async 方法 → 把方法调用打包成任务 → 提交给 TaskExecutor → 立刻返回
                                                     └── worker 线程真正执行方法体
```

- 加 `@EnableAsync` 后，Spring 为含 `@Async` 方法的 bean 生成代理；
- 每次调用被代理拦截，方法体被封装成一个 `Callable`/`Runnable` 丢进线程池；
- **调用方线程拿到的是"任务已提交"的结果，而不是方法执行的结果。**

这个"代理"模型，是理解后面一切坑的总开关——尤其是**第三节的默认执行器**和**第七节的自调用陷阱**，根源都在"任务交给了代理，而代理交给了某个线程池"。

> **触类旁通**：这就像前端的 `debounce`/`throttle` 高阶函数——你调用的其实是一个包装过的函数，真正的执行被换到了另一个时机/另一个上下文里。Java 的代理是同一套思想，只是用在了"换线程"上。

***

## 三、 第一大坑：Spring 的"默认执行器"（无界 × 2）

`@Async` 不指定执行器时，Spring 有一套默认的寻找逻辑，而**默认值里藏着两个无界雷**。

### 雷 ①：SimpleAsyncTaskExecutor——每任务一条新线程

如果 `@Async` 找不到任何可用的 `TaskExecutor`（既没有 `AsyncConfigurer`，也没有 `Executor` bean），Spring 会**兜底**用 `SimpleAsyncTaskExecutor`。这个名字很温和，行为却很凶残：

> 它**不复用线程**，每提交一个任务就 `new` 一条线程；默认 `concurrencyLimit = -1`（无限）。

这相当于把前几篇第一条红线"**永远不要为每个任务 `new Thread()`**"原样踩了回去——线程无限创建，高并发直接打爆内存。生产环境遇到 `task-1`、`task-2`……无限编号的线程，多半就是踩了这个兜底。

### 雷 ②：Spring Boot 自动配置的 applicationTaskExecutor——无界队列

Spring Boot 更"贴心"，它自动装配了一个 `ThreadPoolTaskExecutor`（bean 名 `applicationTaskExecutor`），默认参数来自 `TaskExecutionProperties`：

| 属性 | 默认值 | 隐患 |
|---|---|---|
| `core-size` | `8` | — |
| `max-size` | `Integer.MAX_VALUE` | 形同虚设，永远到不了 |
| `queue-capacity` | `Integer.MAX_VALUE` | **无界队列 → 拒绝分支永不触发 → OOM** |
| `keep-alive` | `60s` | 因队列永不不满，`max-size` 用不上，keep-alive 也无意义 |

对照源码篇的 `execute` 决策树：队列永远"装得下"，第三步（开非核心线程）和第四步（拒绝）**永远不会执行**，任务全堆进队列吃内存——和 `Executors.newFixedThreadPool` 是同一个死法，只是换成了 Spring 包装。

> **结论：别把线程池参数交给默认值。** 无论 `SimpleAsyncTaskExecutor` 兜底还是 Spring Boot 的无界默认，都在等一个"内存替你兜底"的结局。

***

## 四、 ThreadPoolTaskExecutor：被 Spring 包装过的 ThreadPoolExecutor

先澄清一个常见误解：`ThreadPoolTaskExecutor` **不是另一套线程池**，它只是把 `ThreadPoolExecutor` 包了一层，把"构造参数"变成了"setter + 生命周期管理"。所有前几篇的原理（决策树、拒绝策略、`ctl` 状态机）在底层原样成立。

参数对照表（JDK 七参数 → Spring setter）：

| ThreadPoolExecutor 七参数 | ThreadPoolTaskExecutor 对应 |
|---|---|
| `corePoolSize` | `setCorePoolSize(n)` |
| `maximumPoolSize` | `setMaxPoolSize(n)` |
| `keepAliveTime` + `unit` | `setKeepAliveSeconds(n)` |
| `workQueue` | `setQueueCapacity(n)`（内部按容量构造有界队列） |
| `threadFactory` | `setThreadNamePrefix("prefix-")`（Spring 自带命名工厂） |
| `handler` | `setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy())` |
| —（无对应） | `setTaskDecorator(...)` / `setWaitForTasksToCompleteOnShutdown(true)` / `setAwaitTerminationSeconds(n)` |

注意 `threadFactory` 这一格：Spring 用 `setThreadNamePrefix` 一行就完成了前几篇"自定义 `NamedThreadFactory`"干的事——**命名是排障的救命稻草，在 Spring 里别省**。

### 一份可以直接抄的完整配置

```java
@Configuration
@EnableAsync
public class AsyncPoolConfig {

    /** 订单异步线程池：按业务拆池（Bulkheading），不要所有业务共用一个池 */
    @Bean("orderExecutor")
    public ThreadPoolTaskExecutor orderExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);          // 核心线程
        executor.setMaxPoolSize(16);          // 高峰临时扩容上限
        executor.setQueueCapacity(2000);      // ★ 有界队列：挡住任务无限堆积
        executor.setKeepAliveSeconds(60);     // 非核心线程空闲 60s 回收
        executor.setThreadNamePrefix("order-async-");   // ★ 命名：排障一眼定位
        // ★ 拒绝策略：调用方线程自己执行 → 天然背压，不丢任务
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        // ★ 优雅停机：Spring 容器销毁时等存量任务跑完再关
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);
        executor.setTaskDecorator(new MdcTaskDecorator());   // 上下文透传，见第九节
        return executor;
    }
}
```

> **细节**：`ThreadPoolTaskExecutor` 实现了 `InitializingBean`，作为 `@Bean` 注册后，Spring 会自动调用 `afterPropertiesSet()` → `initialize()`，**不需要手动调 `initialize()`**。同理它实现 `DisposableBean`，容器关闭时自动执行优雅停机——这就是第十节要讲的"托管版停机"。

***

## 五、 @Async 的正确用法：返回值、异常、指定线程池

### 1. 返回值：只能是 void 或 Future 系

Spring 官方文档原话：异步方法**必须**返回 `void`，或者 `Future` 类型的值（实践中用 `CompletableFuture`）。

```java
// ✅ 正确：void —— 适合"发出去就行"的通知/清理
@Async("orderExecutor")
public void sendNotification(Order order) { ... }

// ✅ 正确：CompletableFuture —— 调用方能拿到结果、做编排
@Async("orderExecutor")
public CompletableFuture<Order> createOrder(Order order) { ... }
```

**非 void 非 Future 的返回值是不被允许的**——如果你写 `public Order createOrder(...)` 还标了 `@Async`，返回值根本传不回调用方（代理提交任务后立刻返回 `null`）。这是线上"异步方法返回值永远拿不到"的高频乌龙。

### 2. void 方法的异常：默认只打一行日志

这是 Spring 版"异常静默"，比裸线程池的 `submit` 吞异常更隐蔽：

> 官方文档：void 返回的异步方法，异常**无法传回调用方**，默认行为是**仅仅记录日志**。

也就是说，`@Async void` 方法里 `throw new RuntimeException(...)`，调用方一点感觉都没有，日志里只有一行 WARN——**线上任务全挂但没人告警**，和源码篇"事故二"同一个味道。

正确做法：实现 `AsyncConfigurer`，配一个会告警的 `AsyncUncaughtExceptionHandler`：

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    /** 全局默认异步执行器（@Async 不带池名时用它） */
    @Resource(name = "orderExecutor")              // 复用第四节的 @Bean
    private ThreadPoolTaskExecutor orderExecutor;

    @Override
    public Executor getAsyncExecutor() {
        return orderExecutor;
    }

    /** ★ void 异步方法的异常统一在这里收口：记日志 + 接告警 */
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("异步方法 [{}#{}] 执行失败", method.getDeclaringClass().getSimpleName(),
                    method.getName(), ex);
            alertService.alert("async-task-failed", method.getName(), ex);   // 让失败"响起来"
        };
    }
}
```

### 3. CompletableFuture 返回：异常在 future 里，必须主动处理

和裸线程池 `submit` 后必须 `get()` 一个道理：

```java
CompletableFuture<Order> future = orderService.createOrder(order);
future.exceptionally(ex -> {
    log.error("创建订单异步任务失败", ex);
    return fallbackOrder();          // 降级兜底
});
```

### 4. @Async 指定线程池：@Async("beanName")

```java
@Async("orderExecutor")     // 指定池名 → 路由到对应 @Bean
public void doOrder() { ... }

@Async("reportExecutor")    // 报表池：线程少、队列大，独立调优
public void doReport() { ... }
```

> **判断标准**：并发特性差异大、或相互不能容忍阻塞的业务，一定要拆池（`@Async("a")` / `@Async("b")`）。多个 `@Async` 共用一个无界默认池，就是"报表把订单堵死"的前奏。

***

## 六、 异常兜底小结：三种提交方式的异常去向

| 提交方式 | 异常去哪了 | 你要做的 |
|---|---|---|
| `@Async void` | `AsyncUncaughtExceptionHandler`，**默认只打日志** | 实现 `AsyncConfigurer` 配 handler，记日志 + 告警 |
| `@Async` 返回 `CompletableFuture` | 异常存进 future | `exceptionally` / `whenComplete` / `get()` 主动取 |
| 裸 `executor.execute(runnable)` | 工作线程 + `UncaughtExceptionHandler` | 配 `setThreadNamePrefix` 时同步配 handler（Spring 默认打日志） |

> **一句话**：Spring 把"异常不漏"这件事也托管化了，但**托管≠自动**——`AsyncUncaughtExceptionHandler` 和 future 的 `exceptionally` 必须你自己写。

***

## 七、 自调用陷阱：为什么 this.xxx() 不生效

`@Async` 靠代理生效，而代理**只在从外部调用 bean 方法时才会拦截**。在类内部用 `this` 调用，绕过了代理：

```java
@Service
public class OrderService {

    public void handle(Order order) {
        this.sendNotification(order);   // ❌ this 调用 → 不走代理 → 还是同步执行！
    }

    @Async
    public void sendNotification(Order order) { ... }   // 你以为异步了，实际同步
}
```

**这是 Spring 异步/事务/缓存注解共有的经典坑**（`@Transactional`、`@Cacheable` 同样踩）。三种解法：

```java
// 解法一：注入"自身代理"（最直观）
@Service
public class OrderService {
    @Autowired @Lazy private OrderService self;   // 注入的是代理对象

    public void handle(Order order) {
        self.sendNotification(order);             // ✅ 走代理 → 生效
    }

    @Async
    public void sendNotification(Order order) { ... }
}

// 解法二：AopContext.currentProxy()（需 @EnableAspectJAutoProxy(exposeProxy = true)）
((OrderService) AopContext.currentProxy()).sendNotification(order);

// 解法三（最推荐）：拆一个独立的异步服务
@Service
public class OrderAsyncService {
    @Async("orderExecutor")
    public void sendNotification(Order order) { ... }
}
// OrderService.handle() 里注入 OrderAsyncService 调用 —— 天然走代理，无歧义
```

> **触类旁通**：这就是前端 `bind`/箭头函数丢 `this` 的那类问题——你以为调的是同一个函数，其实上下文已经换了。代理环境下，**"从外部调"才等于"带拦截地调"**。

***

## 八、 @Async 与 @Transactional：事务不跨线程

线程池系列反复强调过 **ThreadLocal 跟线程走、不跟任务走**。Spring 的事务上下文（`TransactionSynchronizationManager`）本质就是一个 ThreadLocal——所以：

> **调用方线程开启的事务，异步线程里根本看不到。**

```java
// ❌ 陷阱：事务只对调用线程生效，异步线程里的 DB 操作不在事务内
@Transactional
public void pay(Order order) {
    deductBalance(order);        // 有事务
    this.sendReceipt(order);     // @Async 异步线程里 update 单据 → 没有事务！
}

// ✅ 正确：异步方法自己在 worker 线程上开启新事务
@Async("orderExecutor")
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void sendReceipt(Order order) { ... }
```

两条原则：

1. **不要在异步方法里"蹭"调用方的事务**——它蹭不到，要做 DB 写就自己标 `@Transactional`（在 worker 线程开一个全新事务）；
2. **`@Async` 与 `@Transactional` 叠加时，两者都是代理**——建议把异步方法拆到独立 bean，避免多个代理的拦截顺序纠缠。

> **触类旁通**：前端 `async/await` 里你以为 `await` 还是同一个"执行上下文"，其实事件循环已经切走了。Java 的线程切换比这更彻底——连 ThreadLocal 都被搬走了。

***

## 九、 TaskDecorator：在 Spring 里优雅地透传上下文

前几篇的"ThreadLocal 必须 `finally` 里 `remove`"在 Spring 里有更体面的解法：**`TaskDecorator`**。它在线程池提交任务时统一包装，把调用方线程的上下文**复制进** worker 线程，执行后**清掉**——正是手写 `try-finally` 的托管版。

典型场景：日志链路 `MDC`（traceId）、用户上下文、租户上下文。

```java
/** 把 MDC 上下文从提交线程透传到 worker 线程，执行后清理 */
public class MdcTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> contextMap = MDC.getCopyOfContextMap();   // 提交线程复制一份
        return () -> {
            try {
                if (contextMap != null) {
                    MDC.setContextMap(contextMap);    // worker 线程写入
                }
                runnable.run();
            } finally {
                MDC.clear();                          // ★ 清理：防止串台/泄漏
            }
        };
    }
}
```

接上这个装饰器后，**所有**经过该线程池的任务自动具备上下文透传能力——不用在每一处业务代码里手动 `set`/`remove`。

> 补充：用 `TTL`（TransmittableThreadLocal，阿里开源）能连"线程池复用"场景下的值变更都透传，适合更复杂的链路上下文。但先搞懂 `TaskDecorator` 这一个机制，能覆盖 80% 的需求。
>
> 对照：源码篇讲过，Spring `@Async` 底层用的就是 `beforeExecute`/`afterExecute` 钩子思路，`TaskDecorator` 是把这套能力暴露给业务的标准入口。

***

## 十、 优雅停机：Spring 生命周期托管版

裸线程池要手写"`shutdown → awaitTermination → shutdownNow` 三段式"，**在 Spring 里两个开关搞定**：

```java
executor.setWaitForTasksToCompleteOnShutdown(true);   // 容器销毁时，先等存量任务跑完
executor.setAwaitTerminationSeconds(60);              // 最多等 60s，超时才强制 shutdownNow
```

`ThreadPoolTaskExecutor` 实现了 `DisposableBean`，Spring 容器关闭时会自动走一遍"等存量 → 超时强杀"。如果你用的是 Spring Boot 的自动配置执行器，也可以直接用 yml：

```yaml
spring:
  task:
    execution:
      shutdown:
        await-termination: true            # 发布/停机时等任务跑完
        await-termination-period: 60s      # 超时强杀
```

**发布时的连带坑**：如果应用正在"停止接收新任务"（`shutdown` 已调用），此时还有请求往线程池提交任务，会抛 `TaskRejectedException`（Spring 对拒绝的包装）。所以：

1. 停机要放在**所有提交方之后**——Spring 的 `DisposableBean` 关闭顺序天然靠后，符合预期；
2. 但**网关/入口要先把流量切走**，再等容器优雅关闭，避免"边关边提交、提交即被拒"。

> **触类旁通**：这就是前端 `beforeunload` 里"先保存草稿、再关页面"的顺序问题。优雅下线的本质是——**先停止新的，再消化存量的，最后才动刀**。

***

## 十一、 CompletableFuture + 自定义线程池：别再用公共池

线程池系列的"细节"一节提过：`CompletableFuture.supplyAsync(...)` 不带池时默认走 JVM 全局共享的 `ForkJoinPool.commonPool()`，全应用不可控。在 Spring 工程里，同样的红线换成"**用你的 `ThreadPoolTaskExecutor` 显式传池**"：

```java
@Autowired
private ThreadPoolTaskExecutor orderExecutor;   // 或者注入一个 Executor 类型

CompletableFuture
    .supplyAsync(() -> queryUserInfo(userId), orderExecutor)          // 显式传池
    .thenApplyAsync(UserInfo::maskPhone, orderExecutor)               // 后续编排也传同一个池
    .exceptionally(ex -> { log.error("查询用户信息失败", ex); return null; });
```

而 `@Async` 返回 `CompletableFuture`，本质上就是 Spring 帮你做了"`supplyAsync(..., 指定执行器)`"这件事——两者是同一套东西，选其一即可，别混着用导致线程池分散。

> **红线重申**：任何异步编排，**默认池就是失控池**。要么 `@Async("池名")`，要么 `supplyAsync(..., 池)`，二选一，但绝不能用 `commonPool` 裸奔。

***

## 十二、 检查清单：Spring 线程池自检

写/审 Spring 异步代码前，对照一遍：

- [ ] **从不依赖默认执行器**：`SimpleAsyncTaskExecutor` 兜底和 Spring Boot 的无界 `applicationTaskExecutor` 都是雷，显式声明 `ThreadPoolTaskExecutor` `@Bean`；
- [ ] **有界队列**：`setQueueCapacity` 必须给一个有限值，别留 `Integer.MAX_VALUE`；
- [ ] **给线程命名**：`setThreadNamePrefix("业务-")`，排障一眼定位是哪个池子；
- [ ] **明确拒绝策略**：核心链路用 `CallerRunsPolicy`（背压不丢）或自定义告警策略；
- [ ] **`@Async` 返回值只有 void / `CompletableFuture`**，不写"返回业务对象还标 `@Async`"；
- [ ] **void 异步方法配了 `AsyncUncaughtExceptionHandler`**（记日志 + 告警），不是默认的打日志；
- [ ] **`CompletableFuture` 返回值主动 `exceptionally`/`get()`**，不让异常躺在 future 里；
- [ ] **没有 `this.xxx()` 自调用**：跨 bean 调用或注入自身代理，保证代理生效；
- [ ] **异步里的 DB 写**：异步方法自己标 `@Transactional`，不指望调用方事务跨线程；
- [ ] **上下文透传用 `TaskDecorator`**（MDC/用户上下文复制进 + 执行后清理），不手写裸 `ThreadLocal`；
- [ ] **优雅停机打开**：`setWaitForTasksToCompleteOnShutdown(true)` + `setAwaitTerminationSeconds`，发布先切流量再关容器；
- [ ] **`CompletableFuture` 显式传池**，不用 `commonPool` 裸奔。

***

## 结语

把前三篇的"道"和这一篇的"术"拼在一起，你会得到完整的画面：**裸 JDK 线程池教你理解资源边界，Spring 帮你把边界变成配置。** 这两者不是二选一——恰恰相反，只有先理解 `ThreadPoolExecutor` 的决策树、拒绝策略、`ThreadLocal` 的隐患，你才看得懂 Spring 那几个 setter 和注解背后的取舍。

Spring 的托管没有消灭任何一条红线，只是把"你手写"换成了"你配置"。**从 `new ThreadPoolExecutor` 到 `@Async`，本质是从"自己踩坑"到"配置化避坑"的工程化升级**——而这正是全栈工程师从"会写代码"走向"能上生产"的一步。

> **延伸阅读**：[从「会用」到「精通」](./Java线程池最佳实践-从会用到精通.md)｜[源码篇：从原理到排障](./Java线程池最佳实践-源码篇-从原理到排障.md)｜[线程池炼金术](./从前端到全栈-Java线程池最佳实践.md)
