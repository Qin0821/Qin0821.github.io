---
title: Koin：原理 / 使用场景 / 常用 API / 常见坑
date: 2026-10-21 00:25:29
---

# Koin：原理 / 使用场景 / 常用 API / 常见坑

> 一句话定位：Koin 是 Kotlin 生态里的一个**轻量依赖注入框架**，本质是"运行时服务定位器 + DSL"——用纯 Kotlin DSL 声明依赖，用 `get()` / `by inject()` 在运行时按需解析，**不生成代码、不用注解处理器、不用反射构造**。
> 提示：本文以 Koin 3.x 的 API 习惯为准（如 `koinViewModel`、`checkModules`），不标注具体小版本。

---

## 一、Koin 是什么（原理）

**一句话：Koin 没有魔法，它就是一个"运行时容器 + 惰性工厂"。你在 `module {}` 里注册的是"怎么创建对象"的规则，`get()` 的时候才真的去执行它。**

### 1.1 核心链路

```
module { single { ... } }        声明 BeanDefinition（只是"配方"，不是实例）
        ↓
startKoin { modules(...) }       把所有 Definition 装进全局 Container
        ↓
get<T>() / by inject() / koinViewModel()
        ↓
Container 按类型（可选 qualifier）查找 Definition → 执行 lambda 创建实例
        （single 会缓存复用，factory 每次新建）
        ↓
构造参数里的 get() 递归解析依赖图（默认惰性，用到才建）
```

### 1.2 四个必须记住的点

**① 它是 Service Locator，不是纯正的 DI**

依赖注入（DI）的核心思想是"对象不自己创建依赖，由外部注入"。Koin 的构造函数注入做到了这一点（`class Foo(val bar: Bar)` + `factory { Foo(get()) }`），这部分是标准 DI。但当你在类里写 `private val bar by inject()` 时，你是在**主动向容器索要**依赖，这就是 Service Locator 的用法。所以准确说法是：**Koin 是一个以 Service Locator 为解析机制的 DI 框架**。

**② 三个核心概念**

| 概念 | 含义 |
| --- | --- |
| `module` | 一组依赖定义的集合，可拆可合 |
| `BeanDefinition` | 一条"如何创建某类型"的规则（single/factory/scoped/viewModel...） |
| `Container` | 保存所有 Definition、负责解析和缓存实例的容器 |

**③ "配置即代码"：Koin 3 用 lambda 工厂**

```kotlin
single { ApiService(get(), get()) }     // 这行只是登记了一个 lambda，并没有真的 new
```

Koin 3 之后不再依赖反射工厂，全部用 lambda 描述创建过程。这样 KMP（Kotlin Multiplatform）上的非 JVM 平台也能用（因为反射在 JS/Native 上受限）。

**④ 没有编译期校验——这是它和 Hilt/Dagger 的根本差异**

Koin 的依赖图在**运行时**才解析，所以"某个类型没注册"这种错误只会在运行时抛 `NoDefinitionFoundException`。Hilt/Dagger 用**注解处理器在编译期**生成代码并校验依赖图，缺依赖直接编译失败。这就是"用不上编译期保障，但换来极低的学习成本和构建开销"的取舍。

### 1.3 Koin vs Hilt / Dagger

| 维度 | Koin | Hilt / Dagger |
| --- | --- | --- |
| 实现机制 | 运行时服务定位器 + DSL | 编译期注解处理 + 代码生成 |
| 学习成本 | 低，会写 lambda 就行 | 高，注解体系多（Module/Provides/Inject/Scope...） |
| 编译期校验 | **无**，错了运行时才崩 | **有**，依赖缺失直接编译失败 |
| 构建速度 | 快（不跑注解处理器、不生成代码） | 慢（KAPT/KSP + 生成大量代码） |
| 包体积 | 小 | 生成代码较多，通常更大 |
| 运行时开销 | 极小；有解析开销但默认单例缓存 | 生成代码直接 new，几乎为零 |
| KMP 多平台 | 天然支持，共享 DI 代码 | 主要面向 Android/JVM，KMP 支持弱 |
| Compose 支持 | 好，`koinViewModel()` / `koinInject()` | 用 `hiltViewModel()`，同样好 |
| 测试便利性 | 极易替换（`loadKoinModules`/`unloadKoinModules`） | 用 `@HiltAndroidTest` + `@UninstallModules` |
| 官方推荐度 | 社区广泛使用，Google 官方推荐 Hilt | **Google 官方 Android DI 首选** |

**选型建议**：

- **官方默认答案是 Hilt**（尤其大型、多人协作、想靠编译器兜底的项目）；
- 追求**简洁、快速开发、KMP 共享、不想被注解处理器拖慢构建**，选 Koin；
- 中小项目 / 原型 / Compose 快速迭代：Koin 的体验优势明显；
- 对"上线前必须保证依赖图正确"零容忍：Hilt 更稳。

---

## 二、使用场景

