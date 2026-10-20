---
title: Retrofit：原理 / 使用场景 / 常用 API / 常见坑
date: 2026-10-21 00:24:57
---

# Retrofit：原理 / 使用场景 / 常用 API / 常见坑

> 一句话定位：Retrofit 是 OkHttp 之上的一层「声明式 REST 客户端」——你用注解把 HTTP 接口"声明"出来，它用**动态代理**把接口方法调用翻译成 HTTP 请求，真正发请求的活交给 OkHttp。
> 提示：本文讲的是通用原理与用法，不绑定具体版本号；个别 API 在你项目里的版本上可能有细微差异。

---

## 一、Retrofit 是什么（原理）

**一句话：接口 + 注解 = HTTP 请求，中间靠"动态代理 + 两大扩展点"完成翻译。**

### 1.1 核心链路

```
你的业务代码
   ↓  调用 interface ApiService 上的方法（其实是 Proxy 的 invoke）
Proxy.newProxyInstance 生成的动态代理对象
   ↓  把「方法 + 注解 + 参数」解析成 ServiceMethod（含 OKHttp 的 Request 模板）
   ↓
OkHttpCall（实现 Call<T>）→ 交给 OkHttp 发真正的 HTTP 请求
   ↓
OkHttp（连接池 / 拦截器 / 缓存 / 重试 / HTTP2）
   ↓
拿到 ResponseBody
   ↓  Converter 反序列化成你的数据模型 T
返回给你（或经 CallAdapter 包装成你要的返回类型）
```

### 1.2 四个必须记住的点

**① 动态代理（`Proxy.newProxyInstance`）**

Retrofit 的 `create(ApiService::class.java)` 并不会真的实现你的接口，而是返回一个 JDK 动态代理对象。你调用 `api.getUser(1)` 时，实际进到 `InvocationHandler.invoke()`，Retrofit 在这里把「方法签名 + 注解 + 实参」解析成一次 HTTP 请求。

- 好处：接口零实现、声明式、写法极简；注解在**运行时**才被解析，不需要注解处理器、不生成代码。
- 代价：注解解析有**运行时开销**，所以 Retrofit 把解析结果缓存在 `serviceMethodCache` 里（按 Method 缓存），第二次调用直接复用，不再是每次重新解析。

**② 注解解析成 `ServiceMethod`**

`@GET("users/{id}")`、`@Path("id")`、`@Query("page")` 这些注解会被解析成 `RequestFactory`/`ServiceMethod`，它记录了：

- HTTP 方法（GET/POST/...）和相对 URL；
- 参数如何拼进 URL、Header、请求体（由每个参数的注解决定）；
- 用哪个 `Converter` 做请求体和响应体的序列化。

**③ 两大扩展点：Converter 与 CallAdapter**

| 扩展点 | 干什么 | 例子 |
| --- | --- | --- |
| **Converter**（`Converter.Factory`） | 请求体序列化 + 响应体反序列化 | `GsonConverterFactory`、`MoshiConverterFactory`、`kotlinx.serialization` 的 converter、`ScalarsConverterFactory`（纯字符串） |
| **CallAdapter**（`CallAdapter.Factory`） | 把 `Call<T>` 转成别的返回类型 | 内置支持 `Call<T>`；官方 `retrofit2-kotlin-coroutines`（现 Retrofit 主库已内置 suspend 支持）；`Flow`/`LiveData` 需要额外适配器 |

一句话区分：**Converter 管"数据长什么样"，CallAdapter 管"返回什么壳子"。**

**④ suspend 函数是怎么被支持的**

`suspend fun getUser(): User` 能直接写，是因为 Retrofit 内置了一个协程 CallAdapter：

- 它把 suspend 方法适配成 `Call<T>`；
- `Call.enqueue()` 的回调结果，通过 `kotlin.coroutines` 的 `suspendCancellableCoroutine` 桥接回协程（用 `Continuation` 恢复）；
- 请求执行在 OkHttp 的线程池上，不阻塞调用方线程；
- 协程被取消时会走 `Call.cancel()`，网络请求随之取消。

所以 suspend 版本的异常是**直接抛出来的**（不需要你判 `isSuccessful`），而 `Call<T>` 版本要自己判。

---

## 二、使用场景

