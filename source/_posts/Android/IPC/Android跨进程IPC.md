---
title: Android 跨进程通信（IPC）：原理 / 使用场景 / 常用 API / 常见坑
date: 2026-10-21 00:26:45
---

# Android 跨进程通信（IPC）：原理 / 使用场景 / 常用 API / 常见坑

> 一句话定位：Android 的 IPC 主角是 **Binder**——一个基于 mmap 的内核驱动，用"一次拷贝 + C/S 架构 + 面向对象引用"换掉了 Linux 原生的管道/socket；四大组件跨进程能力的底层本质都是 Binder 的应用层封装。
> 版本提示：Binder 的事务缓冲区约 1MB 量级、线程池默认上限 16 等数字是"量级概念"，不同版本可能有微调，答到量级即可。

---

## 一、IPC 是什么（原理）

### 1.1 为什么需要 IPC：进程隔离

Android 每个应用跑在独立进程里，**每个进程有自己独立的虚拟地址空间**，A 进程不能直接读写 B 进程的内存（这是操作系统的内存保护机制）。所以想在两个进程间传数据或调用方法，必须经过内核中转，这就是 IPC（Inter-Process Communication）。

### 1.2 Linux 有 IPC，Android 为什么要自己造 Binder

Linux 本身就有管道、消息队列、共享内存、信号、socket 等 IPC 手段，但 Android 选择了自研 Binder，主要原因：

| 维度 | Binder 的优势 |
| --- | --- |
| **性能** | **一次拷贝**：发送方数据拷到内核缓冲区，内核通过 mmap 把这块缓冲区映射到接收进程用户空间，接收方直接读——比 socket/管道少一次拷贝 |
| **架构** | 天生 **C/S 架构**：客户端拿到的其实是服务端的 **Proxy**，调用像调本地对象一样自然 |
| **面向对象** | 支持把"对象引用"跨进程传递，能拿到 IBinder 句柄远程调用方法（Linux 原生 IPC 只有"字节流"概念） |
| **安全** | 内核在传输时带上 **UID/PID**，服务端可以校验调用方身份 |
| **线程池** | 服务端有 Binder 线程池，能并发处理多个客户端请求 |
| **死亡通知** | 支持 `linkToDeath`，能感知对端进程死亡并做重连 |

### 1.3 "一次拷贝"具体是什么意思

先看传统方式（socket / 管道）：

```
发送方用户空间 ──①拷贝──▶ 内核缓冲区 ──②拷贝──▶ 接收方用户空间
                        （两次拷贝）
```

Binder 的做法：

```
发送方用户空间 ──①拷贝──▶ 内核缓冲区(位于接收进程地址空间)
                              │  （内核把它 mmap 到接收进程用户空间）
                              ▼
                       接收方直接读取（省掉第二次拷贝）
```

关键点：**Binder 驱动通过 `mmap` 把内核里用于接收的那块缓冲区，直接映射进接收进程的用户空间**，所以内核往里写就等于写进了接收方内存，接收方不需要再把数据从内核拷一遍到用户空间。这样总拷贝次数从 2 次降到 1 次，数据量越大优势越明显。底层是内核里的 Binder 驱动，对应设备节点 **`/dev/binder`**，客户端和服务端通过 `ioctl` 与它交互。

### 1.4 Binder 一次跨进程调用发生了什么

以 AIDL 为例，A 进程调 B 进程的方法：

```
A 进程（客户端）                        B 进程（服务端）
   │
   ├─ 拿到的是 IBinder.Proxy（不是真对象）
   ├─ Proxy 把参数按顺序写入 Parcel（序列化）
   ├─ 通过 ioctl 把 Parcel 交给 Binder 驱动
   │                                   │
   │        Binder 驱动（/dev/binder）  │
   │        · 查表找到目标进程的 binder_node
   │        · 分配/复用目标进程的 Binder 线程执行
   │        ─────────────────────────▶  │
   │                                    ├─ Stub.onTransact() 被调用
   │                                    ├─ 从 Parcel 按同样顺序读出参数
   │                                    ├─ 执行真正的业务方法
   │                                    └─ 结果写回 Parcel
   │        ◀─────────────────────────  │
   ├─ 收到结果 Parcel，反序列化返回给调用方
```

所以一句话：**AIDL 生成的 `Stub` / `Proxy` 只是把方法调用"翻译"成 Parcel 读写，真正干活的是 Binder 驱动 + 服务端的 Binder 线程池**。

