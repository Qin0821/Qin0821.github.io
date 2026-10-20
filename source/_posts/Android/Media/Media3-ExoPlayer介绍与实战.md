---
title: Media3 ExoPlayer 介绍与实战
date: 2026-10-20 23:44:20
---

# Media3 ExoPlayer：介绍 / 原理 / 使用场景 / 常用 API / 常见问题

> 对应面试题 Q036~Q039。大白话讲清"Media3 和 ExoPlayer 到底什么关系、播放器内部怎么跑、Feed 里怎么复用、坑在哪"。
> 版本提示：以 Media3 1.x 为准（对应旧的 `com.google.android.exoplayer2` 已被废弃）。具体 API 在你项目里的版本上可能有小差异。

---

## 一、Media3 和 ExoPlayer 是什么关系

**一句话：Media3 是一个"套件"，ExoPlayer 是这个套件里的播放器实现。**

以前 ExoPlayer 是独立库（`com.google.android.exoplayer:exoplayer`），`androidx.media` 又是另一套（管通知、蓝牙控制）。Google 把这两条线合并进了 Jetpack，叫 **androidx.media3**：

```
androidx.media3（套件）
├── media3-common          通用接口：Player / MediaItem / MediaMetadata / 常量
├── media3-exoplayer        ★ 播放器实现（ExoPlayer），核心
├── media3-exoplayer-hls    HLS 协议支持
├── media3-exoplayer-dash   DASH 协议支持
├── media3-exoplayer-rtsp   RTSP 支持
├── media3-exoplayer-ima    广告（IMA SDK）
├── media3-extractor        容器解析（MP4/TS/FLV...）
├── media3-datasource-okhttp / -cronet   HTTP 数据源实现
├── media3-database         缓存/下载用的数据库层
├── media3-ui               传统 View 的 UI：PlayerView / PlayerControlView
├── media3-ui-compose       Compose 的 UI：PlayerSurface 等
├── media3-session          MediaSession / MediaSessionService（后台播放、通知、车机）
├── media3-cast             投屏
├── media3-transformer      转码 / 编辑（短视频剪辑用）
├── media3-effect           音视频特效
└── media3-decoder-ffmpeg   软解兜底（不在默认包里）
```

所以面试答"关系"就三句话：

1. **不是两个无关框架**，ExoPlayer 是 Media3 的一个子模块；
2. Media3 除了播放器还包含了会话、UI、缓存、转码、投屏这些**上层能力**；
3. 迁移时老代码的包名从 `com.google.android.exoplayer2.*` 换成 `androidx.media3.*`，`SimpleExoPlayer` 被废弃、统一叫 `ExoPlayer`，`MediaSessionCompat` 换成 `media3-session` 的 `MediaSession`。

## 二、原理：一个播放器内部是怎么跑起来的

### 2.1 分层架构（自上而下）

```
你的代码
   ↓  Player 接口（play / pause / seek / setMediaItem）
ExoPlayerImpl —— 对外门面，所有操作 post 到播放线程
   ↓
ExoPlayerImplInternal —— 播放线程上的状态机（唯一调度中心）
   ↓
MediaSource（内容层）→ MediaPeriod / SampleStream（时间轴切片）
   ↓
Renderer（渲染层）：视频 / 音频 / 字幕 / 元数据
   ↓
MediaCodec（硬解码器）+ Surface（渲染目标）
   ↓
DataSource（数据源层）：HTTP / 文件 / 缓存 → Extractor（容器解析）
```

### 2.2 四个必须记住的概念

**① Timeline 与 MediaItem（时间轴模型）**

ExoPlayer 把所有内容抽象成一条 **Timeline**：

- **Window**：一次可播的内容单元（一个 MediaItem 就是一个 Window）；
- **Period**：Window 内部因为广告插播等被切成的时间段；
- 位置由 `(windowIndex, positionMs)` 表示，对外就是你熟悉的 `currentMediaItemIndex` + `currentPosition`。

所以"播放列表"本质就是往 Timeline 里塞多个 Window，`repeatMode` / `shuffleModeEnabled` 是对 Timeline 的取模和洗牌——**不是自己写一个"播完自动播下一个"的监听**。

**② MediaSource 家族（谁来提供内容）**

| 类型 | 用途 |
| --- | --- |
| `ProgressiveMediaSource` | 普通 MP4/FLV 等整文件（最常用） |
| `HlsMediaSource` / `DashMediaSource` / `SsMediaSource` | HLS / DASH / SmoothStreaming 流媒体 |
| `MergingMediaSource` | 合并音视频分离流、挂外部字幕 |
| `ClippingMediaSource` | 只播某一段（试看、片段截取） |
| `ConcatenatingMediaSource` | 拼接多段（**已废弃**，改用 MediaItem 列表） |

Media3 里推荐的写法是 `player.setMediaItems(listOf(...))`，而不是自己拼 ConcatenatingMediaSource。

**③ Renderer + TrackSelector（谁来解码和渲染）**

- Renderer 分四类：`MediaCodecVideoRenderer`、`MediaCodecAudioRenderer`、`TextRenderer`（字幕）、`MetadataRenderer`；
- 从容器解析出的每条轨道是 `Format`，同类型多个 Format 组成 `TrackGroup`，选择哪一条由 **`TrackSelector`** 决定，默认 `DefaultTrackSelector`（还记得它的 `setTrackSelectionParameters` 是**累积式**的坑，后面常见问题里说）；
- 清晰度切换、多语言音轨、字幕开关，本质都是改 TrackSelectionParameters 后重新选轨，**不重新建播放器**。

**④ 缓冲与 LoadControl（什么时候算"可以播了"）**

`DefaultLoadControl` 的默认值是：`minBufferMs = 50s`、`maxBufferMs = 50s`、`bufferForPlaybackMs = 2500ms`、`bufferForPlaybackAfterRebufferMs = 5000ms`。

翻译一下：**能起播的门槛是缓冲够 2.5 秒**（卡顿后重新起播要 5 秒），但缓冲池会一直灌到 50 秒。

### 2.3 线程模型（面试高频）

