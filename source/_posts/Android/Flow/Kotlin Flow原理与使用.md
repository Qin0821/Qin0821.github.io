---
title: Kotlin Flow：原理 / 使用场景 / 常用 API / 常见坑
date: 2026-10-21 00:26:07
---

# Kotlin Flow：原理 / 使用场景 / 常用 API / 常见坑

> 一句话定位：Flow 是协程世界里的**数据流**——用 `suspend` 函数当生产者、用 `collect` 当消费者，天然支持背压和取消；`StateFlow` 管"状态"、`SharedFlow` 管"事件"，用来替代 `LiveData` 和回调式事件总线。
> 版本提示：本文以 kotlinx.coroutines 1.6+ 的 API 为准，个别 API 在更老的版本上行为不同，以你项目的实际版本为准。

---

## 一、Flow 是什么（原理）

### 1.1 Flow 是"冷流"：没人订阅就不干活

**冷流（cold stream）** 的含义是：`flow { }` 这段代码只是"定义了一段生产逻辑"，**在有人 `collect` 之前，一行都不会执行**。而且**每个订阅者都会触发一遍完整执行**。

```kotlin
val flow = flow {
    println("开始生产")          // 只有 collect 时才打印
    emit(1)
    emit(2)
}

// 这里什么都没发生
val f = flow

// 第一次订阅：打印一次"开始生产"
f.collect { println(it) }
// 第二次订阅：又打印一次"开始生产" —— 上游被重新执行
f.collect { println(it) }
```

本质：**Flow 就是一个 `suspend` 的生产者——消费者函数**。上游在 `flow { }` 里用 `emit` 发数据，下游在 `collect` 里逐条消费。它不像 RxJava 的 `Observable` 那样"先有数据源再被订阅"，而是"订阅时才开始跑生产逻辑"。

### 1.2 `collect` 是挂起函数，背压是天然带上的

```kotlin
suspend fun collect(collector: FlowCollector<T>)
```

`collect` 是挂起的，所以它必须在协程里调用。它的执行流程：

```
collect()
   ↓
进入 flow { } 的代码块
   ↓
emit(value)  →  直接调用下游的 collect 逻辑（同一条执行链，不是队列）
   ↓
下游处理完了，emit 才返回，上游才继续 emit 下一个
   ↓
下游如果很慢（比如在 saveToDb），上游下一句 emit 就"卡在那里"
```

这就是 **Flow 的背压模型**：**下游处理慢 → 上游 `emit` 挂起 → 生产者自然减速**。不需要像 RxJava 那样显式选背压策略，因为整个链条是"你处理完我才继续"的协程调用链。

### 1.3 `flowOn` 只改上游的执行上下文

这是最容易被误解的操作符。

```kotlin
flow {
    emit(fetchFromNet())      // 这里跑在 IO
}
    .map { parse(it) }        // 这里也跑在 IO（map 在 flowOn 上游）
    .flowOn(Dispatchers.IO)   // ← 从上游到 flowOn 之间全部改到 IO
    .collect { render(it) }   // collect 跑在"下游"的上下文（比如 Main）
```

**`flowOn` 影响的是"它上游"的所有代码（生产端），不影响下游。** 背后的实现：`flowOn` 会在上下游之间插入一个 `Channel`，上游把这个 Channel 当成自己的消费端、在指定 Dispatcher 上跑，下游从这个 Channel 里取数据。

```
上游（IO 线程）──emit──▶ [Channel 缓冲] ──▶ 下游（Main 线程）collect
```

所以 `flowOn` 也解决了"上游在哪个线程跑"的问题。注意：**一个 Flow 里多次 `flowOn` 只有最靠近上游的那个决定上游线程**，且 `flowOn` 只会改变生产端，下游想换线程要在 `collect` 处 `withContext`。

### 1.4 SharedFlow / StateFlow 是"热流"：先有数据，后订阅

**热流（hot stream）** 与冷流相反：数据源独立存在、一直在产出，**订阅者只是"接入"它**，只收到接入之后的数据（以及配置的重放数据）。

```kotlin
// 热流：即使没有订阅者，updateValue 也在改
val state = MutableStateFlow(0)
state.value = 1                 // 没有 collect 也生效
state.value = 2
// 现在才订阅：立刻收到"当前值 2"（重放）
state.collect { println(it) }   // 打印 2
```

**`SharedFlow` 的三个参数，决定了 `emit` 会不会挂起：**

