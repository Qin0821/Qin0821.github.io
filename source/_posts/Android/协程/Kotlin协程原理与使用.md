---
title: Kotlin 协程：原理 / 使用场景 / 常用 API / 常见坑
date: 2026-10-21 00:26:07
---

# Kotlin 协程：原理 / 使用场景 / 常用 API / 常见坑

> 一句话定位：协程是"用同步的写法写异步代码"的工具——把回调地狱拍平、把线程切换藏进调度器，用**编译器生成的状态机**实现"挂起时不阻塞线程"，再用**结构化并发**保证不会有协程被漏掉。
> 版本提示：本文以 kotlinx.coroutines 1.6+ 的 API 为准，个别 API（如 `limitedParallelism`）在更老的版本上不存在，以你项目的实际版本为准。

---

## 一、协程是什么（原理）

### 1.1 为什么需要协程

异步代码在协程之前有两个老大难：

**① 回调地狱（Callback Hell）**：多个异步操作要串行/嵌套时，代码会往右无限缩进，而且**错误处理、取消、超时**都要在每一层分别写一遍。

```kotlin
// 回调写法：三步串行，缩进越来越深，还不好取消
api.getToken { token, e1 ->
    if (e1 != null) return@getToken onError(e1)
    api.getUser(token) { user, e2 ->
        if (e2 != null) return@getUser onError(e2)
        api.getOrders(user.id) { orders, e3 ->
            if (e3 != null) return@getOrders onError(e3)
            render(orders)                     // 真正的业务逻辑被埋在最里面
        }
    }
}
```

协程把它拍平：

```kotlin
// 协程写法：线性、所见即所得，异常/取消/超时天然可管
val token = api.getToken()          // suspend 函数，看起来像同步
val user = api.getUser(token)
val orders = api.getOrders(user.id)
render(orders)
```

**② 线程切换成本高**：`Thread` 由操作系统调度，创建/切换都要陷入内核态，而且一个线程至少占 1MB 左右的栈空间。想要"成百上千个并发任务"时，线程模型直接撑不住，协程就是来解决这个的。

### 1.2 挂起函数为什么"不阻塞线程"

这是协程最核心的一问。三个关键词：**CPS 变换、状态机、Continuation**。

**第一步：CPS（Continuation-Passing Style）变换**

`suspend` 函数编译后，签名会多出一个 `Continuation` 参数，返回值类型变成 `Any?`（因为既可能是真实结果，也可能是"挂起"这个特殊标记）：

```kotlin
// 源码
suspend fun loadUser(id: String): User

// 编译后大致等价于（简化）
fun loadUser(id: String, cont: Continuation<User>): Any?
```

`Continuation` 就是"回调"的抽象，它只有一个 `resumeWith(result: Result<T>)` 方法——**挂起点之后的代码，被包装进了这个 continuation 里**。所以"挂起"的本质是把"剩下的活儿"存进一个对象，等结果回来再调用它继续跑。

**第二步：状态机**

编译器会把一个 `suspend` 函数的函数体改造成一个**状态机**，用 `label` 记录"我执行到第几个挂起点了"。恢复时从上次的 label 处继续，而不是从头再来。

```kotlin
suspend fun loadUser(id: String): User {
    val token = fetchToken(id)     // 挂起点 A
    val user = fetchUser(token)    // 挂起点 B
    return user
}
```

```kotlin
// 编译后的大致形态（伪代码，仅示意）
fun loadUser(id: String, cont: Continuation<User>): Any? {
    // 首次进入时创建一个状态机对象（它本身就是一个 Continuation）
    val sm = cont as? LoadUserSM ?: LoadUserSM(cont)

    when (sm.label) {
        0 -> {
            sm.label = 1                                  // 记录"下次从 1 恢复"
            val r = fetchToken(id, sm)                    // 把状态机自己当 continuation 传下去
            if (r === COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED  // 真挂起了，直接返回
            // 没挂起（结果同步就绪），往下走
        }
        1 -> { /* 恢复点 A：从 sm.result 取 fetchToken 的结果 */ }
        else -> throw IllegalStateException("call to 'resume' before 'invoke'")
    }
    sm.token = /* 上一步结果 */
    sm.label = 2
    val r2 = fetchUser(sm.token, sm)
    if (r2 === COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
    return r2 as User                                 // 全部跑完，直接返回结果
}
```