- **只有一个播放线程**：所有播放逻辑跑在 `ExoPlayer:Playback` 这个 HandlerThread 上。你在主线程调 `play()` 也是 post 过去执行，所以 API 本身是线程安全的——**但这不代表你可以随意跨线程改 View**。
- **Renderer 有自己的解码/渲染线程**：播放线程把样本交给 Renderer，Renderer 内部再走 MediaCodec 的异步或同步模式。
- **视频渲染路径**：`Surface` → `MediaCodec` 输出 → SurfaceFlinger 合成；`SurfaceView` 性能好但层级（Z-order）受限，`TextureView` 可做动画/变换但多一次拷贝。
- **音频路径**：`AudioTrack`（低版本/低延迟场景会走 AAudio 或 offload）。

### 2.4 状态机：`playbackState` 和 `playWhenReady` 是两个维度

| playbackState | 含义 |
| --- | --- |
| `STATE_IDLE` | 初始态，或 `stop()` / `release()` 之后 |
| `STATE_BUFFERING` | 正在缓冲，位置基本不动 |
| `STATE_READY` | 可以立即播放（**不代表首帧已经上屏**） |
| `STATE_ENDED` | 列表播完，且 `repeatMode = OFF` |

`playWhenReady` 表示"我想不想播"（意愿），`playbackState` 表示"能不能播"（能力），两者正交。真正的判断公式是：

```kotlin
isPlaying = playWhenReady &&
            playbackState == STATE_READY &&
            playbackSuppressionReason == PLAYBACK_SUPPRESSION_REASON_NONE
```

第三个条件常被漏掉——**瞬时音频焦点丢失**（比如来了一条语音消息)时 `playbackState` 仍是 READY、`playWhenReady` 仍是 true，但实际没出声，这时 `isPlaying` 是 false，UI 该显示"暂停状态"。这就是很多"按钮图标和实际播放状态不一致"bug 的根因。

## 三、使用场景

| 场景 | 关键组件 / 做法 |
| --- | --- |
| **短视频 Feed 自动播放** | 单个 ExoPlayer + 在可见项之间迁移 `Surface`（不是每个 item 建播放器） |
| 长视频 / 课程播放 | `PlayerView`（或 Compose 里 `AndroidView { PlayerView }`）+ 手势控制 |
| 播客 / 音乐后台播放 | `media3-session` 的 `MediaSessionService` + 媒体通知 + 锁屏/蓝牙控制 |
| 直播 / 低延迟 | HLS/DASH + `setLiveConfiguration` + 调 `targetOffsetMs`（LL-HLS 可到 2~5s） |
| 视频广告 | `media3-exoplayer-ima` |
| 离线下载 | `DownloadManager` + `DownloadService`（前台服务）+ `SimpleCache` |
| 本地缓存（在线播放但复用已下分片） | `SimpleCache` + `CacheDataSource` + 自定义 `CacheKeyFactory` |
| 短视频剪辑 / 转码 | `media3-transformer`（裁剪、拼接、加滤镜） |
| 投屏 | `media3-cast` |
| 车机 / 多端控制 | `MediaSessionService` + `MediaController`（跨进程控制） |
| 多音轨 / 字幕 / 清晰度切换 | `TrackSelectionParameters`（不重建播放器） |

## 四、常用 API

### 4.1 最简播放

```kotlin
val player = ExoPlayer.Builder(context).build()

player.setMediaItem(MediaItem.fromUri(url))
player.setAudioAttributes(
    AudioAttributes.Builder()
        .setUsage(C.USAGE_MEDIA)
        .setContentType(C.AUDIO_CONTENT_TYPE_MOVIE)
        .build(),
    /* handleAudioFocus = */ true          // 交给 ExoPlayer 处理音频焦点
)
player.playWhenReady = true                 // 意愿：想播
player.prepare()                            // 开始缓冲
// 用完必须 release
player.release()
```

### 4.2 常用构建配置

| 方法 | 作用 |
| --- | --- |
| `setLoadControl(DefaultLoadControl.Builder()...)` | 调缓冲参数（起播速度 ↔ 卡顿的取舍） |
| `setTrackSelector(DefaultTrackSelector(context))` | 码率/清晰度/字幕选择策略 |
| `setMediaSourceFactory(...)` | 指定 HLS/DASH、DataSource、缓存 |
| `setRenderersFactory(...)` | 自定义解码器选择、软解兜底 |
| `setBandwidthMeter(...)` | 带宽估计（ABR 的输入） |
| `setHandleAudioBecomingNoisy(true)` | 耳机拔出自动暂停 |
| `setWakeMode(C.WAKE_MODE_NETWORK)` | 后台播放时保持网络（配 MediaSession 用） |

### 4.3 状态查询（UI 全靠这些）

```kotlin
player.playbackState          // IDLE / BUFFERING / READY / ENDED
player.playWhenReady          // 意愿
player.isPlaying              // 实际是否在播（含 suppression 判断）
player.currentPosition        // 当前进度（毫秒）
player.duration               // 总时长（直播为 C.TIME_UNSET）
player.bufferedPosition       // 已缓冲到的位置
player.bufferedPercentage     // 缓冲百分比（做进度条用）
player.currentMediaItemIndex  // 当前是列表第几条
player.isLoading              // 是否正在加载（做 loading 圈用）
```

### 4.4 事件监听（`Player.Listener`）

```kotlin
player.addListener(object : Player.Listener {
    override fun onPlaybackStateChanged(state: Int) { /* 状态机变化 */ }
    override fun onIsPlayingChanged(isPlaying: Boolean) { /* 图标切换 */ }
    override fun onMediaItemTransition(item: MediaItem?, reason: Int) { /* 切集 */ }
    override fun onPlayerError(error: PlaybackException) { /* 错误码分发 */ }
    override fun onVideoSizeChanged(videoSize: VideoSize) { /* 宽高比，做自适应 */ }
    override fun onTracksChanged(tracks: Tracks) { /* 可选轨道/字幕变化 */ }
})
```

更细粒度的埋点用 `AnalyticsListener`，能拿到 `onRenderedFirstFrame`（**真正的首帧**）、`onDroppedVideoFrames`（掉帧）、`onLoadStarted` / `onLoadCompleted`（每个 HTTP 请求的耗时和字节数）、`onBandwidthEstimate`（带宽估计值）。

### 4.5 播放控制

```kotlin
player.repeatMode = Player.REPEAT_MODE_OFF / ONE / ALL
player.shuffleModeEnabled = true
player.seekTo(positionMs)
player.seekTo(index, positionMs)
player.seekToNextMediaItem() / seekToPreviousMediaItem()
player.setPlaybackParameters(PlaybackParameters(speed = 1.5f, pitch = 1.0f))  // 倍速
player.volume = 0.5f
```

### 4.6 视频画面绑定