| 参数 | 作用 | 默认值 |
| --- | --- | --- |
| `replay` | 给新订阅者**重放**最近几条数据 | `0` |
| `extraBufferCapacity` | 除 replay 外，额外可缓冲的容量 | `0`（即总缓冲 = replay） |
| `onBufferOverflow` | 缓冲满了怎么办：`SUSPEND` / `DROP_OLDEST` / `DROP_LATEST` | `SUSPEND` |

规则：

- 若 `replay + extraBufferCapacity == 0`，而 `onBufferOverflow = SUSPEND`（默认）→ **`emit` 会一直挂起**（因为没有任何缓冲，只能等订阅者来拿）；
- `replay = 0, extraBufferCapacity = 0, onBufferOverflow = DROP_OLDEST`（或 `DROP_LATEST`）→ `emit` 不挂起，直接丢；
- 想让"没有订阅者时 `emit` 也立即返回"，要么给 `extraBufferCapacity > 0` 且用 `tryEmit`，要么把溢出策略设成 `DROP_*`。

```kotlin
// 事件场景常用写法：不重放，给点缓冲，防止 emit 挂起
private val _events = MutableSharedFlow<UiEvent>(
    replay = 0,
    extraBufferCapacity = 1,
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
val events: SharedFlow<UiEvent> = _events

// 发送：tryEmit 不挂起（返回 Boolean），emit 可能挂起
_events.tryEmit(UiEvent.Toast("已保存"))
```

**`StateFlow` 本质就是 `replay = 1` + "按 `equals` 去重"的 `SharedFlow`：**

- 必须有**初始值**（`MutableStateFlow(initial)`）；
- 永远持有"当前值"，用 `.value` 同步读取；
- 新的值和旧值 `equals` 相等时**不通知**（这是"去重"，也是它和 `SharedFlow` 最本质的区别之一）；
- 因此它天生就是"状态容器"，而不是"事件通道"。

### 1.5 三种角色对比

| 维度 | `Flow`（冷流） | `StateFlow` | `SharedFlow` |
| --- | --- | --- | --- |
| 类型 | 冷流 | 热流 | 热流 |
| 是否必须有初值 | 否 | **是** | 否 |
| 是否重放 | 每个订阅者重跑一遍上游 | **重放当前值（replay=1）** | 按 `replay` 配置 |
| 去重 | 不去重 | **按 `equals` 去重** | 不去重 |
| 多订阅者 | 各自独立执行上游（可能重复请求） | 共享 | 共享（广播） |
| 读当前值 | 不支持 | `.value` 同步读 | 支持 `replayCache` 但不能当状态 |
| 典型用途 | 网络请求、DB 查询、转换管道 | UI 状态 | 一次性事件、广播 |

### 1.6 `SharedFlow` 和 `Channel` 的区别

| 维度 | `Channel` | `SharedFlow` |
| --- | --- | --- |
| 消费模型 | **点对点**：一条数据只被一个消费者拿到 | **广播**：每个订阅者都能拿到 |
| 订阅者数量 | 多个消费者会"抢"数据 | 所有订阅者各收一份 |
| 关流后 | `close` 后不能再用 | 可以一直有订阅者进出 |
| 重放 | 支持 `BUFFERED`/`CONFLATED` 等容量语义 | 支持 `replay` |
| 典型用途 | 单消费者任务队列、`callbackFlow` 内部 | 事件总线、UI 事件广播 |

**选型口诀：数据要被多个订阅者看到 → `SharedFlow`；数据只能被消费一次（点对点）→ `Channel`；要表达"状态" → `StateFlow`。**

## 二、使用场景

| 场景 | 为什么用它 | 关键点 |
| --- | --- | --- |
| UI 状态 | `StateFlow` 天然"有当前值 + 去重"，适合驱动 UI | 用 `stateIn` 从冷流转热流，`WhileSubscribed` 控制订阅生命周期 |
| 一次性事件（Toast/导航/弹窗） | 事件不能被"重放"，`SharedFlow(replay=0)` 更合适 | 别用 `StateFlow`，否则会重放老事件（粘性） |
| 搜索框输入防抖 | `debounce` + `filter` 一行搞定 | 放在 `viewModelScope`，配合 `flatMapLatest` 取消旧请求 |
| 多接口聚合 | `combine` 把多个 Flow 的最新值合并 | 任一源变化都重新发射组合结果 |
| 严格配对（A 和 B 一一对应） | `zip` 按顺序配对 | 记得处理"一边先发完"的情况 |
| Room 查询实时刷新 UI | Room 支持返回 `Flow<T>`，表变化自动重查 | DAO 直接返回 `Flow`，在 ViewModel 转 `StateFlow` |
| 下载/上传进度 | 冷流逐条 emit 进度 | 注意不要在主线程 collect 里做重活 |
| 轮询 | 冷流里 `while (isActive) { emit(fetch()); delay(x) }` | 用 `flowOn(IO)`，随作用域取消 |

