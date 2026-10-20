---
title: Glide 与 Coil 对比（区别 / 优缺点 / 选型）
date: 2026-10-20 22:53:40
---

# Glide 与 Coil 对比（区别 / 优缺点 / 选型）

> 大白话版。两者都是"给一个 URL / 资源 ID，帮我把图加载并显示到 ImageView 上"，差别在**内存复用方式、线程模型、依赖体积**这三件事上。

---

## 一、一句话结论

- **Glide**：功能大而全的老牌重装武器。Java 生态、自研 Engine + 线程池、**有 BitmapPool**、GIF/视频帧内置。
- **Coil**：Kotlin 原生轻量派。协程 + OkHttp、体积小、Compose 一等公民、**没有 BitmapPool**。

新项目是 Kotlin / Compose → 优先 **Coil**；老项目重媒体、图片密集长列表 → 留 **Glide**。

## 二、调用方式差在哪

```kotlin
// Glide：RequestBuilder 链式，三件套 with/load/into
Glide.with(context)
    .load(url)
    .placeholder(R.drawable.placeholder)
    .centerCrop()
    .transition(DrawableTransitionOptions.withCrossFade())
    .into(imageView)

// Coil：ImageView 扩展函数，配置写在 lambda 里
imageView.load(url) {
    placeholder(R.drawable.placeholder)
    crossfade(true)
    transformations(CircleCropTransformation())
}

// Coil 在 Compose 里最舒服
AsyncImage(model = url, contentDescription = null, contentScale = ContentScale.Crop)
```

一个明显的体感差异：Glide 所有配置都是链式方法，想复用一套配置得自己封装；Coil 可以直接把 `ImageRequest` 对象（其实是个 data class）缓存起来复用。

## 三、核心区别逐项拆

| 维度 | Glide | Coil |
| --- | --- | --- |
| 语言 / 生态 | Java，注解处理器（kapt/KSP）生成 `GlideApp` 扩展 API | 纯 Kotlin，协程 + OkHttp，无注解处理器 |
| 线程模型 | 自研 Engine，两个优先级线程池（DiskCacheExecutor / SourceExecutor） | 结构化并发，取消请求会**真正中断上游网络读** |
| 生命周期 | 绑 Activity/Fragment，`onStop` **暂停**队列、`onStart` 恢复 | 跟随 ImageView attach/detach 取消；2.x 起可传 `lifecycle` 做 pause/resume |
| 内存缓存 | `ActiveResources`（弱引用，正在显示的不进 LRU）+ `LruResourceCache` + **`BitmapPool`** | `MemoryCache`（LRU，可选弱引用），**没有 BitmapPool** |
| 磁盘缓存 | 自研 DiskLruCache 实现，默认约 **250MB** | 复用 OkHttp 的 `DiskCache`，默认按可用空间 **2%** 算 |
| 内存上限 | `MemorySizeCalculator` 按 memoryClass / 屏幕尺寸 / 是否低内存设备动态算 | `availableMemoryPercentage`，默认 **25% 可用内存** |
| 格式支持 | **内置** GIF、动画 WebP、视频首帧 | 需额外 artifact：`coil-gif` / `coil-video` / `coil-svg` |
| 变换能力 | 内置 circleCrop / roundedCorners / mask，支持多变换串联 | 需 `coil-transformations` 扩展库 |
| 缩略图 | `.thumbnail()` 原生支持 | 需自己先加载小图 |
| 预加载 / 只下载 | `preload()` / `downloadOnly()` 成熟 | 也可用 `ImageRequest` 做 preload / 只写 disk cache，但 API 更绕 |
| 包体积 | 大（依赖多 + 注解处理器），方法数明显更高 | 小，复用项目已有的 OkHttp / 协程，官方称约 Glide 的一半 |
| Compose | 有 `glide-compose`，接入较晚 | `coil-compose` 是首选，Android 官方文档并列推荐 |
| 多平台 | 仅 Android | Coil 3 起支持 KMP（Android / iOS / JS / Desktop） |

### 3.1 缓存链路是最大的分水岭

**Glide 是"四级内存链路"，Coil 是"两级"。**

Glide 取一张图依次找：

1. `ActiveResources`：弱引用 Map，**正在显示**的图放这，不进 LRU，避免被淘汰后又被重新解码；
2. `LruResourceCache`：常规内存 LRU（按字节数计容量）；
3. `BitmapPool`：**不用 Resource 本身，只回收 Bitmap 内存块**，下次解码同尺寸图直接复用这块内存，省掉 malloc + GC；
4. 磁盘 / 网络。

第 3 步是 Glide 在"大图长列表快速滚动"场景明显更稳的根本原因——它省的是**内存分配**，不是省解码。

Coil 只有内存 LRU + 磁盘 LRU，没有 pool，所以滚动时 GC 压力通常更高，内存峰值也更高一些。

### 3.2 线程与取消

- Glide 的 Engine 自己管两套优先级线程池，`into()` 之后请求和 ImageView / 生命周期绑定，`clear()` 或 detach 时取消。它不是协程，所以在 Kotlin 里配合协程只能用 `submit().get()` / `requestFuture()` 桥接。
- Coil 每个请求就是一个协程 job，`imageView.load()` 内部在 ImageView detach 时取消。好处是取消是**结构化、可传递**的：网络层的 OkHttp 调用会一并取消，不做无用功。