```kotlin
player.setVideoSurfaceView(playerView.videoSurfaceView)   // PlayerView
player.setVideoSurface(surface)                           // 自己管 Surface 时
player.clearVideoSurface()                                // 解绑（Feed 复用关键）
```

### 4.7 缓存（本地播放缓存）

```kotlin
// 1) 全局唯一的 SimpleCache（同一目录只能有一个实例，否则抛异常）
val cache = SimpleCache(
    File(context.cacheDir, "media_cache"),
    LeastRecentlyUsedCacheEvictor(200L * 1024 * 1024)   // 上限 200MB
)

// 2) 把 CacheDataSource 接到数据源链上
val httpFactory = DefaultHttpDataSource.Factory()
    .setUserAgent("HideU")
val cacheFactory = CacheDataSource.Factory()
    .setCache(cache)
    .setUpstreamDataSourceFactory(httpFactory)
    .setCacheKeyFactory(CacheKeyFactory.DEFAULT)   // 默认按 URI 做 key，可自定义剔 token
    .setFlags(CacheDataSource.FLAG_IGNORE_CACHE_ON_ERROR)

val mediaSourceFactory = DefaultMediaSourceFactory(context)
    .setDataSourceFactory(cacheFactory)

val player = ExoPlayer.Builder(context)
    .setMediaSourceFactory(mediaSourceFactory)
    .build()
```

### 4.8 后台播放（MediaSession）

```kotlin
// Service 里持有唯一 player，通过 MediaSession 对外暴露控制能力
class PlaybackService : MediaSessionService() {
    private var session: MediaSession? = null

    override fun onCreate() {
        super.onCreate()
        val player = ExoPlayer.Builder(this).build()
        session = MediaSession.Builder(this, player).build()
    }

    override fun onGetSession(controllerInfo: MediaSession.ControllerInfo) = session

    override fun onDestroy() {
        session?.player?.release()
        session?.release()
        super.onDestroy()
    }
}
```

Manifest 必须声明前台服务类型（Android 14 起强制）：

```xml
<service
    android:name=".PlaybackService"
    android:foregroundServiceType="mediaPlayback"
    android:exported="true">
    <intent-filter>
        <action android:name="androidx.media3.session.MediaSessionService" />
    </intent-filter>
</service>
```

## 五、Feed 播放器复用怎么设计（Q036 的重点）

### 结论：**一个活动播放器 + Surface 迁移**，不是有界池

```kotlin
// RecyclerView 滚动停止 → 算出当前最该播的那个 item
fun playItem(holder: VideoHolder) {
    // 1) 先把画面从旧 holder 上摘下来（避免旧 View 被复用时显示错内容）
    player.clearVideoSurface()
    // 2) 内容变了才换媒体项；同一条继续播，不重头开始
    if (currentContentId != holder.contentId) {
        currentContentId = holder.contentId
        player.setMediaItem(holder.mediaItem, savedPositionOf(holder.contentId))
        player.prepare()
    }
    // 3) 绑定到新的 Surface
    player.setVideoSurfaceView(holder.playerView.videoSurfaceView)
    // 4) 恢复用户意图（用户手动暂停过的，不要自动播）
    player.playWhenReady = holder.shouldAutoPlay && !userPaused
}
```

**四条设计原则**（对应答题要点）：

1. **播放状态按内容 ID 保存**，不按 position 保存。position 会变，contentId 不会。切回来时 `setMediaItem(item, /* resetPosition= */ false)` 或手动 `seekTo` 恢复进度。
2. **画面绑定与播放器生命周期分离**：`setVideoSurfaceView` / `clearVideoSurface` 是"贴画面"，`prepare` / `release` 是"管播放器"，两者独立。滑走时只解绑 surface + `pause()`，**不要 `release()`**。
3. **什么时候才用有界池（2 个播放器）**：需要"上一条继续播声音 + 当前条播画面"，或者要提前预缓冲下一条。代价是解码器实例翻倍，设备上 `MediaCodec` 实例数是有硬上限的，所以池的大小必须严格有界（一般 1~2），且必须有淘汰策略。
4. **绝不能每个 item 新建播放器**：一个列表滑 20 条就是 20 个 `MediaCodec`，直接 `ERROR_CODE_DECODER_INIT_FAILED` 或 OOM。

### 配套要处理的 6 件事

| 事件 | 处理 |
| --- | --- |
| 页面 `onStop` | `pause()`（保位置保缓冲），回前台按用户意愿恢复 |
| 应用退后台放音频 | 有 `MediaSessionService` 时继续播；纯 Feed 短视频视频流一律暂停 |
| 音频焦点丢失 | 让 ExoPlayer 自己处理（`setAudioAttributes(..., true)`），配 `onIsPlayingChanged` 刷 UI |
| 耳机拔出 | `setHandleAudioBecomingNoisy(true)` |
| 滚动切换目标 | 去抖（滚动停止后 150~300ms 再切）+ 用 contentId 校验异步准备结果 |
| `onDestroy` | `release()`，并清空 Listener，防泄漏 |

## 六、常见问题与排查

### 6.1 内存泄漏 / 播放器没释放
- 忘 `release()`、`Player.Listener` 用匿名内部类持有 Activity、`PlayerView` 与 Activity 互相引用。
- 排查：LeakCanary 看到 `ExoPlayerImpl` 或 `ExoPlayerImplInternal` 被持有，基本就是漏 release。
- 规矩：**播放器在哪个生命周期创建，就在同一个生命周期释放**；Activity 级播放器建议放 ViewModel + `onCleared()`。

### 6.2 切 Surface 黑屏 / 闪黑
常见三种原因：
1. 只 `clearVideoSurface()` 没给新 View `setVideoSurfaceView()`——画面没地方渲染；
2. Surface 已经销毁（`surfaceDestroyed`）却还挂着播放器，Media3 会给一个空 surface，表现为黑屏；
3. 切换瞬间调了 `stop()` 或 `setMediaItem(..., resetPosition = true)`，触发解码器 flush，首帧要重新渲染。
定位手段：监听 `AnalyticsListener.onRenderedFirstFrame`，看"黑屏期间到底是没渲染首帧，还是渲染了但被遮住"。

### 6.3 `STATE_READY` 不等于首帧已显示
`STATE_READY` 只表示"解码器和缓冲准备好了、可以放"。真实上屏要等 `onRenderedFirstFrame`。所以"首屏时间"这个指标口径必须是 `onRenderedFirstFrame`，否则测出来会比用户感知快几百毫秒。