`loadUser` 的局部变量（id、token）都被搬进了状态机对象的字段，所以**挂起时线程可以走掉，恢复时状态还在**。

**第三步：线程去哪了**

```
调用 loadUser()
   ↓
执行到 fetchToken()，发网络请求
   ↓ 返回 COROUTINE_SUSPENDED
loadUser 立刻 return，把"剩下的代码(状态机+label)"存在堆上
   ↓
调用它的线程此刻就自由了 —— 可以去跑别的协程、或者去做别的事
   ↓（IO 线程收到响应）
网络回调触发 continuation.resumeWith(result)
   ↓ 根据 label 切换到恢复点 A，用对应的 Dispatcher 把代码派发回目标线程
从 A 继续执行，接着 fetchUser()
```

**结论：挂起不是"暂停线程"，而是"释放线程 + 把后续代码存起来"。** 线程被归还给调度器去干别的活，所以一个线程能同时"服务"成千上万个处于挂起状态的协程。真正阻塞线程的是 `Thread.sleep`、`Object.wait`、同步 IO 这类**阻塞式调用**，它们不会让线程归还。

### 1.3 协程和线程的关系

一句话：**协程是"用户态的、由库调度的轻量任务"，线程是"内核态的、由操作系统调度的执行单元"。协程跑在线程上，随时可以换线程。**

| 维度 | 线程 | 协程 |
| --- | --- | --- |
| 调度者 | 操作系统内核 | 用户态的库（`CoroutineDispatcher`） |
| 切换成本 | 内核态切换，微秒级 | 用户态对象跳转，纳秒~微秒级 |
| 资源占用 | 每个线程独立栈（约 1MB 量级） | 一个状态机对象，堆上分配，几十~几百字节量级 |
| 数量级 | 几百~几千就吃紧 | 单线程可跑成千上万个 |
| 阻塞行为 | `Thread.sleep` 直接占住线程 | `delay` 只是登记一个定时任务，线程照跑 |

注意两个常见的说法纠正：

- **协程不是"比线程更轻的线程"**，它是一个编译器+库协作的**任务抽象**，最终还是靠线程池执行；
- **协程不会自动并行**。没有 `Dispatchers` 指定多线程时，它可能一直跑在同一个线程上（比如 `Dispatchers.Main` 上的协程都是串行切换的）。

### 1.4 CoroutineContext：四个元素

`CoroutineContext` 是一个"类似 Map 的不可变集合"，按 `Key` 存元素，索引运算符 `[]` 取值。面试常问的四个：

| 元素 | Key | 作用 |
| --- | --- | --- |
| `Job` | `Job` | 协程的生命周期句柄，管状态、取消、父子关系 |
| `CoroutineDispatcher` | `ContinuationInterceptor` | 决定协程在哪个线程/线程池执行 |
| `CoroutineName` | `CoroutineName` | 协程名字（调试、日志定位用） |
| `CoroutineExceptionHandler` | `CoroutineExceptionHandler` | 兜底处理"没人接住"的异常 |

组合方式是 `+`，后者覆盖同 Key 的前者：

```kotlin
val ctx = Dispatchers.IO + CoroutineName("feed-loader") + Job() + handler
```

**注意**：`CoroutineDispatcher` 在 Context 里对应的 Key 是 `ContinuationInterceptor`，所以它和拦截器是同一个位置的东西——这也是"为什么换调度器就是换 Context"的原因。

### 1.5 结构化并发（Structured Concurrency）

**核心规则三条：**

1. **父协程要等所有子协程结束**才结束（`CoroutineScope` 里的 `launch` 都结束后，外层才算完成）；
2. **取消会向下传播**（父取消，所有子一起取消）；
3. **异常会向上传播**（子协程出错，默认会取消父协程，进而取消父的其他子协程）。

```
       父 Job
      /   |   \
   子1   子2   子3
           ✗ 抛异常
   → 通知父 Job → 父取消 → 子1、子3 一起被取消（除非有 SupervisorJob）
```

这就是"结构化"的含义：协程有清晰的父子树，**不会有"跑飞了没人管"的协程**。反例是 `GlobalScope.launch`，它没有父 Job，出错了没人管、页面销毁了它还在跑，所以基本不该用。

### 1.6 调度器：Main / IO / Default / Unconfined

