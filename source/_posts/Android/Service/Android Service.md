---
title: Android Service：原理 / 使用场景 / 常用 API / 常见坑
date: 2026-10-21 00:26:34
---

# Android Service：原理 / 使用场景 / 常用 API / 常见坑

> 一句话定位：Service 是四大组件里"负责在后台干活"的那一个，但它**跑在所在进程的主线程上，本身不是线程、也不会自动开线程**；现代 Android 对后台启动和存活的限制越来越紧，能用 WorkManager / 协程的地方就别硬用 Service。
> 版本提示：后台限制相关内容以 Android 6~14 的公开行为为准，不同厂商 ROM 还会额外加自己的省电策略。

---

## 一、Service 是什么（原理）

### 1.1 一句话结论

Service 是一个**没有 UI、可被系统在后台保持运行的组件**，生命周期由 `startService` / `bindService` / `stopSelf` / `stopService` / `unbindService` 驱动。

最常见的误解有两个：

1. **以为 Service 就是"后台线程"**——错。Service 默认运行在**进程主线程**，做耗时操作照样 ANR，要自己开子线程/线程池/协程；
2. **以为 Service 会一直活着**——错。后台服务在 Android 8 以后有严格限制，会被 Doze 和厂商 ROM 冻结、杀掉。

### 1.2 三种形态

| 形态 | 怎么启动 | 谁决定它活多久 | 典型用途 |
| --- | --- | --- | --- |
| **started（启动服务）** | `startService()` / `startForegroundService()` | 启动后与启动者解耦，靠 `stopSelf()` / `stopService()` 结束 | 一次性的、可持续的后台任务 |
| **bound（绑定服务）** | `bindService()` + `ServiceConnection` | **跟随绑定者**：所有绑定者 `unbindService` 后，系统销毁它 | C/S 结构，多个组件共享同一个 Service 实例 |
| **foreground（前台服务）** | `startForegroundService()` → 5 秒内 `startForeground()` | 同 started，但必须有常驻通知，进程优先级最高 | 音乐播放、导航、录音、下载等用户可感知的任务 |

三种形态可以**组合**：既 `startService` 又被别人 `bindService`（音乐 App 常见）。组合时必须**同时满足**"已 `stopSelf`"和"没有绑定者"两个条件，系统才会真正销毁它。

### 1.3 生命周期回调

```
started 路径：  onCreate() → onStartCommand() → …… → onDestroy()
bound   路径：  onCreate() → onBind() → [绑定中] → onUnbind() → onDestroy()
组合    路径：  onCreate() → onStartCommand() → onBind() → onUnbind() →（再 bind）→ onRebind() → onDestroy()
```

| 回调 | 说明 |
| --- | --- |
| `onCreate()` | 只在 Service **第一次创建**时调用一次，做初始化 |
| `onStartCommand(intent, flags, startId)` | **每次** `startService` / `startForegroundService` 都会调用一次；`startId` 递增，可用于 `stopSelf(startId)` 精确停止 |
| `onBind(intent)` | 返回 `IBinder`，只在**首次**绑定时调用 |
| `onUnbind(intent)` | 所有绑定者解绑后调用；返回 `true` 表示允许后续重新绑定时走 `onRebind` |
| `onRebind(intent)` | 之前 `onUnbind` 返回过 `true`、且又有新绑定者时调用 |
| `onDestroy()` | 释放资源：取消协程/线程、注销监听、关闭连接 |

**关键区别**：`onCreate` 只执行一次，`onStartCommand` 会执行多次。

### 1.4 onStartCommand 的返回值：能不能"复活"的真相

| 返回值 | 进程被回收后系统是否重建 | 是否重投递 intent |
| --- | --- | --- |
| `START_STICKY` | 会**尝试**重建（重建时 intent 为 `null`） | 不重投递 |
| `START_NOT_STICKY` | 不重建 | 不重投递 |
| `START_REDELIVER_INTENT` | 会**尝试**重建 | 重投递最后（及未处理完）的 intent |
| `START_STICKY_COMPATIBILITY` | 兼容旧行为的 STICKY，**不保证**会给 `null` intent | 不重投递 |

**真相（面试重点）**：

1. 返回值只是"**系统愿意的话会重建**"的**意愿**，不是保证；
2. 现实里一堆限制会打断它：Android 6 起的 **Doze / App Standby** 会冻结后台应用；Android 8 起的**后台服务限制**让"后台启动服务"本身受限；国内厂商 ROM 还有自己的省电杀后台策略；
3. 所以**不能依赖 `START_STICKY` 做保活/拉活**。要"任务最终一定被执行"，用 **WorkManager**（可持久化、满足条件自动重试、重启后恢复）；要"用户可感知地持续运行"，用**前台服务**。