### 6.4 首帧慢 / 频繁缓冲
拆阶段定位：DNS → TLS → 首字节 → 清单（HLS/DASH）→ 媒体分片 → 解码器初始化 → 首帧渲染。
- 服务端：转码、降码率、缩短关键帧间隔（GOP）、MP4 的 moov 前置（faststart）、CDN 命中率；
- 客户端：`bufferForPlaybackMs` 调小能加快起播但更容易卡顿；提前 `prepare()` 预热；用 `SimpleCache` 复用已下分片；ABR 让码率自适应。
- 区分卡顿来源：`onLoadCompleted` 的耗时是网络问题，`onDroppedVideoFrames` 高但网络正常是**解码/渲染性能问题**——这个区分是排查的关键分水岭。

### 6.5 解码器耗尽（多播放器场景）
每个 `ExoPlayer` 实例都会占 `MediaCodec`。列表里一屏挂 3 个播放器、快速滑动又不释放，就会 `ERROR_CODE_DECODER_INIT_FAILED`。
- 方案：单活动播放器 + Surface 迁移；或严格有界池（≤2）+ 界面不可见即释放解码器；
- 兜底：`setEnableDecoderFallback(true)` 让视频渲染器在硬解失败时回退软解（但 CPU 和功耗会上去）。

### 6.6 `TrackSelectionParameters` 是累积的黑洞
```kotlin
// 错误：直接把 overrides 清成空，丢失了自己的其他约束
player.trackSelectionParameters = TrackSelectionParameters.Builder(context)
    .setMaxVideoSizeSd().build()

// 正确：基于当前参数改
val params = player.trackSelectionParameters.buildUpon()
    .setMaxVideoSizeSd()
    .build()
player.trackSelectionParameters = params
```
`buildUpon()` 忘了用，就会出现"我在 A 页面设的限制，到 B 页面莫名其妙还在生效"。

### 6.7 缓存相关问题
- **同一目录只能有一个 `SimpleCache` 实例**，否则直接抛异常。要求全局单例，且别在 Application 里 new 多次。
- **URL 带签名 token 导致缓存永不命中**：默认 `CacheKeyFactory.DEFAULT` 用完整 URI 当 key，token 变了 key 就变了。解法是实现自定义 `CacheKeyFactory`，剔除签名参数只保留资源路径。
- **缓存 ≠ 离线下载**：缓存是"播过顺便存一份、可能被淘汰"；离线下载是"用户主动要、不能被淘汰"，要用 `DownloadManager` + `DownloadIndex` 管理，并处理 HLS/DASH 的清单与分片。
- 签名 URL 过期、跨账号内容隔离、DRM 许可证（license 不随媒体一起缓存）都要单独处理。

### 6.8 后台播放被"莫名暂停"
- Android 14+ 必须声明 `foregroundServiceType="mediaPlayback"` 并申请 `FOREGROUND_SERVICE_MEDIA_PLAYBACK` 权限；
- 用 `MediaSessionService` 而不是普通 Service，否则通知和锁屏控制拿不到；
- Android 12+ 通知有 trampoline 限制，别在通知点击里做重活。

### 6.9 其它高频杂项
- 直播的 `duration` 是 `C.TIME_UNSET`，进度条别用 0 除；
- `MediaItem` 的 `mediaId` 最好由业务自己设（用 contentId），`onMediaItemTransition` 里靠它做归因；
- Compose 里别在重组中重建 Player，用 `remember` 或 ViewModel 持有；
- 配置变化（旋转）不要重建播放器，交给 ViewModel / Service 持有。

## 七、流媒体协议：HLS / DASH / RTSP / SmoothStreaming

> 这四个经常被混着说，但它们**不在同一个层次上**：HLS / DASH / SmoothStreaming 是"切片 + 清单"的分发方案，RTSP 压根不是分发方案，而是一个控制协议。

### 7.1 先建立一句话印象

- **HLS**：苹果的方案。视频切成小片，用 `m3u8` 文本当目录，走普通 HTTP GET。**兼容性最好，延迟最大。**
- **DASH**：ISO 标准版的"HLS"。清单是 `MPD`（XML），**DRM 支持最强**，Android / Web 的主战场。
- **RTSP**：不是切片方案，而是"网络远程遥控器" + RTP 传裸流。点对点、低延迟，**摄像头专用**。
- **SmoothStreaming**：微软的老方案，清单是 ISM XML。**基本被 HLS/DASH 取代**，只在对接微软系老后端时还会遇到。

一句话解释"为什么 HLS/DASH 能上 CDN，RTSP 不行"：**前者是 HTTP 的无状态请求，任何 CDN、代理、防火墙都能缓存和转发；后者是基于 UDP 的有状态长连接会话，NAT 一挡就废。**

### 7.2 HLS（HTTP Live Streaming）

**是什么**：Apple 2009 年提出，2017 年标准化为 RFC 8216。

**原理**：
1. 服务端把视频**切片**：每片一般 2~10 秒（常见 6 秒），容器是 MPEG-TS 或 fMP4；
2. 生成**播放列表 m3u8**，分两级：
   - `master playlist`：列出多个码率的变体（Variant Stream），每个变体指向自己的 media playlist；
   - `media playlist`：按顺序列出分片 URL，配 `#EXT-X-TARGETDURATION`（单片段最大时长）、`#EXT-X-MEDIA-SEQUENCE`（分片序号）等标签；
3. 客户端流程：拉 master → 选一档码率 → 顺序 GET 分片 → 边播边按带宽切换变体；
4. **直播**的本质就是"media playlist 持续刷新"：新分片不断追加、旧分片被移除，客户端周期性重新拉 m3u8。传统 HLS 的刷新周期就是一个分片时长，**延迟大就是这么来的**；
5. 加密靠 `#EXT-X-KEY`（AES-128 / SAMPLE-AES）或 FairPlay；
6. **LL-HLS（低延迟）**：把分片再切成更小的 part（`#EXT-X-PART`），并用 HTTP/2 分块传输 + 阻塞式清单刷新，延迟能压到 2~5 秒。

**特点**：兼容性天花板（iOS/Safari/Apple TV 原生支持）；全 HTTP，CDN 极友好；延迟 6~30 秒是最大短板。

**应用场景**：点播（短视频、影视、课程）、兼容优先的直播（赛事、秀场、电商直播）、任何要覆盖 Apple 生态的地方。**做海外 App 选它最稳。**