| 调度器 | 干什么的 | 线程池 | 特点 |
| --- | --- | --- | --- |
| `Dispatchers.Main` | 主线程（UI） | 单线程（Android 主线程） | 只能在主线程干活，别放耗时操作 |
| `Dispatchers.IO` | 阻塞式 IO：网络、文件、数据库 | 弹性线程池，**默认并行上限为 max(64, CPU 核数)** | 线程数可按需扩，适合"会被阻塞"的任务 |
| `Dispatchers.Default` | CPU 密集型：排序、解析、计算 | **并行度 = max(2, CPU 核数)** | 线程数固定，接近核数 |
| `Dispatchers.Unconfined` | 不切换线程 | 无 | 在调用者线程立即执行，遇到挂起点后由"恢复它的线程"继续，**不推荐业务里用** |

关键区别一句话：**IO 是"会被阻塞所以多给点线程"，Default 是"CPU 就那么几个核，给多了反而互相抢占"，Unconfined 是"我不关心在哪执行"。**

注意 `Dispatchers.Default` 和 `Dispatchers.IO` **共享同一个线程池底层**（只是限制不同），所以不要指望它们是两个独立的机器。想给自定义调度器加并行上限，用 `dispatcher.limitedParallelism(n)`。

## 二、使用场景

| 场景 | 为什么用它 | 关键点 |
| --- | --- | --- |
| 网络请求（Retrofit） | suspend 函数天然线性书写，异常可 try/catch | Retrofit 直接支持 `suspend` 接口；不要自己再包线程池 |
| 数据库读写（Room） | Room 支持 suspend / Flow，不阻塞主线程 | 主线程不能读写，放 `Dispatchers.IO` 或让 Room 的 suspend 自己处理 |
| 并发聚合多个接口 | `coroutineScope { async{}…}` 并行发请求，等全部返回 | 用 `coroutineScope` 包住，任一失败整体失败并取消其余请求 |
| 超时控制 | `withTimeout` 一行搞定，不用自己起定时器 | 抛 `TimeoutCancellationException`，可用 `withTimeoutOrNull` 返回 null |
| 防抖节流 | 用协程 `delay` + `Job` 取消上一次 | 搜索框、按钮点击防连点 |
| 后台任务 / 定时轮询 | `viewModelScope.launch { while(isActive){…delay()} }` | 随作用域取消，`delay` 会让出线程 |
| 组合多个 `suspend` 调用成流水线 | 线性代码 + 结构化并发保证生命周期 | 线程切换交给 `withContext` |

**不适合用协程的场景**：

- **CPU 密集且不涉及挂起**（如一次性大数组排序）：协程不会加速计算，直接算或用 `Default` 即可，别为了"线程切换"而切换；
- **需要真正并行的阻塞式调用**：协程本身不是多线程，阻塞调用要显式 `withContext(Dispatchers.IO)`；
- **极简的单次异步、且在纯 Java 环境**：协程的价值在组合与取消，单点任务用回调更简单；
- **单纯的跨进程 / 系统级并行编程**：那是线程、进程、`WorkManager` 的活，协程管不了进程存活。

## 三、常用 API 与使用方法

### 3.1 启动协程的几种方式

```kotlin
// launch：不关心结果，"发射后不管"，返回 Job
val job: Job = scope.launch {
    doSomething()
}

// async：需要结果，返回 Deferred<T>，用 await() 拿结果
val deferred: Deferred<User> = scope.async { fetchUser() }
val user = deferred.await()

// runBlocking：阻塞当前线程直到协程结束 —— 只用于测试或 main 函数，绝不能在主线程用
runBlocking { delay(100); println("done") }

// withContext：切换上下文后执行，返回结果，是"挂起函数"不是新协程
val data = withContext(Dispatchers.IO) { readFile() }   // 切到 IO，做完自动切回来

// coroutineScope：开一个子作用域，等里面所有子协程结束才返回
coroutineScope {
    val a = async { apiA() }
    val b = async { apiB() }
    Result(a.await(), b.await())   // 两个并发跑，都完成才返回
}
```

| API | 是否新起协程 | 返回 | 是否阻塞 | 典型用途 |
| --- | --- | --- | --- | --- |
| `launch` | 是 | `Job` | 否 | 并发执行、不需要结果 |
| `async` | 是 | `Deferred<T>` | 否（`await` 时挂起） | 需要结果的并发 |
| `runBlocking` | 是 | `T` | **阻塞** | 测试、`main` |
| `withContext` | 否（同一 Job） | `T` | 挂起 | 切线程执行一段逻辑 |
| `coroutineScope` | 闭合子作用域 | `T` | 挂起 | 并发聚合、局部结构化 |
| `supervisorScope` | 闭合子作用域 | `T` | 挂起 | 子任务互不影响 |
| `withTimeout` | 否 | `T` | 挂起 | 超时控制 |
| `delay` | 否 | `Unit` | 挂起 | 非阻塞延时 |