> 使用 `START_STICKY` / `START_REDELIVER_INTENT` 时，`onStartCommand` 里的 `intent` 可能为 `null`（重建场景），必须判空，否则直接崩溃。

### 1.5 进程优先级

系统按"进程里最高优先级的组件"决定 `oom_adj`，越靠前越不容易被回收：

```
前台进程 (Foreground)   ← 有前台服务、或用户正在交互
  > 可见进程 (Visible)   ← onPause 但仍可见
    > 服务进程 (Service) ← startService 起来的普通服务
      > 缓存进程 (Cached) ← 后台没有组件的进程
```

把一个普通服务改成**前台服务**，本质就是**把自己的进程从"服务进程"抬到"前台进程"**，从而大幅降低被杀概率——这就是音乐/导航类必须用前台服务的根本原因。

### 1.6 后台限制演进（重点表）

| 版本 | API | 关键变化 |
| --- | --- | --- |
| Android 6 | 23 | **Doze** 与 **App Standby**：设备静止时冻结后台网络与任务 |
| Android 7 | 24 | 后台优化：Doze on the Go；收紧部分隐式广播（如 `CONNECTIVITY_ACTION`） |
| Android 8 | 26 | **后台服务限制**：应用在后台时 `startService()` 直接抛 `IllegalStateException`，必须用 `startForegroundService()` 并在 **5 秒内 `startForeground()`**；**隐式广播**大多不能再静态注册在清单里；后台定位频率受限 |
| Android 9 | 28 | **后台启动 Activity** 受限；前台服务需要 `FOREGROUND_SERVICE` 权限；后台访问麦克风/传感器受限 |
| Android 10 | 29 | 后台启动 Activity **进一步收紧**（需要用户可见窗口等条件）；后台定位需要 `ACCESS_BACKGROUND_LOCATION` |
| Android 12 | 31 | **后台启动前台服务受限**（从后台启动会抛 `ForegroundServiceStartNotAllowedException`，只有特定豁免场景可以）；**通知 trampoline 限制**（点通知不能"先起一个广播/服务再跳 Activity"）；使用精确闹钟需要 `SCHEDULE_EXACT_ALARM` |
| Android 13 | 33 | 用户可从**任务管理器直接停止前台服务**（会停掉整个应用）；前台服务**通知可被用户划掉**；发通知需要 `POST_NOTIFICATIONS` 运行时权限 |
| Android 14 | 34 | 前台服务**必须声明 `android:foregroundServiceType`** 并申请对应的 `FOREGROUND_SERVICE_*` 权限；**类型必须与实际行为匹配**，否则抛 `SecurityException`；新增 **`shortService`** 类型（短任务，不能长期占用） |

> 版本细节记不清时，就说"这一版又进一步收紧了对后台启动和前台服务类型的约束"，**别硬编造具体行为**。

## 二、使用场景

### 2.1 什么时候该用 Service

| 场景 | 现代推荐方案 | 说明 |
| --- | --- | --- |
| 音乐 / 播客后台播放 | `MediaSessionService`（media3-session）+ `foregroundServiceType="mediaPlayback"` | 前台服务 + 媒体通知 + 锁屏/蓝牙控制 |
| 文件下载 | `DownloadManager`（用户主动下载）或前台服务（要自定义下载逻辑） | 用户可感知，通知带进度 |
| 录音 / 录屏 | 前台服务 + `microphone` / `mediaProjection` 类型 | 必须前台，否则新版本会被限制 |
| 定位跟踪（导航、运动、打卡） | 前台服务 + `location` 类型 + `ACCESS_BACKGROUND_LOCATION` | 用户可感知的持续定位 |
| 数据同步 | **WorkManager 更合适**；只有"用户正在等、必须马上做"才用前台服务 | 可延迟、可重试的交给 WorkManager |
| IM 保活 / 长连接 | 前台服务 + 厂商 Push 兜底 | 别指望后台服务长期存活，Push 才是唤醒手段 |
| 短视频预加载 | 前台服务（可选）或应用内调度 | 优先级低，别给后台加负担 |

### 2.2 不适合用 Service 的场景