### 7.3 DASH（MPEG-DASH）

**是什么**：MPEG 制定的 ISO/IEC 23009-1 标准（2012），定位就是"厂商中立的 HTTP 自适应流"。

**原理**：
1. 清单叫 **MPD**（Media Presentation Description，XML），四层结构：
   `Period`（时间段，例如广告前后）→ `AdaptationSet`（同类轨道集合：视频/音频/字幕）→ `Representation`（同一集合下不同码率、分辨率的具体版本）→ `Segment`（分片）；
2. 分片一般是 **fMP4/CMAF**，URL 可以用 `SegmentTemplate` 让客户端自己算（`$Number$` / `$Time$` 占位符），省掉清单里逐条列 URL；
3. 客户端流程和 HLS 类似，但**码率切换粒度更细**，可以直接在 Representation 之间切；
4. 加密用 `ContentProtection` 元素描述，**可以同时声明多种 DRM**（Widevine + PlayReady），这是它相对 HLS 的核心优势。

**特点**：开放标准、无厂商绑定；多 DRM / 多音轨 / 多字幕表达能力强；Web 靠 dash.js + MSE，Android 靠 ExoPlayer，Safari 需要 MSE 才能播；延迟与 HLS 同档（传统 4~10 秒），LL-DASH 靠 CMAF + chunked 传输能到 1~3 秒。

**应用场景**：Android/Web 为主、需要 Widevine 的点播平台；需要精细 DRM 和多语言轨道的大型平台；"一套分片两种清单"的 CDN 架构。

**面试加分点**：现在主流做法是 **CMAF 统一分片格式，同时产出 m3u8 和 mpd 两份清单**——一份存储既喂 iOS（HLS）又喂 Android/Web（DASH），CDN 成本减半。**HLS 和 DASH 早就不是非此即彼的关系了。**

#### 7.3.1 CMAF 是什么（HLS 与 DASH 的"共用底座"）

**CMAF = Common Media Application Format（通用媒体应用格式）**，MPEG 标准 ISO/IEC 23000-19。

一句话：**它是为了让 HLS 和 DASH 能共用同一份分片文件而定下的"分片格式规范"。**

**① 为什么需要它**

CMAF 出现之前，两套方案的分片格式互不兼容：

| | 旧方案 | 问题 |
| --- | --- | --- |
| HLS | MPEG-TS 分片 | 188 字节定长包，有额外封装开销；容器结构和 DASH 完全不同 |
| DASH | fMP4 分片 | 与 TS 不通用 |

后果就是：**同一部片子要转两次码、存两份文件、CDN 缓存两份。** CMAF 就是来消灭这个重复的。

**② CMAF 规定了什么**

1. **分片容器统一为 fMP4**（ISO BMFF 的 `moof` + `mdat` 片段结构），即 CMAF fragment；
2. **初始化段（init segment）** 用 m3u8 里的 `#EXT-X-MAP` 引用，MPD 里也有对应声明——这正是"两份清单指向同一批分片"的接法；
3. **每个分片必须能独立解码、独立拼接**，因此编码参数（profile/level、分辨率、声道布局）在整条流里必须保持一致；
4. **时间基准由 fMP4 内部的 `tfdt`（base media decode time）承载**，不再依赖 MPEG-TS 那套 PCR/PTS。这样 HLS 和 DASH 各自用自己的方式表达时间戳也不会错位——这是"同一份分片能被两种清单消费"的关键前提；
5. **CMAF chunk**：允许把分片再切成更小的块，服务端边生成边通过 chunked transfer 下发。这是 **LL-HLS 和 LL-DASH 的共同底座**；
6. **两种加密模式**：
   - `cenc`：AES-CTR，Widevine / PlayReady 传统走这个；
   - `cbcs`：AES-CBC，**Apple FairPlay 强制要求这个**。

**③ 最大的收益**

```
        一份转码产物（CMAF 分片存储 + 一份 CDN 缓存）
              ├── master.m3u8   → iOS / Safari
              └── manifest.mpd  → Android / Web
```

- 存储、转码、CDN 成本直接减半；
- 新增或切换客户端只需再生成一份清单，**不动媒体文件**；
- 低延迟直播有了统一的分片切片方式。

**④ 它没有解决的事（面试喜欢追这点）**

1. **清单还是各写各的**：m3u8 和 MPD 依然要分别生成，CMAF 只管分片这一层；
2. **DRM 依然是两套**：密钥体系、许可证服务器不会因为 CMAF 就统一；
3. **加密模式有冲突**：FairPlay 要 `cbcs`，部分老 Widevine 环境偏 `cenc`。当前主流做法是统一用 `cbcs`（Widevine 也支持），实在不行只能存两份加密分片；
4. **老设备兼容**：不支持 fMP4 形式 HLS 的老 iOS、老 Android、老机顶盒仍需要保留 MPEG-TS 分片兜底。

**所以准确的说法是：CMAF 让 HLS 和 DASH 的"媒体层"合并了，但"清单层和 DRM 层"仍然分开。**

### 7.4 RTSP（Real Time Streaming Protocol）

**是什么**：IETF 的 RFC 2326（1998，2.0 版为 RFC 7826）。**注意：RTSP 不是编码格式也不是封装格式，它是一个"控制协议"**——可以理解成"媒体流的 HTTP 版"或"网络的远程遥控器"。它只负责发指令，真正的数据走 **RTP**。

**原理**：
1. 客户端先发 `DESCRIBE`，服务端用 **SDP** 回复"这里有哪几路流、编码是什么、端口是多少"；
2. 客户端对每路流（视频一路、音频一路）分别 `SETUP`，协商传输方式：
   - **RTP over UDP**（默认，延迟低但丢包不管）；
   - **RTP over TCP 交织（interleaved）**（能穿 NAT，但会队头阻塞、延迟变高）；
3. `PLAY` 开始推流，服务端不停发 RTP 包，客户端自己组包、解码、渲染；
4. `PAUSE` / `TEARDOWN` 结束会话。全程**没有"清单"和"分片"的概念，所以也不存在自适应码率**（RTSP 2.0 才补了一些能力，但生态没起来）。

**特点**：延迟最低（几百毫秒到 1~2 秒），因为不需要等一整片下完；但 NAT 穿透差、CDN 几乎不支持、并发规模上不去；带宽抖动就花屏；移动端必须引库。