| 场景 | 为什么用它 | 关键点 |
| --- | --- | --- |
| 中小型 App / 独立开发 | 上手快、样板代码少 | 一套 `StartKoin` + 几个 module 就能跑 |
| KMP 多平台共享 DI | 代码不依赖注解处理器，各平台共用 | DI 声明放在 commonMain |
| 快速原型 / Demo | 改依赖不需要重编译注解 | 改完立刻生效 |
| Compose 项目 | `koinViewModel()` 与 Compose 配合顺滑 | 注意 viewModel 的 scope 归属 |
| 想做模块化但不想上 Dagger | module 可拆分包、按需加载 | 拆分的粒度要规划好 |
| 测试替换依赖 | `loadKoinModules` 动态注入 mock | 测试隔离非常方便 |

### 不适合的场景

- **超大型、多人协作、强编译期保障要求的项目**：Koin 的"运行时才崩"在规模化开发中会成为风险，Hilt 更稳。
- **对启动性能极致敏感的场景**：Koin 启动时要遍历 module 建立容器；虽然开销很小，但在极端场景下是额外成本。
- **团队已经重度使用 Hilt/Dagger**：没必要为了"简洁"引入两套 DI。

---

## 三、常用 API 与使用方法

### 3.1 启动 Koin

```kotlin
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidLogger()                       // Android 日志
            androidContext(this@App)              // 注册 Application 作为 Context 来源
            modules(appModule, dataModule, viewModelModule)   // 加载所有模块
        }
    }
}
```

### 3.2 定义模块（覆盖常用注册方式）

```kotlin
// 数据库：全局单例
val dataModule = module {
    single {
        Room.databaseBuilder(androidContext(), AppDatabase::class.java, "app.db").build()
    }
    single { get<AppDatabase>().userDao() }
    single<ApiService> { RetrofitService.create() }

    // factory：每次 get 都新建，适合无状态、轻量的对象
    factory { CreateUserUseCase(get()) }

    // scoped：在指定 scope 内是单例，超出 scope 就销毁（比如按页面/会话）
    scope(named("SessionScope")) {
        scoped { SessionManager(get()) }
    }

    // named / qualifier：同类型多个实现要区分
    single(named("prod")) { ServerConfig("https://api.example.com") }
    single(named("mock")) { ServerConfig("http://localhost") }

    // parametersOf：创建时需要运行时参数
    factory { (userId: Long) -> UserDetailViewModel(userId, get()) }
}

val viewModelModule = module {
    viewModel { HomeViewModel(get()) }            // 自动绑定 ViewModel 生命周期
}
```

### 3.3 注入方式对比

| 方式 | 写法 | 是否推荐 |
| --- | --- | --- |
| 构造函数注入 | `class Repo(val api: ApiService)` + `factory { Repo(get()) }` | **强烈推荐**：显式、可测 |
| `by inject()` | `val repo: Repo by inject()` | 惰性、字段注入，测试难替换 |
| `get()` | `val repo = get<Repo>()` | 立即解析，命令式 |
| `koinViewModel()` | Compose/Activity 里拿 ViewModel | 推荐（ViewModel 场景） |
| `KoinComponent` | `class X : KoinComponent { val y by inject() }` | 过渡用法，能用构造函数就别用 |

**推荐写法：构造函数注入 + 在 module 里用 `get()` 组装。**

```kotlin
class HomeViewModel(private val repo: UserRepository) : ViewModel()  // 依赖显式声明

val viewModelModule = module {
    viewModel { HomeViewModel(get()) }   // 只在这里"接线"
}
```

### 3.4 模块拆分的推荐做法

- **按 feature 拆**：`featureHomeModule`、`featureProfileModule`，每个 feature 自带自己的 ViewModel 和 UseCase；
- **按层拆**：`dataModule`（网络、数据库、Repository）、`domainModule`（UseCase）、`presentationModule`（ViewModel）；
- 大型项目可以两者结合：core 层按层、业务层按 feature；
- 保持"谁声明谁加载"，避免 `startKoin` 里堆一大串。

### 3.5 在 Compose / Activity 里取依赖

```kotlin
// Activity
private val viewModel: HomeViewModel by viewModel()

// Compose
@Composable
fun HomeScreen(vm: HomeViewModel = koinViewModel()) { /* ... */ }

// 直接注入任意类型
val repo: UserRepository = koinInject()
```

---

## 四、常见坑与排查

### 4.1 忘注册 → 运行时崩（最致命）

`NoDefinitionFoundException: No definition found for class 'Foo'`。这是 Koin 唯一但最要命的缺点——**编译器不会提醒你**，只有跑到那行代码才崩。

缓解手段：
1. **在模块里集中声明**，避免散落各处；
2. **写 `checkModules` 校验测试**（见 4.6），在 CI 里跑，把运行时错误提前到测试期；
3. 命名规范，`single`/`factory`/`viewModel` 声明位置固定。

### 4.2 多模块 modules 加载不全

模块化项目里，feature 模块的 Koin module 没被 `startKoin` 加载，就会出现"主页能进、点进详情页崩"。做法：每个 feature 暴露自己的 `val xxxModule`，由壳工程统一收集加载，或按需 `loadKoinModules()`（注意要配对 `unloadKoinModules()`）。