### 1.5 四大组件的跨进程本质

| 组件 | 跨进程能力的底层 |
| --- | --- |
| `Activity` | AMS（ActivityManagerService）通过 Binder 调用应用进程的 `IApplicationThread` |
| `Service` | `bindService` 返回的 `IBinder` 就是 Binder 对象（本地/AIDL/Messenger） |
| `ContentProvider` | `ContentResolver` 底层是 `IContentProvider` 这个 Binder 接口 |
| `Broadcast` | AMS 通过 Binder 把广播投递到各进程的 `IIntentReceiver` |

**它们全都是 Binder 在应用层的封装**。理解这一点，很多"为什么跨进程有大小限制""为什么回调不在主线程"就能串起来了。

## 二、IPC 方式对比与使用场景

### 2.1 各种 IPC 方式横向对比（必背）

| 方式 | 适用数据量 | 双向 | 性能 | 复杂度 | 典型场景 |
| --- | --- | --- | --- | --- | --- |
| **Bundle / Intent 传参** | 小（几百 KB 内，总事务约 1MB 量级） | 单向 | 高 | 极低 | 启动 Activity/Service 时传小参数 |
| **文件共享** | 大 | 单向/需自己加锁 | 低 | 低 | 少量、非实时的数据交换（注意并发安全） |
| **SharedPreferences** | 小 | 单向 | 低 | 低 | **不推荐跨进程**（`MODE_MULTI_PROCESS` 已废弃） |
| **ContentProvider** | 中到大 | 双向（CRUD） | 中 | 中 | 结构化数据共享，跨 App 数据访问 |
| **AIDL** | 中（受 1MB 限制） | **双向** | 高 | 高 | 跨进程方法调用 + 回调，需要精确控制 |
| **Messenger** | 小（只能传 Message） | 双向 | 中 | 中 | 简单的跨进程消息传递，天然串行 |
| **Socket** | 大 | 双向 | 中（两次拷贝） | 中 | 跨设备/本机网络通信 |
| **共享内存（MemoryFile/Ashmem）** | **很大** | 双向 | **最高（零拷贝）** | 高 | 传大图/大视频帧，需自己做同步 |
| **Broadcast** | 小 | 单向 | 低 | 低 | 一对多通知，跨 App 事件 |
| **Binder 直连** | 中 | 双向 | 高 | 高 | 直接持有 IBinder 做精细控制（AIDL 的底层） |

### 2.2 场景表

| 场景 | 推荐方案 | 说明 |
| --- | --- | --- |
| 多进程拆分（WebView / 图库 / 播放器 / 推送独立进程） | `android:process` + 按需 AIDL/ContentProvider | 崩溃隔离、内存隔离 |
| 跨 App 能力调用（支付、分享、健康数据） | AIDL / ContentProvider / 系统 Intent | 看对方开放的是接口还是数据 |
| 数据共享（多个 App 读同一份数据） | **ContentProvider** | 配合权限控制，支持 `ContentObserver` 监听变化 |
| 长连接式控制（音乐控制器、跨进程服务控制） | **AIDL** + `RemoteCallbackList` | 需要方法与回调 |
| 简单消息跨进程传递 | **Messenger** | 不想写 AIDL 时的简化选择，天然串行 |
| 大文件 / 大图传输 | **传 URI 或文件描述符（FD）** | 千万别把大 Bitmap 塞进 Bundle |
| 高吞吐大块数据（视频帧、大图） | 共享内存（`MemoryFile` / Ashmem） | 零拷贝，但同步要自己处理 |

### 2.3 不适合用多进程 / IPC 的场景

| 需求 | 别这么做，改用 | 原因 |
| --- | --- | --- |
| 单进程就能满足的业务 | **就留在主进程** | 多进程要付出 Application 多次初始化、静态状态不共享、IPC 复杂度上升的代价，收益往往不划算 |
| 高频小数据交互 | AIDL / Flow，**别用 Broadcast** | 广播开销大、不保证顺序和可靠，高频场景性能差 |
| 传大数据（大图/大文件） | 传 **URI / 文件描述符**，别塞进 Bundle/AIDL 参数 | 会触发 `TransactionTooLargeException` |
| 少量、低频、非实时的配置共享 | 文件 / DataStore，**别上 AIDL** | 杀鸡用牛刀，实现和调试成本都更高 |
| 需要强一致的多进程共享内存状态 | 统一收口到一个进程写 | 多进程各自缓存会导致数据不一致，维护成本高 |