| 需求 | 别用 Service，用 | 原因 |
| --- | --- | --- |
| 可延迟的后台任务（上传、同步、清理） | **WorkManager** | 可持久化、满足约束（充电/联网）才跑、系统重启后恢复、自动重试 |
| 一次性短任务（几百毫秒的解析/写库） | **协程 / 线程池** | 没必要起组件，Service 生命周期管理是纯负担 |
| 定时任务 | **WorkManager**（周期性）/ **AlarmManager**（精确到点） | 后台服务定时拉起在新版本基本不可行 |
| 只是"想在后台跑点代码" | `applicationScope` 或 WorkManager | 现代限制下"裸后台服务"根本活不了 |

## 三、常用 API 与使用方法

### 3.1 前台服务完整代码

```kotlin
class SyncService : Service() {

    companion object {
        private const val CHANNEL_ID = "sync_channel"
        private const val NOTIFICATION_ID = 1001

        // 外部统一用这个方法启动，自动兼容 8.0 前后的差异
        fun start(context: Context) {
            val intent = Intent(context, SyncService::class.java)
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                context.startForegroundService(intent)   // 8.0+ 必须用这个
            } else {
                context.startService(intent)
            }
        }
    }

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private var notifyManager: NotificationManager? = null

    override fun onCreate() {
        super.onCreate()
        notifyManager = getSystemService(NotificationManager::class.java)
        ensureChannel()
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // 注意：STICKY 重建时 intent 可能为 null，别直接 intent.xxx
        // 必须在 5 秒内调用 startForeground，否则系统抛异常并杀掉进程
        startForeground(NOTIFICATION_ID, buildNotification("准备中…", 0))

        scope.launch {
            for (progress in 1..100) {
                delay(100)
                // 更新前台服务通知（同一个 NOTIFICATION_ID 覆盖）
                notifyManager?.notify(NOTIFICATION_ID, buildNotification("同步中…", progress))
            }
            // 任务结束：移除前台通知并停止服务
            stopForeground(STOP_FOREGROUND_REMOVE)
            stopSelf(startId)      // 用 startId 防止误停后来的启动请求
        }

        return START_NOT_STICKY
    }

    override fun onBind(intent: Intent?): IBinder? = null   // 纯启动式服务，不支持绑定

    override fun onDestroy() {
        scope.cancel()          // 必须取消，否则协程泄漏
        super.onDestroy()
    }

    private fun ensureChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                CHANNEL_ID,
                "后台同步",
                NotificationManager.IMPORTANCE_LOW     // 前台服务通知用 LOW，避免打扰
            ).apply { setShowBadge(false) }
            notifyManager?.createNotificationChannel(channel)
        }
    }

    private fun buildNotification(text: String, progress: Int): Notification {
        val openApp = PendingIntent.getActivity(
            this, 0, packageManager.getLaunchIntentForPackage(packageName),
            PendingIntent.FLAG_IMMUTABLE
        )
        return NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("正在同步")
            .setContentText(text)
            .setSmallIcon(R.drawable.ic_sync)
            .setOngoing(true)                       // 常驻，不可划掉（系统仍可能强制展示取消按钮）
            .setProgress(100, progress, progress == 0)
            .setContentIntent(openApp)
            .build()
    }
}
```

几个必须记住的点：

- **`startForeground` 必须在 5 秒内调用**，否则系统抛 `ForegroundServiceDidNotStartInTimeException`（属于 `RemoteServiceException` 一类）并杀掉进程；
- `stopForeground(STOP_FOREGROUND_REMOVE)`：**移除通知并退出前台**；若想"退出前台但保留通知"，用 `STOP_FOREGROUND_DETACH`；
- `PendingIntent` 在 Android 12+ 必须显式指定 `FLAG_IMMUTABLE` 或 `FLAG_MUTABLE`，否则崩溃。

### 3.2 Android 14 的 Manifest 声明

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- 基础前台服务权限（Android 9+ 必需） -->
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <!-- Android 14：按类型申请对应权限 -->
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
    <!-- 按需：通知、定位等运行时权限 -->
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />

    <application>
        <!-- 数据同步类前台服务 -->
        <service
            android:name=".SyncService"
            android:exported="false"
            android:foregroundServiceType="dataSync" />

        <!-- 播放类前台服务（Android 14 起类型必须与行为匹配） -->
        <service
            android:name=".PlaybackService"
            android:exported="true"
            android:foregroundServiceType="mediaPlayback" />

        <!-- 短任务：后台短时间内必须完成，系统到期会回调超时 -->
        <service
            android:name=".ShortTaskService"
            android:exported="false"
            android:foregroundServiceType="shortService" />
    </application>