### 4.3 `viewModel {}` 报错找不到

用了 `viewModel { }` 但没加 `koin-android-viewmodel`（Koin 3.2+ 已合并进 `koin-android`）相关的依赖，或没引入 ViewModel 的 DSL。检查依赖是第一步。

### 4.4 单例持有 Context 造成内存泄漏

```kotlin
// ❌ 直接把 Activity 塞进单例
single { SomeManager(activity) }

// ✅ 用 androidContext() 拿 Application
single { SomeManager(androidContext()) }
```

单例的生命周期和进程一样长，持有 Activity 会导致 Activity 无法回收。需要 Context 时统一用 `androidContext()`（就是 Application），或者用 `get<Context>()`（前提是 `androidContext()` 已注册）。

### 4.5 `GlobalContext` 已废弃

Koin 2.x 里常用 `GlobalContext.get()` 这种全局取法，Koin 3 已废弃。**不要再用它**，改用 `getKoin()` / 注入 / `KoinComponent`。看到老代码里的 `GlobalContext` 就知道是历史遗留。

### 4.6 测试隔离：`loadKoinModules` / `unloadKoinModules` / `KoinApplication`

- 已有全局 Koin 的测试：用 `loadKoinModules(module { single<ApiService> { mockApi } })` 覆盖，测完 `unloadKoinModules(...)`；
- 完整隔离的测试：用 `KoinApplication.init().modules(testModule)` 起一个独立容器，不污染全局。

**关键：必须写 `checkModules` 校验测试。**

```kotlin
class ModuleCheckTest {
    @Test
    fun verifyModules() {
        // 校验依赖图能否被完整解析（Koin 3 的模块校验能力）
        koinApplication {
            modules(appModule, dataModule, viewModelModule)
            // checkModules() / verify() 会在解析失败时报错
            checkModules()
        }
    }
}
```

有了这类测试，Koin 最大的缺点（运行时才发现缺依赖）就被大幅缓解——这也是面试里能加分的点：**"我们不能改变它运行时解析的本质，但可以用 checkModules 把它拉到测试期暴露"。**

### 4.7 其它高频杂项

- `factory` 用在有状态对象上是坑：每次 `get()` 都是新实例，状态不会共享；
- `named` / `qualifier` 用错了会拿到同类型的另一个实现，很难发现——类型安全无法保证，靠命名纪律；
- `parametersOf()` 参数要和 lambda 形参类型一一对应，顺序/类型对不上会崩；
- 别在 `single` 里做耗时初始化（如网络请求），会阻塞首次解析。

---

## 五、面试追问速答

**Q1：Service Locator 和依赖注入的区别？**
依赖注入（DI）强调"依赖由外部传入"，对象不关心依赖从哪来，最典型的是**构造函数注入**；Service Locator 是"对象主动向一个中心容器要依赖"，比如 `get<T>()` / `by inject()`，对象知道容器的存在。Koin 两者都能做：用构造函数注入时是 DI，用 `by inject()` 时是 Service Locator。区别在于耦合度——构造函数注入的类可以在不启动 Koin 的情况下被单测直接 new，Service Locator 的类必须依赖容器。

**Q2：Koin 为什么会有运行时崩溃，怎么缓解？**
因为它的依赖图在运行时解析，没有注解处理器做编译期校验，所以"没注册""类型不匹配"这类错误编译器看不到，只会在 `get()` 那一刻抛 `NoDefinitionFoundException`。缓解办法：集中声明模块、写 `checkModules` 校验测试放进 CI、用构造函数注入让依赖显式化。本质上无法消除，只能前移暴露时间。

**Q3：Koin 和 Hilt 怎么选？**
看三个维度：**团队规模**（大团队/强规范 → Hilt，编译期保障更重要）、**项目形态**（KMP 多平台 → Koin，Hilt 支持弱）、**迭代节奏**（原型/快速迭代 → Koin 构建快、样板少）。Google 官方推荐 Hilt 作为 Android 默认 DI，所以"没特别理由选 Hilt"是安全答案；"要 KMP 共享或极度追求简洁"就选 Koin。

**Q4：为什么推荐构造函数注入而不是字段注入？**
构造函数注入让依赖**显式写在类的签名上**：一看就知道这个类需要什么、能直接 `new` 出来做单测、不会出现"对象已创建但依赖还没注入"的中间态。字段注入（`by inject()`）依赖隐式、创建和注入分离，测试时必须先启动 Koin 容器，坏味道更重。所以规范是"类里用构造函数注入，只在 module 里用 `get()` 接线"。

**Q5：Koin 的 single / factory / scoped 有什么区别？**
`single` 全局唯一、懒加载创建后缓存复用；`factory` 每次 `get()` 都新建实例，适合无状态的轻量对象；`scoped` 在指定 scope 内是单例、离开 scope 即销毁，适合"一次会话/一个页面"级别的共享对象。用错会出问题——比如把有状态的东西声明成 `factory`，会导致状态不共享、每次都是新的。