## 三、常用 API 与使用方法

### 3.1 AIDL 完整示例

**① 定义接口（`IPlayerController.aidl`）**

```java
// IPlayerController.aidl
package com.example.ipc;

import com.example.ipc.PlayState;              // Parcelable 自定义类型必须显式 import
import com.example.ipc.IPlayerListener;

interface IPlayerController {
    void play(in String url);                 // 默认 in：客户端 → 服务端
    int getPosition();                        // 有返回值：同步调用，会阻塞到返回
    void registerListener(IPlayerListener listener);
    void unregisterListener(IPlayerListener listener);
    void setVolume(float volume);
    oneway void close();                      // oneway：异步、无返回值，不等服务端处理完
}
```

**② in / out / inout 的方向语义**

| 方向 | 数据流向 | 说明 |
| --- | --- | --- |
| `in` | 客户端 → 服务端 | 默认。客户端传进去的值服务端能读到，服务端修改**不会**回传 |
| `out` | 服务端 → 客户端 | 客户端传的只是"容器"，里面值不保证有效；服务端写完后回传 |
| `inout` | 双向 | 服务端修改会回传，代价是**双向序列化，开销最大** |

原理：`in` 只做一次序列化（客户端写、服务端读）；`out`/`inout` 需要服务端把修改后的对象再序列化回来，多一次拷贝。**能用 `in` 就别用 `inout`**。基本类型和 `String` 默认就是 `in`。

**③ oneway 的含义**

`oneway` 表示这次调用**不阻塞客户端、没有返回值、不保证执行顺序**（多个 oneway 调用可能被并行处理）。适合"通知类、不关心结果"的方法，比如 `close()`。

**④ 自定义 Parcelable 类型（`PlayState.aidl`）**

```java
// PlayState.aidl —— 只需声明，具体实现写 Kotlin/Java 类
package com.example.ipc;
parcelable PlayState;
```

**⑤ 服务端实现（Stub）**

```kotlin
class PlayerService : Service() {

    private val listeners = RemoteCallbackList<IPlayerListener>()   // 跨进程回调集合

    private val binder = object : IPlayerController.Stub() {
        override fun play(url: String) { /* 注意：运行在 Binder 线程，不是主线程 */ }

        override fun getPosition(): Int = 0

        override fun registerListener(listener: IPlayerListener?) {
            listeners.register(listener)     // 内部持有的是 Binder 引用，弱引用式管理
        }

        override fun unregisterListener(listener: IPlayerListener?) {
            listeners.unregister(listener)
        }

        override fun setVolume(volume: Float) {}

        override fun close() { stopSelf() }

        // 想回调客户端：
        private fun notifyState(state: Int) {
            val n = listeners.beginBroadcast()
            try {
                for (i in 0 until n) {
                    listeners.getBroadcastItem(i)?.onStateChanged(state)
                }
            } finally {
                listeners.finishBroadcast()   // 必须成对调用
            }
        }
    }

    override fun onBind(intent: Intent?): IBinder = binder

    override fun onUnbind(intent: Intent?): Boolean {
        listeners.kill()      // 解绑时清空所有跨进程回调，防止泄漏/崩溃
        return super.onUnbind(intent)
    }
}
```

**⑥ 客户端调用**

```kotlin
class MainActivity : AppCompatActivity() {

    private var controller: IPlayerController? = null

    private val conn = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
            // 关键：客户端拿到的 service 是 IBinder，要转成接口
            controller = IPlayerController.Stub.asInterface(service)
            controller?.registerListener(listener)
        }

        override fun onServiceDisconnected(name: ComponentName?) {
            controller = null      // 服务进程异常死亡，必须置空并准备重连
        }
    }

    private val listener = object : IPlayerListener.Stub() {
        override fun onStateChanged(state: Int) { /* 在 Binder 线程回调，改 UI 要切主线程 */ }
    }

    override fun onStart() {
        super.onStart()
        bindService(Intent(this, PlayerService::class.java), conn, Context.BIND_AUTO_CREATE)
    }

    override fun onStop() {
        super.onStop()
        controller?.unregisterListener(listener)
        unbindService(conn)
        controller = null
    }
}
```

> `onServiceDisconnected` **只在异常断开时回调**（比如服务进程被杀），正常 `unbindService` 不会触发它。所以断连重连逻辑要写在这里。

### 3.2 Messenger（AIDL 的简化版）