**应用场景**：**安防摄像头 / IPC / NVR**（绝对主场）、IPTV 机顶盒、局域网实时预览、无人机与工业视觉回传。**不要用它做面向公网的短视频分发。**

### 7.5 SmoothStreaming（MSS / MS-SSTR）

**是什么**：微软 2008 年前后为 Silverlight 推出的自适应流方案，思路和 HLS/DASH 一模一样，只是清单格式和分片命名约定是微软自己的。

**原理**：
1. 清单叫 **Manifest（`.ism/manifest`）**，结构：`SmoothStreamingMedia` → `StreamIndex`（音频/视频/字幕）→ `QualityLevel`（不同码率）→ `c`（chunk：每片时长和数量）；
2. 分片是 fMP4（`.ismv` / `.isma`），URL 由清单里的时间片推导；
3. 客户端流程与 HLS/DASH 相同：选 QualityLevel + 拉分片。

**特点**：微软生态遗留（Windows / Xbox / 老版 Azure Media Services）；DRM 走 PlayReady；ExoPlayer 至今仍保留 `SsMediaSource` 支持。**新项目不会主动选它。**

**应用场景**：对接 Azure Media Services 老账号、Xbox/UWP 生态、历史遗留播放链路。

### 7.6 横向对比表

| 维度 | HLS | DASH | RTSP | SmoothStreaming |
| --- | --- | --- | --- | --- |
| 主导方 | Apple | MPEG / ISO | IETF | Microsoft |
| 清单文件 | M3U8（文本） | MPD（XML） | SDP（DESCRIBE 返回） | Manifest（ISM XML） |
| 分片容器 | MPEG-TS / fMP4 | fMP4 / MPEG-TS | 无分片（RTP 打包） | fMP4（.ismv/.isma） |
| 承载传输 | HTTP(S) | HTTP(S) | RTP over UDP / TCP | HTTP(S) |
| 典型延迟 | 6~30s；LL-HLS 2~5s | 4~10s；LL-DASH 1~3s | 0.2~2s | 与 HLS 同档 |
| 自适应码率 | 支持（Variant） | 支持（Representation） | **不支持** | 支持（QualityLevel） |
| DRM | AES-128 / FairPlay | Widevine / PlayReady / ClearKey | 一般无（靠信令层） | PlayReady |
| CDN 友好度 | 极高 | 极高 | 差 | 高 |
| 平台支持 | iOS/Safari 原生；Android 需 ExoPlayer | Web/Android 好；Safari 需 MSE | 需专门库 | 需 ExoPlayer 等 |
| ExoPlayer 模块 | `media3-exoplayer-hls` | `media3-exoplayer-dash` | `media3-exoplayer-rtsp` | `media3-exoplayer-smoothstreaming` |
| 选它的理由 | 兼容性、Apple 生态 | 开放标准、多 DRM | 低延迟、点对点实时 | 兼容遗留后端 |

### 7.7 延迟梯度与选型

```
< 1s 超低延迟    WebRTC / SRT  ──  连麦、互动直播、远程控制
0.2~2s 实时      RTSP          ──  摄像头、局域网预览、IPTV
1~5s  低延迟     LL-HLS / LL-DASH（CMAF + chunked）── 带货直播、赛事互动
6~30s 常规       HLS / DASH    ──  点播、普通直播、大规模分发
```

选型口诀：**要规模就 HTTP（HLS/DASH），要延迟就 RTSP/WebRTC，要 Apple 就 HLS，要 DRM 和多轨就 DASH。**

### 7.8 ExoPlayer 里怎么落地

`DefaultMediaSourceFactory` 会按 URI 后缀和 MIME **自动挑 MediaSource**：`.m3u8` → HLS、`.mpd` → DASH、`.ism/.isml` → SmoothStreaming、`rtsp://` → RTSP。所以通常只要把对应依赖加上就行：

```kotlin
dependencies {
    implementation("androidx.media3:media3-exoplayer:1.x.x")
    implementation("androidx.media3:media3-exoplayer-hls:1.x.x")           // 不加就播不了 m3u8
    implementation("androidx.media3:media3-exoplayer-dash:1.x.x")          // 不加就播不了 mpd
    implementation("androidx.media3:media3-exoplayer-rtsp:1.x.x")          // 不加就播不了 rtsp://
    implementation("androidx.media3:media3-exoplayer-smoothstreaming:1.x.x")
}
```

URL 没有后缀时手动指定 MIME：

```kotlin
val item = MediaItem.Builder()
    .setUri(url)
    .setMimeType(MimeTypes.APPLICATION_M3U8)   // HLS；DASH 用 APPLICATION_MPD
    .build()
```

### 7.9 面试追问速答

- **HLS 和 DASH 的本质区别是什么？** 结构上都是"HTTP 分片 + 清单 + 自适应码率"，区别在**清单格式（m3u8 vs XML）、分片容器约定、DRM 表达能力和生态归属**。用 CMAF 统一分片后，两者只差清单这一层。
- **HLS 延迟为什么大？** 它建立在"分片完整才能写入播放列表"这个前提上，客户端还要周期性刷新清单，延迟天然等于"分片时长 × 2~3"。LL-HLS 用部分分片 + chunked 传输打破了这个前提。
- **RTSP 为什么不能做 CDN 分发？** 它是基于 UDP 的有状态长连接会话，CDN 没法像缓存 HTTP 分片那样中转；NAT 还会挡住服务端主动发来的 RTP 包。
- **RTSP 拉流能转成 HLS 吗？** 能，这就是"视频网关 / 转封装服务"在干的事：从摄像头拉 RTSP，实时切片并生成 m3u8 再对外分发。代价是延迟从 1 秒涨到 5~10 秒。
- **ExoPlayer 播不了 m3u8 通常是为什么？** 没加 `media3-exoplayer-hls` 依赖；或者 URL 没有后缀、也没指定 MIME，导致它按 Progressive 去解析，报容器解析错误。

## 八、WebSocket：原理 / 应用场景 / 与第三方 IM、Push 的关系

> 本章是消息链路的补充（对应岗位要求里的"WebSocket / 第三方 IM / Push 实时推送机制"）。核心结论先给：**WebSocket 是"通道"，第三方 IM 是"通道 + 业务能力"的封装，Push 是"进程不在时的唤醒通道"，三者是分层协作关系，不是三选一。**

### 8.1 WebSocket 是什么

HTML5 标准之一，正式规范是 **RFC 6455**。一句话：**在一条 TCP 连接上，用一次 HTTP 握手"借用"通道，之后双方随时可以互相发数据（全双工）。**