</manifest>
```

常见 `foregroundServiceType` 与对应权限：

| 类型 | 对应权限 | 典型用途 |
| --- | --- | --- |
| `mediaPlayback` | `FOREGROUND_SERVICE_MEDIA_PLAYBACK` | 音乐/播客播放 |
| `dataSync` | `FOREGROUND_SERVICE_DATA_SYNC` | 上传/下载/同步 |
| `location` | `FOREGROUND_SERVICE_LOCATION` | 导航、定位跟踪 |
| `microphone` | `FOREGROUND_SERVICE_MICROPHONE` | 录音 |
| `camera` | `FOREGROUND_SERVICE_CAMERA` | 录像/摄像头 |
| `mediaProjection` | `FOREGROUND_SERVICE_MEDIA_PROJECTION` | 录屏/投屏 |
| `connectedDevice` | `FOREGROUND_SERVICE_CONNECTED_DEVICE` | 蓝牙/USB 外设通信 |
| `shortService` | 无需额外权限 | 后台短任务，需尽快结束 |
| `specialUse` | `FOREGROUND_SERVICE_SPECIAL_USE` | 无法归类的特殊用途（上架需说明） |

### 3.3 绑定服务（本地 Binder + ServiceConnection）

```kotlin
class PlayerService : Service() {

    inner class LocalBinder : Binder() {
        fun getService(): PlayerService = this@PlayerService
    }
    private val binder = LocalBinder()

    fun play(url: String) { /* 真正播放逻辑 */ }
    fun pause() { /* … */ }

    override fun onBind(intent: Intent?): IBinder = binder
    override fun onUnbind(intent: Intent?): Boolean = super.onUnbind(intent)
}
```

```kotlin
class PlayerActivity : AppCompatActivity() {

    private var playerService: PlayerService? = null
    private var bound = false