**不适合用 Flow 的场景**：

- **只取一次值**：用 `suspend` 函数或 `withContext` 更直接，别为了"统一风格"包一层 Flow；
- **需要跨进程 / 全局广播**：那是 `BroadcastReceiver`、`LocalBroadcast`、消息队列的活；
- **需要"绝对不丢"的可靠消息**：Flow 的缓冲可能丢（`DROP_*`），可靠投递要靠落库重试；
- **纯同步的简单逻辑**：Flow 带来的是异步组合能力，同步逻辑套 Flow 反而绕。

## 三、常用 API 与使用方法

### 3.1 操作符分类总表

**创建**

| 操作符 | 说明 |
| --- | --- |
| `flow { emit(x) }` | 最基础，冷流，块内是挂起函数 |
| `flowOf(1, 2, 3)` | 直接给定若干值 |
| `listOf(1,2,3).asFlow()` | 集合转 Flow |
| `channelFlow { send(x) }` | 可以**并发**发射（内部用 Channel），能在多协程里 emit |
| `callbackFlow { ...; awaitClose { } }` | 把**回调式 API** 转成 Flow，必须 `awaitClose` |

**变换 / 合并**

| 操作符 | 说明 |
| --- | --- |
| `map` / `filter` | 逐条变换 / 过滤 |
| `transform` | 可以"一条进、多条出"（能多次 `emit`） |
| `flatMapLatest` | 新上游值来了，**取消上一条**的内层 Flow（搜索场景首选） |
| `flatMapMerge` | 内层 Flow **并发**执行，结果乱序返回 |
| `flatMapConcat` | 内层 Flow **顺序**执行，前后串行 |
| `combine` | 多个 Flow 的**最新值**任意一方变化就组合发射 |
| `zip` | 按顺序**一对一**配对，短的先结束就结束 |
| `merge` | 多个 Flow 并发合并，谁先有谁先出 |

**过滤 / 节流**

| 操作符 | 说明 |
| --- | --- |
| `debounce(ms)` | 静默 `ms` 后才发射最后一条（防抖） |
| `sample(ms)` | 每 `ms` 采样一次最新值（节流） |
| `distinctUntilChanged` | 值没变就不发射（默认按 `equals`） |
| `take(n)` / `takeWhile` | 取前 n 条 / 满足条件时停止 |

**异常 / 生命周期**

| 操作符 | 说明 |
| --- | --- |
| `catch { }` | 捕获**上游**异常，可以重新 `emit` 兜底值 |
| `retry(n)` / `retryWhen { }` | 重试 |
| `onCompletion { }` | 流正常结束或异常结束都会回调，适合做收尾 |
| `onStart` / `onEach` | 开始前 / 每条数据时执行副作用 |

**上下文 / 背压**

| 操作符 | 说明 |
| --- | --- |
| `flowOn(dispatcher)` | 改变**上游**执行上下文（经 Channel 中转） |
| `buffer(n)` | 给上下游之间加缓冲，提升吞吐 |
| `conflate()` | 只保留最新值（等于 `buffer(0, DROP_OLDEST)`） |
| `collectLatest { }` | 新值来了就**取消上一个**处理块（适合搜索，不适合提交） |

**终端**

| 操作符 | 说明 |
| --- | --- |
| `collect { }` | 逐条消费（挂起，触发冷流执行） |
| `collectLatest { }` | 逐条消费，新值取消上一个 |
| `toList()` / `first()` / `firstOrNull()` / `last()` | 收集为集合 / 取首条 / 取末条 |
| `reduce` / `fold` | 聚合 |
| `stateIn(scope, started, initial)` | 冷流转 `StateFlow`（热流） |
| `shareIn(scope, started, replay)` | 冷流转 `SharedFlow`（热流） |

### 3.2 Android 落地：`repeatOnLifecycle` + `stateIn`

```kotlin
// ViewModel：把冷流转成可共享的 StateFlow
class SearchViewModel(private val repo: SearchRepository) : ViewModel() {

    private val query = MutableStateFlow("")

    // 冷流：query 变化 → 防抖 → 请求 → 只保留最新
    val results: StateFlow<List<Item>> = query
        .debounce(300)                                   // 输入停止 300ms 才发
        .filter { it.isNotBlank() }
        .distinctUntilChanged()                          // 值没变不重复请求
        .flatMapLatest { q ->                            // 新查询来了，取消上一个请求
            flow { emit(repo.search(q)) }                // 网络请求在 IO
                .flowOn(Dispatchers.IO)
                .catch { emit(emptyList()) }             // 捕获上游异常，给兜底值
        }
        .stateIn(
            scope = viewModelScope,                      // 在 ViewModel 作用域内共享
            started = SharingStarted.WhileSubscribed(5_000), // 有订阅才启动，5s 无订阅才停
            initialValue = emptyList()
        )

    fun onQueryChanged(text: String) { query.value = text }
}
```