| 场景 | 为什么用它 | 关键点 |
| --- | --- | --- |
| 标准 REST 接口调用 | 声明式、注解即协议，代码量极少 | 一个接口对应一组后端资源 |
| 文件上传 / 下载 | 复用 OkHttp 的连接池和流式能力 | 上传用 `@Multipart`，下载用 `@Streaming` + `ResponseBody` |
| 动态 baseUrl | 不同环境/租户切换域名 | `@Url` 整体覆盖，或 `@GET` 传完整 URL |
| 多 baseUrl 分组 | 业务域不同（用户域/内容域/支付域） | **不同域名各建一个 Retrofit 实例**，共享同一个 OkHttpClient |
| 统一鉴权 / 日志 / 重试 | 拦截器机制统一处理 | 交给 OkHttp 的 `Interceptor` / `Authenticator` |
| 多序列化库混用 | 老接口 JSON、新接口 Protobuf | 按顺序注册多个 Converter，或给方法单独指定 |

### 不适合的场景

- **纯 WebSocket / 长连接**：Retrofit 是请求-响应模型，实时推送应该直接用 OkHttp 的 `newWebSocket`。
- **需要极致定制 HTTP 语义（如自定义帧、非标准协议）**：直接用 OkHttp 更直接，多一层注解反而别扭。
- **超时 / 重试 / 连接池 / 缓存**：这些**不属于 Retrofit**，全在 OkHttp 上配，写错地方是新手最常见的坑。
- **服务端返回的是二进制大文件且要边下边写**：Retrofit 能做，但要用 `@Streaming`，否则会一次性读进内存导致 OOM。

---

## 三、常用 API 与使用方法

### 3.1 构建 Retrofit（标准四件套）

```kotlin
val okHttp = OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS)     // 超时在 OkHttp 配，Retrofit 不管
    .readTimeout(30, TimeUnit.SECONDS)
    .writeTimeout(30, TimeUnit.SECONDS)
    .addInterceptor(HeaderInterceptor())      // 统一 Header / Token
    .addInterceptor(HttpLoggingInterceptor().apply { level = BASIC })
    .build()

val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/")      // 必须以 / 结尾
    .client(okHttp)                           // 不传就用默认 OkHttpClient
    .addConverterFactory(GsonConverterFactory.create())   // 响应体反序列化
    .addCallAdapterFactory(RxJava3CallAdapterFactory.create()) // 可选，转 RxJava
    .build()

val api = retrofit.create(ApiService::class.java)
```

### 3.2 定义接口（覆盖常用注解）

```kotlin
interface ApiService {

    // GET + 路径占位 + 查询参数
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Long, @Query("detail") detail: Boolean = true): User

    // 查询参数按 Map 批量拼
    @GET("search")
    suspend fun search(@QueryMap params: Map<String, String>): List<Item>

    // POST + JSON 请求体
    @POST("users")
    suspend fun createUser(@Body body: CreateUserReq, @Header("X-Trace-Id") traceId: String): User

    // 表单提交（application/x-www-form-urlencoded）
    @FormUrlEncoded
    @POST("login")
    suspend fun login(@Field("account") account: String, @Field("pwd") pwd: String): Token

    // 多文件上传
    @Multipart
    @POST("upload")
    suspend fun upload(
        @Part("desc") desc: RequestBody,
        @Part file: MultipartBody.Part
    ): UploadResult

    // 下载大文件：加 @Streaming 才不会一次性读进内存
    @Streaming
    @GET
    suspend fun download(@Url url: String): ResponseBody

    // 动态覆盖 baseUrl
    @GET
    suspend fun dynamic(@Url fullUrl: String): Response<User>
}
```

### 3.3 返回值三种形态对比

| 写法 | 特点 | 异常怎么处理 |
| --- | --- | --- |
| `suspend fun x(): T` | 最简洁，非 2xx 会**直接抛异常** | 直接 try/catch |
| `suspend fun x(): Response<T>` | 拿到状态码/头/原始响应，非 2xx 不抛 | 自己判 `response.isSuccessful` 和 `response.code()` |
| `fun x(): Flow<T>` | 适合要在流里做重试/去抖/取消的组合 | **不内置**，需要额外的 CallAdapter 适配 |