### 3.2 异常处理

**规则一：`launch` 里的异常会直接抛给 `CoroutineExceptionHandler`（没配就崩溃），不会被 `try/catch` 之外的地方接住。**

```kotlin
// launch 的异常：自己 try 住，或交给 ExceptionHandler
scope.launch {
    try {
        riskyWork()
    } catch (e: IOException) {
        showError(e)                    // 在这里就地处理
    }
}

// 兜底全局处理（要放在顶层作用域的 Context 里，且是给"根协程"用）
val handler = CoroutineExceptionHandler { _, e ->
    log("未捕获异常: $e")               // 注意：它不能恢复协程，只能记录/上报
}
val scope = CoroutineScope(Dispatchers.Main + handler)
```

**规则二：`async` 的异常不会立刻抛，而是"存起来"，等到 `await()` 时才抛。**

```kotlin
val d = scope.async { throw IllegalStateException("boom") }  // 此刻不会崩
Thread.sleep(1000)
d.await()                                                    // 这里才抛出来
```

所以用 `async` 一定要 `await`，否则异常会被静默吞掉（这点在很多"任务默默失败了"的 bug 里能看到）。

**规则三：`SupervisorJob` / `supervisorScope` 让子协程互不牵连。**

```kotlin
// 普通 Job：一个子失败 → 父取消 → 其他子也一起被取消
scope.launch {
    launch { failA() }    // 失败
    launch { taskB() }    // 会被连带取消
}

// SupervisorJob：子协程失败不向上传播，兄弟任务照跑
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
scope.launch { failA() }  // 只是它自己失败
scope.launch { taskB() }  // 不受影响，继续执行

// supervisorScope：局部的作用域版，也需要每个子协程自己处理异常
supervisorScope {
    launch { failA() }    // 需要它自己 try/catch，否则仍会崩
    async { taskB() }     // taskB 不受 failA 影响
}
```

注意 `supervisorScope` 里的 `launch` 失败仍然会崩（因为没有父来兜底取消），只是**不影响兄弟协程**；而 `async` 在 Supervisor 下失败则要 `await` 才看到异常。

**`try/catch` 放哪里的口诀：能就地处理就包在 `launch` 的 lambda 里；要全局兜底就配 `CoroutineExceptionHandler`；`async` 必须 `await` 才能拿到异常。**

### 3.3 取消：协作式取消

**协程的取消是"协作式"的：`cancel()` 只是把 Job 标记为 cancelling，并不会强行中断线程。协程需要在挂起点（或主动检查）才会真正停下。**

```kotlin
val job = scope.launch {
    while (isActive) {          // ① 主动检查：循环里用 isActive
        heavyCompute()          //    如果这里是纯 CPU 死循环，cancel 不会生效
        yield()                 // ② 或让出：给对方取消的机会
    }
}

job.cancel()                    // 请求取消
job.cancelAndJoin()             // 请求取消并等待它真正结束（推荐）
job.join()                      // 只等结束，不取消
```

可用的检查手段：

| 手段 | 说明 |
| --- | --- |
| `isActive` | 检查当前协程是否还在活动状态 |
| `ensureActive()` | 不活动直接抛 `CancellationException` |
| `yield()` | 检查取消 + 让出线程给别的协程 |
| 挂起函数 | `delay` / `withContext` 等挂起点会自动检查取消 |

**取消后在 `finally` 里做清理，如果清理本身是挂起操作（比如要 `delay` 关闭资源、要提交日志），必须先切到 `NonCancellable`：**

```kotlin
val job = scope.launch {
    try {
        doWork()
    } finally {
        // 错误：协程已经处于 cancelling，这里任何挂起调用都会立刻抛 CancellationException
        // delay(100)

        // 正确：用 NonCancellable 让清理逻辑在"已取消"状态下仍能挂起执行
        withContext(NonCancellable) {
            saveProgress()
            delay(50)
        }
    }
}
```

**超时控制：**