它解决的是 HTTP 的根本限制：HTTP 是"请求-响应"模型，服务端**没有办法主动**给客户端发消息。想要"服务端主动推"，老办法只有轮询（短轮询/长轮询），代价是延迟高、开销大。

| 方案 | 延迟 | 每次开销 | 方向 |
| --- | --- | --- | --- |
| 短轮询 | 一个轮询周期 | 完整 HTTP 头 + 新建连接 | 单向拉 |
| 长轮询（Comet） | 中 | 每次响应后要重建连接 | 伪推送（服务端→客户端） |
| SSE | 低 | 一次连接、文本流 | 单向推，不支持双向 / 二进制 |
| **WebSocket** | 最低 | **帧头仅 2~14 字节** | **全双工，文本 + 二进制** |
| WebRTC DataChannel | 最低 | 基于 SCTP/UDP | 全双工，可 P2P |

### 8.2 原理

**① 握手：借 HTTP 的"协议升级"机制**

客户端发一个看起来像普通 HTTP 的请求：

```
GET /ws HTTP/1.1
Host: im.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

服务端返回 **101 Switching Protocols**：

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

- `Sec-WebSocket-Key` 是客户端生成的 16 字节随机数（base64）；
- `Sec-WebSocket-Accept` = 把 key 拼接固定 GUID `258EAFA5-E914-47DA-95CA-C5AB0DC85B11`，做 **SHA-1 再 base64**。校验它就是为了确认"对面真的懂 WebSocket"，而不是某个代理把普通 HTTP 响应错当成升级成功（防止代理缓存投毒）；
- **为什么用 HTTP 握手？** 为了活下来。HTTP 的 80/443 能穿几乎所有代理、NAT 和防火墙——这是 WebSocket 相比"自研裸 TCP 协议"最大的落地优势；
- 101 之后，**这条 TCP 连接就不再是 HTTP 了**，双方改按 WebSocket 帧协议通信。

**② 数据帧格式（2~14 字节头 + 载荷）**

| 字段 | 作用 |
| --- | --- |
| FIN | 是否为一个消息的最后一帧（大消息会被分成多个帧） |
| Opcode | `0x1` 文本 / `0x2` 二进制 / `0x8` Close / `0x9` Ping / `0xA` Pong |
| Mask | **客户端发往服务端的帧必须掩码**（用 4 字节 Masking-Key 异或），服务端发往客户端的帧不允许掩码 |
| Payload length | 7 位（≤125）、7+16 位（≤64KB）、7+64 位（大载荷） |

两点值得记：**帧头极小**（对比 HTTP 每次几百字节的头），所以高频小消息场景下 WebSocket 省流量又省电；**客户端必须掩码**是历史包袱（防止早期代理设备把帧内容当 HTTP 缓存下来投毒）。

**③ 保活与断线（工程上真正的难点）**

- **为什么要心跳**：NAT / 运营商网关会静默回收"长时间无数据"的连接（常见 30 秒 ~ 5 分钟），两侧都不知道连接已经死了（TCP 半开）。所以要靠心跳探测；
- **两种心跳**：协议层 Ping/Pong（OkHttp 的 `pingInterval` 会自动发 Ping、收到 Pong 才认为连接存活），或应用层自定义心跳（`{"type":"ping"}`），后者能穿越不支持协议层 Ping 的网关、也能顺带做业务状态上报；
- **重连策略**：指数退避 + 随机抖动（例如 1s、2s、4s、8s ±30%），避免服务端抖动时全量客户端同时重连把服务打挂；
- **断线期间的消息会丢**：WebSocket 不保证"离线消息"。所以必须有兜底——**带序号的 ack 机制 + 重连后按本地最大 seq 向服务端 HTTP 补拉**，再配合 `messageId` 本地去重。这是"长连接 + 拉取对账"的经典组合。

### 8.3 Android 上怎么用（OkHttp）

```kotlin
val client = OkHttpClient.Builder()
    .pingInterval(30, TimeUnit.SECONDS)      // 协议层 Ping，自动探测死连接
    .build()

val request = Request.Builder()
    .url("wss://im.example.com/ws")          // 生产必须用 wss（TLS）
    .addHeader("Authorization", "Bearer $token")
    .build()

val ws = client.newWebSocket(request, object : WebSocketListener() {
    override fun onOpen(webSocket: WebSocket, response: Response) {
        // 连接建立：发鉴权帧 / 拉离线消息
    }

    override fun onMessage(webSocket: WebSocket, text: String) {
        // 收到消息：解析 → 本地落库（带 seq）→ 用 Flow 推给 UI → 回 ack
    }

    override fun onFailure(webSocket: WebSocket, t: Throwable, response: Response?) {
        // 断线：指数退避重连 + 按本地最大 seq 触发 HTTP 补拉
    }

    override fun onClosing(webSocket: WebSocket, code: Int, reason: String) {
        webSocket.close(1000, null)
    }
})

ws.send(authFrameJson)     // 二进制用 send(ByteString)
ws.close(1000, "bye")      // code 1000 = 正常关闭
```

工程注意点：

| 事项 | 做法 |
| --- | --- |
| 连接持有者 | 放 ViewModel 或前台 Service（IM 场景），**不要绑 Activity**，否则旋转/返回就断连 |
| 重连时机 | 主动 `close()` 后不重连；`onFailure` 才重连；网络恢复（`ConnectivityManager` 回调）时立即重试一次 |
| 进程被杀 | 长连接一定断，只能靠 **Push 唤醒** |
| 前后台切换 | 退后台可保留连接一段时间做"兜底在线"，但不要指望长期存活——这是 Push 存在的意义 |
| 多端/多进程 | 全 App 只维护一条连接，用单例 + 引用计数控制（页面进出不重连） |

### 8.4 应用场景

| 场景 | 为什么用它 |
| --- | --- |
| IM 聊天（单聊/群聊） | 消息必须"秒达"，且要双向（发消息 + 收消息 + 已读回执） |
| Feed / 社区实时互动 | 点赞数、评论数、在线人数实时刷新，避免轮询打爆接口 |
| 直播弹幕 / 互动信令 | 高频小包，帧头开销小最关键；连麦信令也走这里（媒体流另走 WebRTC/SRT） |
| 协同编辑 / 实时文档 | 需要低延迟的双向增量同步 |
| 行情 / 体育比分 / 游戏大厅 | 服务端高频主动推送 |
| 游戏对战内通信 | 小包高频，可容忍 UDP 场景会换 KCP/QUIC 方案 |
| App 内客服 / 工单 | 双向会话，长连接体验优于轮询 |

**不适合它的场景**：一次性数据拉取（用普通 HTTP 更好，能利用 CDN 和缓存）；需要多端可靠投递但客户端可能被杀的离线通知（用 Push）；大文件传输（HTTP 分块/断点续传更合适）。

### 8.5 三者的关系（WebSocket / 第三方 IM / Push）

**它们是分层的，不是替代关系：**

```
业务层：消息列表、会话、已读回执、撤回、敏感词过滤
   ↑