> 注意：`Flow<T>` 不是 Retrofit 自带的。要么用 `retrofit2-kotlinx-serialization` 之外的三方 adapter，要么自己用 `flow { emit(api.x()) }` 包一层。**面试常被问这句，别答成"Retrofit 原生支持 Flow"。**

### 3.4 拦截器：统一 Header / Token / 重试

```kotlin
class HeaderInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val newReq = chain.request().newBuilder()
            .header("Authorization", "Bearer ${TokenStore.token}")  // 统一加 Token
            .header("Platform", "android")
            .build()
        return chain.proceed(newReq)
    }
}
```

### 3.5 API / 注解速查表

| 注解 | 作用 |
| --- | --- |
| `@GET` / `@POST` / `@PUT` / `@DELETE` / `@PATCH` / `@HEAD` / `@OPTIONS` | HTTP 方法 |
| `@Path` | 替换 URL 里的 `{占位符}`；`encoded = true` 表示值已编码、不再转义 |
| `@Query` / `@QueryMap` | 拼到 URL 问号后面的查询参数 |
| `@Field` / `@FieldMap` | 表单字段，需配 `@FormUrlEncoded` |
| `@Body` | 请求体，交给 Converter 序列化 |
| `@Header` / `@Headers` | 单个 / 固定多个请求头 |
| `@Url` | 动态传入完整或相对 URL，覆盖 baseUrl |
| `@Multipart` / `@Part` | 多部分上传（文件、混合字段） |
| `@Streaming` | 流式读取响应体，适合大文件下载 |
| `@FormUrlEncoded` | 声明表单编码方式 |
| `Response<T>` / `Call<T>` | 带完整响应的返回壳 / 可取消的调用对象 |
| `suspend` / `Flow` | 协程返回 / 需要额外 adapter 的流式返回 |

---

## 四、常见坑与排查

### 4.1 `baseUrl` 必须以 `/` 结尾

```kotlin
.baseUrl("https://api.example.com")   // ❌ 运行期抛 IllegalArgumentException
.baseUrl("https://api.example.com/")  // ✅
```

原因是 Retrofit 用 baseUrl 的 path 前缀去和 `@GET("users")` 的相对路径做拼接，不以 `/` 结尾会被判定为"最后一节是文件名"，拼出来路径错。**这是最高频的第一次踩坑。**

### 4.2 `@Path` 传特殊字符

如果路径参数里含 `/`、`?`、`%` 等，默认会被 URL 编码。若参数**已经是编码好的**，要加 `encoded = true`，否则会被二次编码。

### 4.3 超时配错地方

Retrofit 本身**没有**超时配置，`connectTimeout` / `readTimeout` / `writeTimeout` / `callTimeout` 全在 `OkHttpClient.Builder` 上。在 Retrofit.Builder 上找不到这些方法不是 bug。

### 4.4 主线程同步 `execute()`

`Call.execute()` 是同步阻塞的，在主线程调用会抛 `NetworkOnMainThreadException`。要么用 `enqueue()`，要么用 suspend 函数。

### 4.5 `Unable to create converter`

报 `IllegalStateException: Unable to create converter for class ...`，原因通常是：

- 忘记 `addConverterFactory(...)`；
- 加了 Gson 但返回类型是 `String`，Gson 处理不了 → 补一个 `ScalarsConverterFactory`；
- 返回类型不是接口声明里的泛型（如 `List<Foo>` 没配对）。

排查顺序：先看返回类型 T 是否被某个已注册 Converter 支持。

### 4.6 返回 `Flow<T>` 直接报错

`Flow` 不是 Retrofit 默认支持的返回类型，会在创建接口时抛"未找到 CallAdapter"。要么加支持 Flow 的 CallAdapter，要么在业务层自己包 `flow {}`。**别在生产上想当然。**

### 4.7 统一错误处理要分三层

```
第一层：网络异常（IOException / UnknownHost / SocketTimeout）→ 无网、超时、DNS 失败
第二层：HTTP 错误码（4xx / 5xx）→ 服务器拒绝、鉴权失败、服务端异常
第三层：业务错误码（HTTP 200 但 body 里 code != 0）→ 后端业务逻辑错误
```

三层要分别处理、分别上报，否则线上根本分不清"是网络问题还是业务问题"。

### 4.8 Token 过期刷新（并发风暴）

单请求刷新简单；难点是**多个请求同时收到 401**，会触发多次刷新。两种主流做法：