底层也是 AIDL，只是把接口固定成"收发 Message"，**天然串行**（所有消息在服务端一个 Handler 线程上排队处理），只能传 `Message`/`Bundle`，不能传返回值和复杂对象。

```kotlin
// 服务端
class MessengerService : Service() {
    private lateinit var messenger: Messenger

    override fun onCreate() {
        super.onCreate()
        val handler = object : Handler(Looper.getMainLooper()) {
            override fun handleMessage(msg: Message) {
                when (msg.what) {
                    MSG_SAY_HELLO -> { /* 处理请求，可通过 msg.replyTo 回消息 */ }
                    else -> super.handleMessage(msg)
                }
            }
        }
        messenger = Messenger(handler)
    }

    override fun onBind(intent: Intent?): IBinder = messenger.binder
}

// 客户端
val conn = object : ServiceConnection {
    override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
        val remote = Messenger(service)
        val msg = Message.obtain(null, MSG_SAY_HELLO)
        msg.replyTo = localMessenger      // 想收回复就把自己的 Messenger 带上
        remote.send(msg)                  // 异步发送
    }
    override fun onServiceDisconnected(name: ComponentName?) { }
}
```

**AIDL vs Messenger 怎么选**：需要并发、返回值、回调接口 → AIDL；只是简单异步消息、能接受串行 → Messenger（省去写 AIDL）。

### 3.3 ContentProvider 跨进程访问

```kotlin
// Provider 端：实现 CRUD
class NoteProvider : ContentProvider() {
    override fun onCreate(): Boolean = true          // 在主线程、且先于 Application.onCreate 执行

    override fun query(
        uri: Uri, projection: Array<String>?, selection: String?,
        selectionArgs: Array<String>?, sortOrder: String?
    ): Cursor? { /* 返回 Cursor */ return null }

    override fun insert(uri: Uri, values: ContentValues?): Uri? = null
    override fun update(uri: Uri, values: ContentValues?, selection: String?, args: Array<String>?): Int = 0
    override fun delete(uri: Uri, selection: String?, args: Array<String>?): Int = 0
    override fun getType(uri: Uri): String? = null
}
```

```kotlin
// 客户端：通过 ContentResolver 访问（即使跨进程也像调本地一样）
val cursor = contentResolver.query(
    Uri.parse("content://com.example.provider/notes"),
    null, null, null, null
)

// 监听数据变化
contentResolver.registerContentObserver(uri, /* notifyForDescendants = */ true, object : ContentObserver(Handler(Looper.getMainLooper())) {
    override fun onChange(selfChange: Boolean) { /* 数据变了，重新查询 */ }
})
```

### 3.4 多进程配置（Manifest）

```xml
<manifest>
    <!-- 私有进程：只有本应用能进去，名称以 ":" 开头 -->
    <activity android:name=".player.PlayerActivity" android:process=":player" />
    <service android:name=".push.PushService" android:process=":push" />

    <!-- 全局进程：以包名全称命名，可以被其他应用共享（需同签名/共享 UID） -->
    <service android:name=".shared.SharedService" android:process="com.example.shared" />
</manifest>
```

**多进程的代价（重点）**：

1. **每个进程都会创建一个 Application 实例**，`Application.onCreate` 会被执行多次。必须按进程名做差异化初始化，否则会重复初始化、浪费内存甚至出错：

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        if (currentProcessName() == packageName) {
            // 只有主进程才做这些（初始化埋点、推送等）
        }
        // 所有进程都要做的（比如日志）
    }

    private fun currentProcessName(): String {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
            Application.getProcessName()
        } else {
            // 低版本兜底：读 /proc/self/cmdline
            File("/proc/self/cmdline").readText().trim('\u0000', ' ')
        }
    }
}
```

2. **静态变量、单例、各种缓存都不共享**：每个进程一份，改了一个另一个看不到。

### 3.5 跨进程传大文件的正确方式

**永远不要**把大 Bitmap / 大 List / 大字符串塞进 Bundle 或 AIDL 参数，会触发 `TransactionTooLargeException`。正确做法：

| 方案 | 做法 |
| --- | --- |
| 传 URI | 用 `content://` URI，接收方自己通过 `ContentResolver.openInputStream` 读 |
| 传文件描述符（FD） | AIDL 里用 `ParcelFileDescriptor`，接收方直接拿到 FD 读写，不经过 Binder 数据拷贝 |
| 传文件路径 | 仅限双方有共同可访问目录（如私有目录不适用）时 |