```kotlin
// 超时会抛 TimeoutCancellationException
try {
    val r = withTimeout(3_000) { apiCall() }
} catch (e: TimeoutCancellationException) {
    showTimeout()
}

// 不想抛异常，超时返回 null
val r: Data? = withTimeoutOrNull(3_000) { apiCall() }
```

### 3.4 作用域：生命周期与协程绑定

| 作用域 | 生命周期 | 说明 |
| --- | --- | --- |
| `GlobalScope` | 与应用进程同生共死 | **不要用**，没有父 Job，泄漏、无法取消 |
| `lifecycleScope` | 绑定 `LifecycleOwner`（Activity/Fragment） | 组件销毁自动取消 |
| `viewModelScope` | 绑定 `ViewModel` | `onCleared()` 时取消，适合承载页面数据请求 |
| 自定义 `CoroutineScope` | 自己控制 | 要自己管理 `cancel()`，适合单例/Service |

```kotlin
// ViewModel 里：随 ViewModel 一起销毁
class FeedViewModel : ViewModel() {
    private val _state = MutableStateFlow<UiState>(UiState.Loading)
    val state: StateFlow<UiState> = _state

    fun load() {
        viewModelScope.launch {          // onCleared 时自动取消
            _state.value = UiState.Loading
            _state.value = try {
                UiState.Success(repo.fetch())
            } catch (e: Exception) {
                UiState.Error(e.message)
            }
        }
    }
}
```

```kotlin
// Activity 里：只在界面 STARTED 时收集 Flow，退到后台就自动停，避免浪费
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.state.collect { render(it) }   // 每次回到 STARTED 会重新 collect
    }
}
```

**为什么必须用 `repeatOnLifecycle`**：直接在 `lifecycleScope.launch` 里 `collect`，协程会活到 `onDestroy` 才取消，**退到后台时上游还在跑**（浪费电、浪费流量，UI 更新也没意义）。`repeatOnLifecycle(STARTED)` 会在 `onStop` 取消内部协程、`onStart` 重新启动，做到"界面可见才收集"。

### 3.5 并发控制：Mutex / Semaphore / limitedParallelism

```kotlin
// Mutex：互斥锁（协程版 synchronized），挂起式加锁，不阻塞线程
private val mutex = Mutex()

suspend fun safeIncrement() {
    mutex.withLock {                 // 同一时刻只有一个协程能进
        counter++
    }
}

// Semaphore：信号量，控制"同时最多几个"
private val semaphore = Semaphore(permits = 3)

suspend fun downloadAll(urls: List<String>) {
    coroutineScope {
        urls.forEach { url ->
            launch {
                semaphore.withPermit {   // 最多 3 个并发下载，其余等待
                    download(url)
                }
            }
        }
    }
}

// limitedParallelism：给某个调度器限制并行度
val limited = Dispatchers.IO.limitedParallelism(4)   // 最多 4 个并发
withContext(limited) { doIo() }
```

对比表：

| 工具 | 解决的问题 | 特点 |
| --- | --- | --- |
| `Mutex` | 互斥访问共享可变状态 | 挂起式加锁，不是阻塞锁，别和 `synchronized` 混用 |
| `Semaphore(n)` | 限制并发数量 | 常用来做"限流"，n=1 即互斥 |
| `limitedParallelism(n)` | 限制某个 Dispatcher 的并行度 | 更轻量，针对调度器而非业务资源 |
| `Dispatchers.IO` | 承载阻塞任务 | 线程数会随阻塞增长，别用它当"无限并发" |

## 四、常见坑与排查