### 3.3 生命周期

- Glide 通过注入一个隐藏 Fragment 来监听宿主生命周期，`onStop` 是**暂停**（停掉队列、保留已解码结果），回前台自动重发。这个设计对"列表里加载到一半退出"的省电体验更好。
- Coil 默认只做"detach 就取消"，2.x 之后才补上 `lifecycle` 参数来支持真正的 pause/resume。用法上要显式传：

```kotlin
imageView.load(url) {
    lifecycle(lifecycleOwner.lifecycle)   // 停了暂停，回来继续
}
```

## 四、Glide 的优缺点

**优点**
- 功能最全：GIF、动画 WebP、视频首帧、`.thumbnail()`、多变换串联，基本不用自己造轮子。
- **BitmapPool 减少内存分配**，列表滚动更稳，大图 OOM 概率更低。
- 生命周期管理最彻底（暂停而非取消），体验上的细腻度更高。
- 生态成熟，RecyclerView 复用错位、长图加载、占位图这类疑难问题社区都有现成答案。

**缺点**
- 重：包体积、方法数、编译期成本都高，注解处理器和 Kotlin 2.0 / KSP 有兼容摩擦。
- API 啰嗦，`with/load/into` 三件套在 Compose 里很别扭。
- 无原生协程支持，和现代 Kotlin 代码风格格格不入。
- 只支持 Android，没有多平台余地。

## 五、Coil 的优缺点

**优点**
- 轻：体积小、方法数小、无注解处理器、编译快；复用项目里已有的 OkHttp，不引入重复依赖。
- API 极简，`load()` 一个扩展函数；Compose 里 `AsyncImage` 体验最好。
- 协程原生，天然可取消、可并发编排（多个图 `awaitAll`），不会写回调地狱。
- 默认配置合理（内存 25%、磁盘 2% 可用空间自动算），开箱即用。

**缺点**
- **没有 BitmapPool**，大图长列表滚动内存峰值更高、GC 更活跃。
- GIF / 视频帧 / SVG 要额外引 artifact，是"按需拼装"而不是开箱全有。
- 迭代快、有破坏性变更：Coil 2 → 3 网络层拆包（`coil-network-okhttp` / `coil-network-ktor`）、API 调整，升级要改代码。
- 高级能力（缩略图、复杂变换、复杂动画）不如 Glide 丰富，部分场景要自己扩展。
- 生态厚度不如 Glide，第三方扩展（如各种自定义 decoder）更少。

## 六、选型建议

| 场景 | 建议 |
| --- | --- |
| Kotlin + Compose 新项目 | **Coil**，接入成本最低 |
| 图片密集的复杂列表 / 重媒体（GIF、视频帧） | **Glide**，BitmapPool + 格式内置 |
| 对包体积敏感 | **Coil** |
| 已有 Java + Glide 的老项目 | 不动，迁移收益低于风险 |
| 多平台（KMP） | **Coil 3** |
| 混合项目 | 可以共存，但注意两套内存缓存会**互相挤占**，最好统一入口 |

## 七、面试追问速答

- **为什么 Glide 不用协程？** Glide 诞生于协程之前，它那套 Engine + 优先级线程池要解决的是"同一张图被多个 target 共享、请求可合并、可暂停恢复"的问题，用自定义引擎更好控制；而且它的取消粒度是绑 target 的，不是绑调用栈的。
- **Coil 为什么不做 BitmapPool？** 官方取舍是简单 + 轻量。BitmapPool 需要精确管理 Bitmap 生命周期和尺寸匹配，容易在 API 28 以下出现"复用后被外部持有"的崩溃风险，Coil 选择把这个复杂度交给系统分配器。
- **Glide 的 ActiveResources 为什么必须存在？** 因为它存的是弱引用，正在显示的图不走 LRU，否则会出现"图正在屏幕上，但被 LRU 淘汰 → 又触发一次解码"的循环加载问题。
- **怎么自己判断该用哪个？** 看两个指标：**项目里图片的平均尺寸和滚动帧率要求**（大图 + 列表 → Glide），以及**包体积预算**（敏感 → Coil）。
- **Coil 取消真的能省流量吗？** 能，但要看网络层实现。用 OkHttp 且支持 cancel 时，请求会真正中断；如果包裹的是不支持取消的自定义 fetcher，就只是丢弃结果。

## 八、常见坑

- **Glide 在 RecyclerView 里复用错位**：`into()` 一定要配上 placeholder / error，否则复用时旧图会残留在 View 上；另外 `clear()` 要在合适时机调用，别在 `onBindViewHolder` 里无条件 clear 造成闪烁。
- **Coil 的 `size` 解析时机**：`imageView.load()` 依赖 ImageView 当前尺寸做下采样，如果 View 还没布局完（宽高为 0），解码尺寸会不准。要主动指定 `.size(w, h)` 或 `.size(Size.ORIGINAL)`。
- **两套库的内存缓存不共享**：同一个 App 里既有 Glide 又有 Coil，等于开了两份内存 + 两份磁盘缓存，容易把内存顶爆，建议统一成一个 ImageLoader 入口。
- **Coil 3 迁移**：网络层从内置改成显式依赖，升级时必须显式加 `coil-network-okhttp`，否则会报"没有网络 fetcher"。