```kotlin
// AIDL 里声明返回文件描述符
// ParcelFileDescriptor getFile();
override fun getParcelFileDescriptor(): ParcelFileDescriptor? =
    ParcelFileDescriptor.open(File(cacheDir, "big.bin"), ParcelFileDescriptor.MODE_READ_ONLY)
```

## 四、常见坑与排查

### 4.1 `TransactionTooLargeException`

Binder 的事务缓冲区大小有限（**约 1MB 量级，且是每个进程共享的**，所有并发事务共用这块空间），传大 Bitmap、大 List、大 JSON、大 Bundle 都会崩。而且这个限制**不只是 Bundle**——AIDL 的参数同样受限。解法：

- 传 **URI / 文件描述符** 而不是内容本身；
- 传大图先压缩/落盘，再传路径；
- 分片传输（每次传一小块，服务端拼装）。

### 4.2 Binder 线程池有限，嵌套同步调用容易阻塞甚至死锁

Binder 线程池**默认最多 16 个线程**。如果 A 同步调 B、B 又同步调回 A，而 A 的 Binder 线程都被占满在等 B 返回，就会**互相等待形成死锁**。规避：**跨进程调用尽量是单向的（oneway），或者把同步调用改成异步回调**；别写"双向同步调用"的环。

### 4.3 AIDL 方法在服务端的 Binder 线程执行

服务端 `Stub` 里的方法**不在主线程、也不在 UI 线程**，而在 Binder 线程池的某个线程里。所以：

- 想更新 UI 必须切回主线程（`Handler` / `LifecycleScope` / `runOnUiThread`）；
- 不能被 `Looper` 依赖的逻辑（如直接刷 View）会崩。

同理，**客户端收到的回调也在 Binder 线程**，改 UI 要切主线程。

### 4.4 进程被杀后 Binder 对象失效

对端进程死亡后，它发过来的代理就失效了，调用会抛 `RemoteException` / `DeadObjectException`。必须：

```kotlin
try {
    controller?.getPosition()
} catch (e: RemoteException) {
    controller = null    // 置空 + 走重连流程
}
```

想**主动感知**死亡，用 `linkToDeath`：

```kotlin
binder.linkToDeath(object : IBinder.DeathRecipient {
    override fun binderDied() {
        // 对端进程挂了，在这里触发重连
    }
}, 0)
```

### 4.5 多进程下静态变量 / 单例 / SharedPreferences 缓存不共享

每个进程是独立的 JVM 内存空间，静态变量和单例都是**各一份**。跨进程共享数据必须走 IPC（ContentProvider / 文件 / AIDL），不能靠静态变量。`SharedPreferences` 的 `MODE_MULTI_PROCESS` **已废弃且不可靠**（各进程有各自的缓存，会读到脏数据），跨进程共享请用 ContentProvider 或数据库。

### 4.6 Application 被创建多次导致重复初始化

见 3.4。多进程下 `Application.onCreate` 每个进程都会走一次，必须在里面**按进程名判断**，只让主进程做重初始化，否则会重复注册、重复起服务、浪费内存。

### 4.7 ContentProvider 的 `onCreate` 在主线程且先于 `Application.onCreate`

`ContentProvider.onCreate()` 在**应用启动早期、主线程**上执行，而且**比 `Application.onCreate` 还早**。所以：

- 别在里面做耗时操作（会拖慢冷启动、容易 ANR）；
- 有些库会用它来"偷偷"拿 `Context` 做初始化，理解这个时序很重要。

### 4.8 `oneway` 不保证顺序、也没有返回值

`oneway` 是异步的，调用方不等服务端处理完就返回，多个 `oneway` 调用**可能被并行处理，顺序不保证**，也拿不到返回值。所以涉及顺序依赖或需要结果的逻辑，不能用 `oneway`。

### 4.9 Parcel 读写顺序必须严格一致

`Parcelable` 或 AIDL 参数里，**写的时候什么顺序，读的时候就必须什么顺序**，类型也要对应。写错顺序/漏读会导致数据错乱或崩溃，而且很难查。自定义 `Parcelable` 时 `writeToParcel` 和 `CREATOR.createFromParcel` 必须一一对应。

### 4.10 多进程下数据库并发写

多个进程同时用 SQLite/Room 写同一个库，会互相锁、抛 `SQLiteDatabaseLockedException`。应对：