```kotlin
// Activity：界面可见才收集，退后台自动停，避免浪费
class SearchActivity : AppCompatActivity() {
    private val vm: SearchViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {   // onStart 收集，onStop 取消
                vm.results.collect { list ->
                    renderList(list)                        // 这里已经在主线程（Main）
                }
            }
        }

        editText.doAfterTextChanged { vm.onQueryChanged(it.toString()) }
    }
}
```

### 3.3 一次性事件：`SharedFlow` 的配置

```kotlin
// 事件用 SharedFlow：不重放、有缓冲，避免 emit 挂起
class LoginViewModel : ViewModel() {
    private val _events = MutableSharedFlow<Event>(
        replay = 0,                                   // 不重放：事件不能被"补看"
        extraBufferCapacity = 1,                      // 给点缓冲，没订阅者时也不挂起
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val events: SharedFlow<Event> = _events

    fun login(pwd: String) {
        viewModelScope.launch {
            if (pwd.isEmpty()) {
                _events.emit(Event.ShowToast("密码不能为空"))   // 有缓冲，不会挂
                return@launch
            }
            _events.emit(Event.NavigateHome())
        }
    }

    sealed interface Event {
        data class ShowToast(val msg: String) : Event
        object NavigateHome : Event
    }
}
```

```kotlin
// Activity 侧：repeatOnLifecycle 里收集事件，用 launch 避免阻塞收集
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        vm.events.collect { event ->
            when (event) {
                is LoginViewModel.Event.ShowToast -> toast(event.msg)
                LoginViewModel.Event.NavigateHome -> navigateHome()
            }
        }
    }
}
```

### 3.4 把回调 API 转成 Flow：`callbackFlow`

```kotlin
// 例：把位置监听 / 传感器 / 老式回调接口包装成 Flow
fun locationFlow(locationManager: LocationManager): Flow<Location> = callbackFlow {
    val listener = object : LocationListener {
        override fun onLocationChanged(location: Location) {
            trySend(location)                 // 非挂起发送
        }
        override fun onProviderDisabled(provider: String) {
            close(IllegalStateException("定位已关闭"))  // 用 close 结束流
        }
    }

    locationManager.requestLocationUpdates(GPS_PROVIDER, 0L, 0f, listener)

    awaitClose {                               // 必须：收集者取消时解除注册，否则泄漏
        locationManager.removeUpdates(listener)
    }
}

// 使用
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        locationFlow(lm).collect { loc -> updateMap(loc) }
    }
}
```

### 3.5 冷流共享：`shareIn` 避免重复请求

```kotlin
// 冷流每个订阅者都会重跑上游 —— 两个 collect 就是两次请求
val cold = flow { emit(api.fetch()) }

// 用 shareIn 变成热流：所有订阅者共享同一次执行
val shared: SharedFlow<Data> = cold.shareIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(5_000),
    replay = 1
)
```

## 四、常见坑与排查

| 坑 | 现象 / 原因 | 正确做法 |
| --- | --- | --- |
| **`StateFlow` 重放导致"粘性事件"** | 打开页面就收到老消息/老 Toast | 事件别用 `StateFlow`，用 `SharedFlow(replay=0)` |
| **`SharedFlow` 不配缓冲时 `emit` 挂起** | 没有订阅者时 `emit` 永久挂住 | 配 `extraBufferCapacity` 或 `onBufferOverflow`，或改用 `tryEmit` |
| **`flowOn` 只影响上游** | 以为 `flowOn(IO)` 后 `collect` 也在 IO | 下游线程由 `collect` 所在的上下文决定 |
| **`collectLatest` 会取消上一个任务** | 用它做"提交订单"，用户连点导致前面的提交被取消 | 搜索/展示用 `collectLatest`，提交/写库用 `collect` |
| **主线程 `collect` 里做耗时操作** | UI 卡顿 / ANR | 耗时工作放上游 + `flowOn(IO)`，`collect` 只做 UI |
| **把 Flow 当普通 List 用** | 直接 `for` 遍历、或想"取一次结果"却用了冷流 | 要一次结果用 `first()` / `toList()`，或干脆用 `suspend` |
| **多订阅者导致冷流重复执行** | 两个 `collect` 触发两次网络请求 | 用 `shareIn` / `stateIn` 共享 |
| **以为结流就脱离了协程** | Flow 必须跑在协程里，`collect` 是挂起函数 | Flow 不是"脱离协程的响应式"，它本身就是协程 | 
| **`catch` 只捕上游异常** | 下游 `collect` 里抛异常，`catch` 接不到 | `catch` 放中间捕上游；下游异常在 `collect` 处 `try/catch` |
| **操作符顺序放错** | `debounce` 放在 `flowOn` 后、`map` 前，行为不符合预期 | 记住顺序即执行顺序：`flow{}.map{}.debounce{}.flowOn{}.collect{}` 从上到下 |
| **`onEach` 顺序问题** | `onEach` 放在 `catch` 之后，捕获不到上游异常 | `onEach` / `catch` 的位置决定它能看到哪一段的异常 |