| 坑 | 现象 / 原因 | 正确做法 |
| --- | --- | --- |
| **`GlobalScope` 泄漏** | 页面销毁了协程还在跑，回调用到了已销毁的 View | 用 `viewModelScope` / `lifecycleScope` |
| **`runBlocking` 阻塞主线程** | 在主线程用 `runBlocking { apiCall() }` 触发 ANR | 只用于测试/`main`，业务用 `launch` |
| **`launch` 里异常直接崩** | 根协程 `launch` 抛异常且没 Handler，进程崩溃 | 就地 `try/catch`，或配 `CoroutineExceptionHandler` |
| **`async` 异常被吞** | `async` 抛异常但从不 `await`，任务静默失败 | 一定 `await`，或用 `supervisorScope` + 逐个处理 |
| **取消传播导致外层全挂** | 一个 `async` 失败，`coroutineScope` 整体取消，其他请求白做 | 用 `supervisorScope` 隔离，或每个任务自己兜异常 |
| **`withContext` 里抛异常到哪** | `IOException` 会**原样抛给调用者**（`withContext` 不是新 Job） | 在调用处 `try/catch` 即可 |
| **`finally` 里做挂起操作被取消** | 协程已 cancelling，`finally` 里的 `delay` 立刻抛异常，清理没做完 | 清理逻辑包进 `withContext(NonCancellable)` |
| **重复启动协程** | 快速点两次按钮 → 两个请求并发，结果乱序 | 用 `Job` 取消上一个：`job?.cancel(); job = launch{}` |
| **`Dispatchers.IO` 线程爆炸** | 用 `IO` 跑海量并发任务，线程数被撑起来 | 用 `limitedParallelism` 或 `Semaphore` 限流 |
| **CPU 死循环无法取消** | 协程里纯计算循环不检查取消 | 循环里加 `isActive` / `ensureActive()` / `yield()` |
| **`suspend` 函数里做阻塞调用** | 用 `Thread.sleep`、`okio` 同步 IO 阻塞了调度线程 | 阻塞操作必须 `withContext(Dispatchers.IO)` |

**排查套路**：

1. 现象是崩溃 → 先看是不是 `launch` 没接住异常，抓 `CoroutineExceptionHandler` 里的堆栈；
2. 现象是"任务默默没结果" → 看 `async` 有没有 `await`；
3. 现象是"页面关了还在跑/内存泄漏" → 看作用域是不是 `GlobalScope` 或自定义 Scope 没 `cancel`；
4. 现象是 ANR → 看主线程是不是有 `runBlocking` 或阻塞调用；
5. 现象是"取消没生效" → 看有没有在挂起点/循环检查 `isActive`。

## 五、面试追问速答

- **`suspend` 为什么能"不阻塞"？** 因为编译器做了 CPS 变换 + 状态机：`suspend` 函数编译后多一个 `Continuation` 参数，遇到真挂起时返回 `COROUTINE_SUSPENDED` 并把后续代码存进状态机对象，调用线程立刻被释放去做别的事。它释放的是"线程"，不是"任务"——任务还活着，只是换了个时机继续。

- **挂起后在哪里恢复？** 由结果回来源头的线程触发 `continuation.resumeWith(...)`，再根据协程 Context 里的 `ContinuationInterceptor`（也就是 Dispatcher）把恢复代码派发到目标线程。所以"在 IO 挂起、恢复到 Main"是调度器决定的，不是协程本身记住了线程。

- **一个线程能跑多少个协程？** 理论上只受内存限制，成千上万甚至更多。因为挂起的协程不占线程、只占堆上的状态机对象（几十~几百字节级别），和线程的 1MB 栈不是一个量级。但**同一时刻真正"在跑"的协程数量受调度器并行度限制**。

- **结构化并发的意义是什么？** 协程有清晰的父子 Job 树：父等子、取消向下传播、异常向上传播。它保证了协程不会"跑飞"——要么被父统一管理，要么随作用域一起取消。这样才解决了回调时代"任务泄漏、异常丢失、取消遗漏"三大问题。

- **`CoroutineScope` 和 `coroutineScope` 区别？** `CoroutineScope` 是一个**接口/容器**，持有 Context 并能 `launch`；`coroutineScope` 是一个**挂起函数**，它创建一个闭合子作用域、等内部子协程全部结束才返回。前者是"我有个作用域"，后者是"我在这里开一个临时作用域并等它结束"。

- **为什么不要在 ViewModel 用 `GlobalScope`？** `GlobalScope` 没有父 Job，生命周期等于整个进程；ViewModel 销毁后它不会取消，容易持有已失效的引用造成泄漏，异常也没人兜底。应该用 `viewModelScope`，随 `onCleared` 取消，异常也可控。

- **`Dispatchers.IO` 会不会线程爆炸？** 它默认并行上限是 max(64, CPU 核数)，被阻塞时最多扩到 64 个左右，不是无限扩。但如果你用它跑成千上万的任务，仍会占满线程池、任务排队。要控制并发就用 `Semaphore` 或 `limitedParallelism`，别把 IO 当并发开关。

- **协程能替代线程吗？** 不能。协程是"任务抽象"，底层仍然跑在线程/线程池上，最终的并行能力还是由 CPU 核数和线程决定。协程替代的是"回调式异步 + 手动线程管理"这套写法，让你少写线程代码，而不是消灭线程。