IM 能力层：自研 or 第三方 SDK（会话管理 / 群组 / 离线消息 / 多端同步 / 漫游）
   ↑
长连接层：WebSocket / 自研 TCP 协议   ← 实时通道
   ↑
传输层：TCP / TLS
- - - - - - 进程被杀、被冻结、网络切换的分界线 - - - -
唤醒层：Push（海外 FCM / 国内厂商通道）→ 把 App 拉起来 → 重建长连接
   ↑
拉取层：HTTP 拉历史消息 / 增量同步（最终对账）
```

一句话对应：

- **WebSocket = 通信能力**（一个协议，自己得写重连、心跳、鉴权、可靠性）；
- **第三方 IM = 通信能力 + 业务能力**（环信、融云、腾讯云 IM 这类，内部就是自研长连接，帮你把会话/群组/离线消息/多端同步都做完，代价是费用和定制受限）；
- **Push = 唤醒能力**（FCM / APNs / 厂商通道，**进程不在时唯一能把 App 叫起来的手段**）。

**为什么要三个一起上**，而不是只用 Push：

| 只靠 Push 的问题 | 长连接怎么补 |
| --- | --- |
| 延迟不可控（秒级到分钟级，弱网/省电模式更慢） | 长连接是毫秒到百毫秒级 |
| 有配额/频率限制，不能当高频通道 | 长连接无此限制 |
| 不保证顺序、不保证不丢 | 长连接配 seq + ack 可做到有序 |
| 国内厂商通道各家 API 不同、后台展示限制多 | 长连接消息自己渲染 |

**反过来，只靠长连接也不行**：App 被杀、被系统冻结、用户清理后台后连接必断，这时只能靠 Push 唤醒或直接由 Push 载荷展示通知。

**标准的工程组合（面试答这个就够）**：

1. **前台**：WebSocket 长连接负责实时收发，消息带 `seq`/`messageId`，本地落库（Room）后由 Flow 推 UI；
2. **切后台/进程被杀**：连接断开，服务端把消息转成 Push 下发（FCM / 厂商通道）；
3. **用户点开通知或回到前台**：重建长连接，**按本地最大 seq 向服务端 HTTP 补拉增量**，用 `messageId` 去重；
4. **结果**：长连接管实时，Push 管唤醒兜底，HTTP 拉取管最终一致——**"推送是提示，拉取才是真相"**。

针对这个岗位的海外场景补充：**FCM 是免费的、Google Play 生态必备**，所以海外 App 的 Push 通道不存在国内那种"厂商通道各自为政"的复杂度；但弱网国家（印度、印尼、巴西等）FCM 到达率和延迟都不理想，所以 IM 场景**必须保留前台长连接 + 重连补拉**，不能指望 FCM。

### 8.6 与本文档音视频部分的关系（一句话区分）

**WebSocket 管"信令"，SRS/RTMP/WebRTC 管"媒体流"。** 直播里的弹幕、连麦协商、房间状态、礼物动效走 WebSocket；真正的音视频帧走 RTMP/SRT 上行、WebRTC 或 LL-HLS/LL-DASH 下行。两者分离的原因：媒体流要的是低延迟和抗丢包，信令要的是可靠和有序，需求不同，硬塞一条通道会互相拖累。

### 8.7 面试追问速答

- **WebSocket 握手为什么借用 HTTP？** 为了穿透代理、NAT 和防火墙，80/443 是"基础设施默认放行的端口"；纯自研 TCP 协议在弱网和多网络环境下部署成本高得多。
- **`Sec-WebSocket-Accept` 怎么算，为什么需要它？** 客户端 key 拼固定 GUID → SHA-1 → base64。校验它是为了确认对端真的支持 WebSocket，避免中间代理把普通 HTTP 响应误当升级成功（防投毒）。
- **101 状态码是什么？** Switching Protocols，表示协议切换成功，此后这条连接不再是 HTTP。
- **为什么客户端必须给帧做掩码？** 防止早期代理设备把帧内容误当 HTTP 请求缓存并投毒；服务端到客户端的帧则不允许掩码。
- **WebSocket 和 SSE 怎么选？** 需要双向交互、二进制、自定义心跳选 WebSocket；只是服务端单向推文本流（如行情、日志），SSE 更轻、能自动重连、走标准 HTTP 更好运维。
- **怎么发现连接已经死了？** 靠心跳。TCP 半开时本端完全无感知，所以必须有协议层 Ping/Pong（OkHttp `pingInterval`）或应用层心跳，超时未收到 Pong 就主动断开重连。
- **断线期间的消息怎么办？** 长连接不保证离线消息。用"带序号 ack + 重连后按最大 seq HTTP 补拉 + messageId 去重"三板斧，保证不丢不重。
- **有了 WebSocket 还需要 FCM 吗？** 需要。长连接只在进程活着时有效，进程被杀后唯一能唤醒 App 的就是 Push；FCM 负责唤醒，真正的消息内容回到前台后从服务端拉取。

## 九、面试口述版（30 秒）

> "Media3 是 Jetpack 的媒体套件，ExoPlayer 是里面负责播放的那个实现，两者是包含关系不是并列关系。ExoPlayer 内部是分层架构：Player 接口 → 播放线程上的状态机 → MediaSource 提供内容 → Renderer 走 MediaCodec 解码 → DataSource 取数据。它只有一个播放线程，所以 API 线程安全，但状态要分 `playbackState` 和 `playWhenReady` 两个维度看，`isPlaying` 还要额外判断播放抑制原因。Feed 场景的标准做法是一个活动播放器加 Surface 迁移，播放状态按内容 ID 保存，页面不可见只解绑 surface 和 pause、不 release；只有需要'边播上一条边预缓冲下一条'时才用严格有界的两实例池，因为 MediaCodec 实例在设备上是稀缺资源。"