**排查套路**：

1. 现象是"打开页面收到旧数据" → 检查是不是把事件塞进了 `StateFlow`；
2. 现象是"`emit` 卡住不返回" → 检查 `SharedFlow` 的缓冲与溢出策略；
3. 现象是"请求发了两次" → 检查冷流被多个订阅者 `collect`，改用 `shareIn`；
4. 现象是"异常没被兜住" → 检查 `catch` 是不是放在了异常点的下游；
5. 现象是"上游还在跑/退后台还耗电" → 检查有没有用 `repeatOnLifecycle` + `WhileSubscribed`；
6. 现象是"提交操作莫名被取消" → 检查是不是用了 `collectLatest`。

## 五、面试追问速答

- **Flow 和 RxJava 的区别？** 核心是 Flow 建在协程和 `suspend` 之上，天然有结构化并发和取消传播；操作符更少但更直观，背压是"下游慢上游就挂起"的隐式模型，不像 RxJava 需要显式选 `BackpressureStrategy`。RxJava 线程调度操作符更丰富、生态更老，但 Flow 与协程/生命周期配合更好，新项目一般选 Flow。

- **`StateFlow` 和 `LiveData` 的区别？** `StateFlow` 是纯 Kotlin、必须给初值、有当前值可同步读、按 `equals` 去重、天然支持协程和 `switchMap` 式的 `flatMapLatest`；`LiveData` 是 Android 专属、生命周期感知是内置的、但只能在主线程 `setValue`、不支持复杂操作符。`StateFlow` + `repeatOnLifecycle` 更能替代 `LiveData` 在 MVVM 里的角色。

- **`StateFlow` 和 `SharedFlow` 怎么选？** 表达"当前状态"用 `StateFlow`（有初值、重放 1 条、去重）；表达"一次性事件"用 `SharedFlow(replay=0)`（不重放、可多条、不去重）。千万别用 `SharedFlow` 存状态（拿不到当前值、容易乱），也别用 `StateFlow` 发事件（会重放粘性事件）。

- **Flow 为什么是冷的，`SharedFlow` 为什么是热的？** 冷流只是"生产的定义"，每次订阅都重新执行上游，所以能做到"按需生产、用完即停"；热流自身持有数据源和缓冲，订阅者只是"接入"，所以数据不因订阅与否而存在与否。`StateFlow`/`SharedFlow` 内部靠 `SharedFlowImpl` 维护订阅者列表和重放缓存实现广播。

- **Flow 里怎么写"取消后发最后一条通知"？** 用 `onCompletion`；如果想在 `finally` 里做挂起操作，记得包 `withContext(NonCancellable)`。比如 `flow{...}.onCompletion { emit(FinalState()) }`，或 `try { collect } finally { withContext(NonCancellable){ cleanup() } }`，保证取消也能收尾。

- **怎么实现"两个接口都要成功才更新 UI"（combine vs zip）？** 要"两个都成功"应该各起一个 `async` 然后 `await` 两个结果，或用 `zip`（严格配对，任一出错整体失败）；`combine` 是"任意一方有最新值就发射"，可能在另一方还没返回时就用旧值组合，语义不同——要"都成功"用 `zip`/`await`，要"实时看两边最新值"用 `combine`。

- **Flow 的并发（`flatMapMerge`）与顺序（`flatMapConcat`）？** `flatMapConcat` 是串行：前一个内层 Flow 完全结束后才收集下一个，结果有序；`flatMapMerge` 是并发：所有内层 Flow 同时收集，结果按完成顺序乱序发射，适合"批量并发拉取"；`flatMapLatest` 只保留最新，来新值就取消上一个，适合搜索防抖。另外 `channelFlow` 内部也可以用 `launch` 并发 `send`。