- **`Authenticator`**（OkHttp 404/401 时回调）：返回值会自动重放原请求。注意它可能被并发调用，需要加锁 + 判断"是否已被别的请求刷新过"（比较请求里携带的旧 token，若当前 token 已变则直接用新 token 重放，不再刷新）。
- **`Interceptor` + 并发排队**：拦截到 401 时，第一个请求去刷新，其余请求挂起等待；刷新完成后统一用新 token 重放；刷新失败则全部失败。

关键点：**刷新用独立的 Retrofit/OkHttp 实例**（不要用带 401 拦截器的那个），否则刷新请求自己 401 会死循环。

### 4.9 其它高频杂项

- 一个 `OkHttpClient` 全局共享，`Retrofit` 实例可以按 domain 多建，但别每次都 new 新的 `OkHttpClient`（连接池和线程池会浪费）。
- `@Body` 传 null 会抛异常，可选体用可空 `RequestBody?`。
- Gson 反序列化对 Kotlin 非空字段不友好（字段缺失会得到 null 但不报错），必要时用 Moshi 或 kotlinx.serialization。

---

## 五、面试追问速答

**Q1：Retrofit 为什么用动态代理？有什么代价？**
接口本身没有实现类，用 JDK 动态代理可以在运行时"凭空"生成一个实现了该接口的对象；它的 `invoke` 里拿到 Method 和实参，把注解解析成 HTTP 请求。好处是接口零实现、声明式、不生成代码；代价是注解解析发生在**运行时**而非编译期，所以 Retrofit 用 Method 级别的缓存（serviceMethodCache）避免重复解析。另外它无法在编译期校验接口写错（比如 `@GET` 配了 `@Body`），错误要跑到运行时才暴露。

**Q2：Converter 和 CallAdapter 的区别？**
Converter 负责**数据内容的序列化/反序列化**，决定请求体怎么变成字节、响应字节怎么变成 `User` 这类模型，比如 Gson/Moshi。CallAdapter 负责**返回类型的外壳转换**，决定接口方法返回的是 `Call<T>`、`suspend T`、还是 `Observable<T>`。一句话：Converter 管"数据长什么样"，CallAdapter 管"返回什么包装"。

**Q3：Retrofit 和 OkHttp 的分工？**
OkHttp 是**真正的 HTTP 客户端**：管连接池、TLS、拦截器、缓存、重试、HTTP/2、超时。Retrofit 是**声明式的上层封装**：管注解解析、URL 拼接、参数绑定、数据转换。Retrofit 的所有网络能力都来自 OkHttp，你可以理解为"Retrofit 把 OkHttp 用接口的方式重新表达了一遍"。

**Q4：suspend 方法内部怎么实现的？异常抛什么？**
Retrofit 内置协程 CallAdapter，把 suspend 方法转成 `Call<T>`，再用 `suspendCancellableCoroutine` 把 `enqueue` 的回调桥接成协程的恢复/取消。请求跑在 OkHttp 线程池上，协程取消时会 `Call.cancel()`。`suspend fun x(): T` 形态下，非 2xx 会抛 `HttpException`，网络层问题是 `IOException` 及其子类——**它不会给你 `Response`，需要你自己 try/catch**；要拿状态码就改用 `Response<T>`。

**Q5：怎么做统一错误处理和 Token 刷新？**
统一处理放在 OkHttp 拦截器里，分三层识别：IOException（网络）、`response.code`（HTTP 状态码）、body 里的业务 code。Token 刷新用 `Authenticator` 或拦截器 + 并发排队：拦截 401 → 只让第一个请求去刷新 → 其余挂起等待 → 刷新成功统一用新 token 重放，失败则整体失败。刷新请求必须走**不带该拦截器的独立客户端**，避免刷新自身 401 造成死循环。并发场景要加锁并判断 token 是否已被刷新，防止"刷新风暴"。

**Q6：Retrofit 支持 HTTP/2 吗？**
支持，但**不是 Retrofit 的能力**——HTTP/2 由底层 OkHttp 提供（OkHttp 默认在 TLS 上协商 ALPN 使用 HTTP/2，也支持 h2c），Retrofit 只是转发。你不需要在 Retrofit 上做任何配置，只要服务端和 OkHttp 协商成功即可。