- **尽量让"写"只在一个进程里做**，其他进程通过 IPC 请求它写；
- 打开 WAL 模式、设置合理的超时；
- 数据量不大时考虑改用 ContentProvider 统一收口。

## 五、面试追问速答

**Q1：为什么 Android 选 Binder 而不用 socket？**
核心是**性能、架构和安全**三点。性能上 Binder 只有一次拷贝（借助 mmap 把内核缓冲区映射进接收方用户空间），socket/管道要走两次拷贝；架构上 Binder 天生是 C/S 模式，客户端拿到的是 Proxy，能像调本地对象一样调远程方法，而 socket 只是字节流，所有协议要自己定义；安全上 Binder 由内核携带调用方的 UID/PID，服务端可校验身份，socket 拿不到这些。此外 Binder 还支持线程池、对象引用传递和 `linkToDeath` 死亡通知，这些对组件化系统非常关键。

**Q2："一次拷贝"具体指什么，为什么比管道快？**
指数据从**发送方用户空间拷贝到内核缓冲区**这一次，而这块内核缓冲区被内核通过 `mmap` 直接映射到了**接收进程的用户空间**，所以内核往里写就等于写进了接收方内存，接收方**不需要再从内核拷一次到用户空间**。管道/socket 的路径是"发送方用户空间 → 内核 → 接收方用户空间"，两步拷贝。Binder 省掉的就是第二步。数据量越大，少一次内存拷贝带来的优势越明显。

**Q3：Binder 的 1MB 限制在哪一层，怎么绕过？**
限制在 **Binder 驱动层的事务缓冲区**：每个进程用于收发事务的缓冲区大小有限，约 1MB 量级，而且**是该进程上所有并发事务共享**的，不是每个事务独享 1MB。一旦超过就抛 `TransactionTooLargeException`。要绕过它：传大数据时不要塞内容，改传 **URI 或文件描述符（`ParcelFileDescriptor`）**，让接收方自己去读；或者分片传输。注意这个限制不只针对 Bundle，AIDL 参数同样受限。

**Q4：怎么判断当前进程是不是主进程？**
最直接的是**把当前进程名和包名比较**：进程名等于包名的就是主进程。API 28（Android 9）起可以直接用 `Application.getProcessName()`；更低版本可以读 `/proc/self/cmdline`，或者用一个 `ContentProvider` 在启动早期取进程名（一些多进程库就是这么做的）。拿到进程名后，在 `Application.onCreate` 里做判断，只让主进程执行重初始化，子进程按需轻量化初始化。

**Q5：进程被杀后怎么优雅恢复？**
两步：**先感知、再重连**。感知上用 `IBinder.linkToDeath` 注册 `DeathRecipient`，对端进程死亡时会回调 `binderDied()`；同时所有跨进程调用都要 `try-catch` 住 `RemoteException`/`DeadObjectException`，一旦抛出就把本地代理置空。重连上，在 `DeathRecipient` 回调或 `onServiceDisconnected` 里触发 `bindService` 重连，并重新注册回调和恢复状态。此外业务状态要持久化，恢复后能接着干，而不是假设对端永远在线。

**Q6：AIDL、Messenger、ContentProvider 怎么选？**
`AIDL`：需要**跨进程方法调用、并发、返回值、回调接口**，控制力最强但最复杂。`Messenger`：只需要**简单的异步消息传递**，能接受服务端**串行**处理、只传 `Message`/`Bundle`、不关心返回值，实现成本低（本质是基于 Handler 的 AIDL 封装）。`ContentProvider`：主要解决**结构化数据的跨进程/跨 App 共享**，提供 CRUD 和 `ContentObserver` 监听，偏向"数据"而非"行为"。一句话：**调方法用 AIDL，传消息用 Messenger，共享数据用 ContentProvider**。

**Q7：多进程下哪些东西会失效？**
会在多个独立内存空间里"各存一份"而失效的有：**静态变量、单例、内存缓存、SharedPreferences 的内存缓存**（`MODE_MULTI_PROCESS` 已废弃，不可靠）。此外 **`Application` 会被创建多次**（每个进程一个），需要按进程名做差异化初始化；**Binder 对象、回调在进程死亡后失效**，要 `linkToDeath` + 重连；**数据库**多进程并发写会有锁冲突，要统一写入口或做好并发控制。解决办法是：跨进程共享数据一律走 IPC（ContentProvider / 文件 / 数据库收口），不要依赖进程内的静态状态。