    // 注意：这两个回调都在主线程执行，别在里面做耗时操作
    private val connection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
            val binder = service as PlayerService.LocalBinder
            playerService = binder.getService()
            bound = true
        }

        // 只在"异常断开"（如服务进程被杀）时回调，正常 unbind 不会回调
        override fun onServiceDisconnected(name: ComponentName?) {
            playerService = null
            bound = false
        }
    }

    override fun onStart() {
        super.onStart()
        bindService(Intent(this, PlayerService::class.java), connection, Context.BIND_AUTO_CREATE)
    }

    override fun onStop() {
        super.onStop()
        if (bound) {
            unbindService(connection)   // 解绑，否则泄漏 ServiceConnection / Activity
            bound = false
            playerService = null
        }
    }
}
```

本地 Binder 只适合同一进程。**跨进程**必须走 AIDL 或 Messenger，见 IPC 那篇文档。

### 3.4 用协程管理后台任务

`LifecycleService` 自带 `lifecycleScope`，销毁时自动取消：

```kotlin
class SyncService : LifecycleService() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        super.onStartCommand(intent, flags, startId)   // 必须调 super
        lifecycleScope.launch(Dispatchers.IO) {
            // 服务销毁时 scope 自动取消
        }
        return START_NOT_STICKY
    }
}
```

普通 Service 则自己维护 `CoroutineScope`：

```kotlin
class SyncService : Service() {
    // SupervisorJob：子任务失败不影响其他子任务
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        scope.launch { /* doWork() */ }
        return START_NOT_STICKY
    }

    override fun onDestroy() {
        scope.cancel()   // 关键：不取消就会泄漏线程和引用
        super.onDestroy()
    }
}
```

### 3.5 通知更新的正确姿势

- **用同一个 `NOTIFICATION_ID`**，`notify()` 会覆盖而不是新增；
- 进度更新别太频繁（几百毫秒一次即可），否则通知栏刷新掉帧；
- 结束后要么 `stopForeground(STOP_FOREGROUND_REMOVE)` 直接移除，要么主动 `cancel(NOTIFICATION_ID)`。

### 3.6 IntentService / JobIntentService 的废弃

| 类 | 状态 | 替代方案 |
| --- | --- | --- |
| `IntentService` | **API 30（Android 11）起废弃** | 协程 + `Service`，或直接用 WorkManager |
| `JobIntentService` | **已废弃** | `WorkManager` |

它们当年解决的是"串行执行 + 不阻塞主线程 + 干完自动停"，现在这些 WorkManager 和协程都能更干净地做到。

### 3.7 常用 API 速查表

| API | 作用 |
| --- | --- |
| `startService(intent)` | 启动服务（后台调用在 8.0+ 会受限） |
| `startForegroundService(intent)` | 启动前台服务，必须 5 秒内 `startForeground` |
| `bindService(intent, conn, flags)` | 绑定服务，`BIND_AUTO_CREATE` 表示没实例就创建 |
| `unbindService(conn)` | 解绑，必须与 `bindService` 配对 |
| `startForeground(id, notification)` | 进入前台并挂常驻通知 |
| `startForeground(id, notification, type)` | 指定前台服务类型（Android 14 可用） |
| `stopForeground(STOP_FOREGROUND_REMOVE)` | 退出前台并移除通知 |
| `stopSelf()` / `stopSelf(startId)` | 停止服务；带 `startId` 可避免误停新请求 |
| `stopService(intent)` | 从外部停止服务 |
| `onTaskRemoved(intent)` | 用户从最近任务划掉 App 时回调（可用于清理，不要用于拉活） |
| `onTimeout(startId)` | 前台服务"超时"时回调（如 `shortService` 到期，Android 14） |

## 四、常见坑与排查

### 4.1 在 Service 主线程做重活导致 ANR

Service 回调都在主线程。网络、数据库、大文件读写一律丢到 `Dispatchers.IO` / 线程池。排查用 ANR 日志，看主线程堆栈是不是卡在 Service 的回调里。

### 4.2 以为 `START_STICKY` 就一定能复活

见 1.4。返回值只是"意愿"，Doze、后台限制、厂商 ROM 都会打断。要可靠执行用 WorkManager；要持续运行用前台服务。

### 4.3 忘记 `stopSelf` / `stopForeground` 导致常驻泄漏

启动了不停止的服务会一直挂着通知、占着进程。**任务结束就 `stopForeground(STOP_FOREGROUND_REMOVE)` + `stopSelf(startId)`**。

### 4.4 `startForegroundService` 后 5 秒内没 `startForeground` 直接崩

典型异常 `ForegroundServiceDidNotStartInTimeException`（`RemoteServiceException` 的子类）。原因通常是：

- 在 `startForeground` 之前做了耗时初始化（等网络、建连接）；
- 通知渠道没建好，构通知时抛异常了。

**规矩**：`onStartCommand` 一进来就先 `startForeground`，再去干活。

### 4.5 Android 14 前台服务类型不匹配报异常

- 没声明 `foregroundServiceType` → 直接异常；
- 声明了类型但没申请对应 `FOREGROUND_SERVICE_*` 权限 → `SecurityException`；
- 声明了 `dataSync` 却用了定位/麦克风等 → 类型与行为不匹配，系统抛 `SecurityException` 或拒绝。

**规矩**：类型、权限、实际行为三者必须一致。

### 4.6 `ServiceConnection` 不解绑导致泄漏

`bindService` 一定要在 `onStop`/`onDestroy` 配对 `unbindService`，否则 Activity 泄漏、Service 也不会销毁。绑定回调里**别持有 Activity 的强引用做耗时任务**。

### 4.7 绑定后 Activity 旋转

旋转会重建 Activity → 重新 `bindService`。正确做法：

- 把"业务状态"放 **ViewModel**，把"长生命周期对象"（播放器、连接）放 **Service** 或 ViewModel；
- 避免在 Activity 里 new 出需要跨配置变化存活的对象。

### 4.8 前台服务通知被用户划掉

Android 13 起，用户默认可以直接划掉很多类型的前台服务通知。注意：

- **划掉通知 ≠ 一定停止服务**，服务通常还在跑；
- 用户从**任务管理器"停止"**会终止整个应用（Android 13 起），这是系统行为，不要试图绕过；
- 所以业务侧要能容忍"服务没了"的情况，用 WorkManager / 服务端状态兜底，而不是假设它永远在。

### 4.9 `onTaskRemoved` 不是"拉活"手段

`onTaskRemoved` 只在用户从最近任务划掉 App 时回调。可以用它做**清理**（比如停止播放、清理临时文件），**不要**在里面重启服务——后台启动服务会被限制，而且持续拉起会被系统惩罚。

### 4.10 Service / WorkManager / BroadcastReceiver 选型

| 需求 | 选谁 |
| --- | --- |
| 用户可感知的持续任务（播放、导航、录音） | 前台服务 |
| 可延迟、需要保证执行的后台任务（同步、上报） | WorkManager |
| 跨进程共享同一实例、要调用方法 | 绑定服务（AIDL/Messenger） |
| 只是"发个事件通知别人" | 广播（同应用内优先用 `LiveData`/`Flow`，跨进程才用广播） |
| 一次性短计算 | 协程 |

## 五、面试追问速答

**Q1：Service 和 Thread 的区别是什么？**
Service 是 Android 的**组件**，由系统管理生命周期，运行在所在进程的**主线程**上，本身不提供线程能力；Thread 是 Java 的**并发执行单元**，没有生命周期概念，也不受系统组件管理。很多人误以为 Service 是后台线程，其实在 Service 里做耗时操作一样 ANR，必须自己开线程/协程。二者可以配合：Service 负责"生命周期与进程优先级"，Thread/协程负责"真正干活"。

**Q2：IntentService 为什么废弃，用什么替代？**
IntentService 当年解决的是"把 intent 排队、在子线程串行执行、干完自动 stopSelf"。它的问题是不能并行、不能感知约束、进程被杀就丢、`onHandleIntent` 无法方便地回传结果。API 30 起被标记废弃。替代方案：需要"可延迟、可重试、进程重启后仍要执行"→ **WorkManager**；需要"当前进程内异步干活且跟组件生命周期绑定"→ **协程**（`lifecycleScope` / 自定义 `CoroutineScope`）。

**Q3：怎么保证后台任务不丢？**
第一层是**选对工具**：可延迟的任务用 WorkManager，它把任务持久化到数据库，进程被杀、设备重启后都能恢复，还能设约束（联网、充电）和自动重试。第二层是**业务幂等与对账**：任何上报都带唯一 ID，服务端去重，客户端在下次启动时按本地状态补拉补传。第三层才是前台服务——它只保证"任务运行期间不被轻易杀掉"，不保证"任务一定完成"，不能当唯一手段。

**Q4：前台服务在 Android 14 有哪些新要求？**
三点：一是**必须声明 `android:foregroundServiceType`**，不声明直接异常；二是**必须申请与该类型对应的 `FOREGROUND_SERVICE_*` 权限**（如 `dataSync` 对应 `FOREGROUND_SERVICE_DATA_SYNC`），否则抛 `SecurityException`；三是**声明的类型必须和实际行为匹配**（比如声明 `dataSync` 却去用定位/麦克风，会被系统拒绝）。此外 Android 14 新增了 `shortService` 类型用于"后台短时间必须完成"的任务，到期会有超时回调。

**Q5：START_STICKY 到底能不能复活？**
它能表达"系统愿意的话可以重建服务"的意愿，重建时给一个 `null` 的 intent。但它**不是保证**：Android 6 起的 Doze/App Standby 会冻结后台应用，Android 8 起后台启动服务本身受限，各家厂商 ROM 还有自己的省电策略。所以在现代 Android 上不能依赖它做保活。要任务可靠执行就用 WorkManager，要用户可感知地持续运行就用前台服务，并且 `onStartCommand` 里必须对 `intent == null` 做判空。

**Q6：绑定服务与启动服务的区别与组合使用？**
启动服务（`startService`）启动后与启动者解耦，靠 `stopSelf`/`stopService` 结束，外界拿不到它的方法；绑定服务（`bindService`）通过 `ServiceConnection` 拿到 `IBinder`，可以像调本地对象一样调它的方法，生命周期**跟随绑定者**——所有绑定者解绑后就销毁。组合使用（先 `startService` 再 `bindService`）的场景是音乐播放器：`startService` 保证即使界面退出音乐也继续放，`bindService` 让界面能拿到实例做进度条、切歌等交互；此时必须**既 `stopSelf` 又全部 `unbind`**，服务才会真正销毁。

**Q7：后台被杀如何做任务恢复？**
分两类看。**可延迟的后台任务**：交给 WorkManager，任务本身持久化，进程被杀或设备重启后系统会重新调度，天然具备恢复能力。**用户可感知的持续任务**（播放、下载）：用前台服务运行，被杀后靠"用户重新打开 App 时读取本地状态"恢复——比如把播放位置、下载进度持久化，重启后从断点继续；同时可以用 `START_REDELIVER_INTENT` 让系统在重启服务时把最后一个 intent 重新投递，但不要依赖它。核心思路是**把状态持久化 + 做幂等**，而不是指望系统帮你把服务拉起来。
