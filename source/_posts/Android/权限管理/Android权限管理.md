---
title: Android 权限管理：原理 / 版本演进 / 使用场景 / 常用 API / 常见坑
date: 2026-10-21 00:25:44
---

# Android 权限管理：原理 / 版本演进 / 使用场景 / 常用 API / 常见坑

> 一句话定位：Android 权限本质是"内核 uid 隔离 + 包管理器记账 + 用户授权"三件事，从 Android 6.0 起由"装完就给"变成"用的时候再问"，之后每一代系统都在把权限切得更细、管得更严。
> 版本口径：下表按 Android 6（API 23）到 Android 15（API 35）梳理，行为差异一律写清"从哪个版本开始变"，具体细节以官方文档和真机为准。

---

## 一、Android 权限是什么（原理）

### 1.1 底座：Linux 的 uid / gid 与应用沙箱

Android 跑在 Linux 内核上，权限体系的地基就是 Linux 那套：

- **每个 App 装上去就分配一个独立的 uid**（比如 `u0_a123`），不同 App 默认互相看不到对方的文件；
- **每个 App 在 `/data/data/<包名>/`（真实路径 `/data/user/0/<包名>/`）下有自己的私有目录**，别的 App 的 uid 没权限读，这就是"应用沙箱"；
- 历史上有 `android:sharedUserId` 可以让多个 App 共用一个 uid 互相访问数据，**已废弃**，新项目不要用；
- 应用要访问系统资源（相机、位置、网络），就是向内核/系统服务申请对应的 gid 或能力，这一步由系统在安装/授权时统一发牌。

所以"权限"不是 Android 发明的概念，**它是把 Linux 的 uid/gid 机制包装成用户能看懂的一张张许可单**。

### 1.2 关键转折：Android 6.0 的运行时权限

Android 6.0（API 23）之前，权限是**安装时一次性授予**的：装的时候弹一屏"这个 App 要这些权限"，要么全同意，要么别装。用户根本不会细看。

Android 6.0 起引入**运行时权限（Runtime Permission）**：

- App **装的时候不再索要**危险权限；
- 第一次真正用到某个能力时，App 主动弹窗申请，用户当场决定；
- 用户随时可以在系统设置里关掉某个权限；
- **对 `targetSdkVersion >= 23` 的 App 强制生效**；老 App（target < 23）走兼容逻辑，安装时授予。

一句话记忆：**"安装时授权" → "使用时申请 + 随时可撤"**。

### 1.3 权限的四种类型（protection level）

| 类型 | 授予方式 | 典型权限 | 特点 |
| --- | --- | --- | --- |
| **normal（普通）** | 安装时自动授予 | `INTERNET`、`ACCESS_NETWORK_STATE`、`VIBRATE`、`FOREGROUND_SERVICE` | 不涉及隐私，用户无感知，**申请了就有** |
| **dangerous（危险）** | 运行时弹窗申请 | `CAMERA`、`RECORD_AUDIO`、`ACCESS_FINE_LOCATION`、`POST_NOTIFICATIONS`、`READ_MEDIA_IMAGES` 等 | 涉及用户隐私，**必须运行时申请，用户可拒绝/撤销** |
| **signature（签名）** | 系统签名一致才授予 | 系统级/厂商内部权限 | 普通 App 拿不到，多用于系统 App 间 |
| **special（特殊）** | **不弹标准权限框，走系统设置页** | `SYSTEM_ALERT_WINDOW`（悬浮窗）、`REQUEST_INSTALL_PACKAGES`（装未知来源）、`WRITE_SETTINGS`、勿扰访问、`SCHEDULE_EXACT_ALARM`（精确闹钟） | 需要用户手动去设置页开，`requestPermissions` 申请不了 |

**面试最容易踩的坑**：悬浮窗、安装未知应用这类"特殊权限"，你调 `requestPermissions` 是**永远拿不到的**，必须跳设置页。

### 1.4 权限状态存在哪，怎么查

系统在**包管理器（PackageManagerService）**里维护每个包的权限授予状态，App 通过两个 API 查询：

```kotlin
// 1) 当前是否已授予
val granted = ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA)
        == PackageManager.PERMISSION_GRANTED

// 2) 是否需要向用户解释"为什么需要这个权限"
val needExplain = ActivityCompat.shouldShowRequestPermissionRationale(this, Manifest.permission.CAMERA)
```

`shouldShowRequestPermissionRationale` 的返回值是面试重点，它有三种状态（见 3.3）。

## 二、使用场景

| 场景 | 需要的权限 | 目标 API 版本注意事项 |
| --- | --- | --- |
| **拍照** | `CAMERA`（危险，运行时） | 自己调 Camera/CameraX 必须申请；**用系统相机 Intent（`ACTION_IMAGE_CAPTURE`）且 Manifest 没声明 `CAMERA` 时可免申请** |
| **定位（前台）** | `ACCESS_FINE_LOCATION` / `ACCESS_COARSE_LOCATION` | Android 12+ 用户可选"大致位置"，只给 `COARSE`；**请求 `FINE` 时必须同时请求 `COARSE`** |
| **定位（后台）** | 前台定位权限 + `ACCESS_BACKGROUND_LOCATION` | Android 10 起引入；**Android 11 起不能和前台定位一起弹窗**，必须先拿到前台定位，再引导用户去设置页选"始终允许" |
| **通知** | `POST_NOTIFICATIONS` | **Android 13（API 33）起是运行时权限**；13 以下不需要申请，但用户仍可在设置里关闭 |
| **读取媒体** | Android 13+：`READ_MEDIA_IMAGES` / `READ_MEDIA_VIDEO` / `READ_MEDIA_AUDIO`；Android 12 及以下：`READ_EXTERNAL_STORAGE` | Android 13 起媒体权限**按类型拆分**，取代 `READ_EXTERNAL_STORAGE` |
| **只访问部分照片** | `READ_MEDIA_VISUAL_USER_SELECTED` | **Android 14 起**，用户可只授权"选中的照片"，App 要处理"部分访问" |
| **蓝牙** | Android 12+：`BLUETOOTH_CONNECT` / `BLUETOOTH_SCAN`（运行时）；Android 11 及以下：`BLUETOOTH` / `BLUETOOTH_ADMIN`（normal） | **Android 12 起旧蓝牙权限被拆成运行时权限**，老代码不改造会直接拿不到 |
| **悬浮窗** | `SYSTEM_ALERT_WINDOW`（special） | 走 `Settings.ACTION_MANAGE_OVERLAY_PERMISSION`，必须用户手动开 |
| **安装未知来源应用** | `REQUEST_INSTALL_PACKAGES`（special） | 走 `ACTION_MANAGE_UNKNOWN_APP_SOURCES` 设置页 |
| **录音** | `RECORD_AUDIO`（危险，运行时） | Android 11+ 若在后台用麦克风，还要配套前台服务类型 |

**不适合用"权限"解决的场景**：只是想拿一张图给用户选？**别申请读媒体权限，用 Photo Picker**；只是想展示一个悬浮提示？**优先用应用内 UI，别为了悬浮窗去要 `SYSTEM_ALERT_WINDOW`**（上架审核也会问用途）。权限越少，过审越稳。

### 2.1 版本演进总表（面试必背）

| 版本 | API | 权限/存储的关键变化 |
| --- | --- | --- |
| **Android 6** | 23 | **引入运行时权限**：dangerous 权限改为使用时申请、可随时撤销 |
| **Android 10** | 29 | **分区存储（Scoped Storage）**默认开启（可用 `requestLegacyExternalStorage` 临时退回）；引入 **`ACCESS_BACKGROUND_LOCATION`**；定位弹窗改为"仅在使用时允许/始终允许"；`ACTIVITY_RECOGNITION` 变为运行时权限 |
| **Android 11** | 30 | **一次性授权**（仅此一次）；**长期不用的 App 权限自动重置**；**请求后台定位必须跳设置页**（不能与前台定位同时申请）；拒绝两次视为"不再询问"；分区存储**强制执行**（`requestLegacyExternalStorage` 被忽略），文件管理器类 App 改用 `MANAGE_EXTERNAL_STORAGE` |
| **Android 12** | 31 | **精确/大致位置分离**（用户可选只给大致位置）；**蓝牙权限拆分为 `BLUETOOTH_CONNECT` / `BLUETOOTH_SCAN`（+ `BLUETOOTH_ADVERTISE`）运行时权限**；**精确闹钟 `SCHEDULE_EXACT_ALARM`**（另有免申请的 `USE_EXACT_ALARM`，仅限闹钟/日历类）；麦克风/摄像头增加使用指示器与全局开关 |
| **Android 13** | 33 | **`POST_NOTIFICATIONS` 通知运行时权限**；**媒体权限按类型拆分：`READ_MEDIA_IMAGES` / `READ_MEDIA_VIDEO` / `READ_MEDIA_AUDIO` 取代 `READ_EXTERNAL_STORAGE`**；新增 `NEARBY_WIFI_DEVICES`、`BODY_SENSORS_BACKGROUND` |
| **Android 14** | 34 | **前台服务必须声明类型并申请对应 `FOREGROUND_SERVICE_*` 权限**（如 `FOREGROUND_SERVICE_LOCATION` / `_CAMERA` / `_MICROPHONE` / `_MEDIA_PROJECTION` 等）；**`READ_MEDIA_VISUAL_USER_SELECTED`：用户可只授权"选中的部分照片"** |
| **Android 15** | 35 | 权限层没有面向普通 App 的重大新运行时权限，重点是**后台行为进一步收紧**：对部分前台服务类型（如 `dataSync`、`mediaProcessing`）增加运行时长上限、后台启动前台服务限制更严。具体细节随版本调整，以官方文档为准 |

## 三、常用 API 与使用方法

### 3.1 Activity Result API 申请单个权限（推荐）

老的 `ActivityCompat.requestPermissions` + `onRequestPermissionsResult` 已被官方标记为过时，**统一改用 Activity Result API**：

```kotlin
class CameraFragment : Fragment() {

    // 必须作为属性初始化（或至少在 onCreate 中）注册，绝不能等到点击时才注册
    private val requestCamera = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted: Boolean ->
        if (granted) openCamera() else onCameraDenied()
    }

    private fun ensureCamera() {
        when {
            // 1) 已授权，直接用
            ContextCompat.checkSelfPermission(requireContext(), Manifest.permission.CAMERA)
                    == PackageManager.PERMISSION_GRANTED -> openCamera()

            // 2) 用户拒绝过但没勾"不再询问"：先解释再申请
            shouldShowRequestPermissionRationale(Manifest.permission.CAMERA) ->
                showRationaleDialog { requestCamera.launch(Manifest.permission.CAMERA) }

            // 3) 从未申请过 或 永久拒绝
            else -> {
                if (prefs.getBoolean("camera_requested", false)) {
                    guideToAppSettings()                       // 永久拒绝：只能引导设置页
                } else {
                    prefs.edit().putBoolean("camera_requested", true).apply()
                    requestCamera.launch(Manifest.permission.CAMERA)
                }
            }
        }
    }
}
```

### 3.2 一次申请多个权限

```kotlin
private val requestMultiple = registerForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions()
) { result: Map<String, Boolean> ->
    // result 是"权限 -> 是否授予"的 Map，部分授予是允许的
    val fineOk  = result[Manifest.permission.ACCESS_FINE_LOCATION] == true
    val coarseOk = result[Manifest.permission.ACCESS_COARSE_LOCATION] == true
    when {
        fineOk   -> getPreciseLocation()
        coarseOk -> getApproximateLocation()     // Android 12+ 用户可能只给大致位置
        else     -> onLocationDenied()
    }
}

// 注意：请求 FINE 时一定要带上 COARSE，否则在 Android 12+ 上会被直接忽略
requestMultiple.launch(arrayOf(
    Manifest.permission.ACCESS_FINE_LOCATION,
    Manifest.permission.ACCESS_COARSE_LOCATION
))
```

### 3.3 `shouldShowRequestPermissionRationale` 的三种状态（必考）

| 情况 | `checkSelfPermission` | `shouldShow...Rationale` | 正确做法 |
| --- | --- | --- | --- |
| **从未申请过** | DENIED | **false** | 直接 `launch()` 申请 |
| **拒绝过一次**（没勾不再询问） | DENIED | **true** | 先弹解释弹窗，用户同意后再申请 |
| **永久拒绝**（勾了不再询问 / Android 11 拒两次） | DENIED | **false** | **`launch()` 会立刻回调 denied**，必须引导去设置页 |

**关键点**：第 1 种和第 3 种的返回值**都是 false，光靠系统 API 分不出来**。所以你必须**自己用一个标记（SharedPreferences 等）记录"是否申请过"**，才能区分"第一次"和"永久拒绝"。这是面试最爱追的点。

### 3.4 永久拒绝后引导去应用详情设置页

```kotlin
private fun guideToAppSettings() {
    AlertDialog.Builder(this)
        .setTitle("需要相机权限")
        .setMessage("您已拒绝且不再询问，请到系统设置中手动开启。")
        .setPositiveButton("去设置") { _, _ ->
            startActivity(
                Intent(
                    Settings.ACTION_APPLICATION_DETAILS_SETTINGS,
                    Uri.fromParts("package", packageName, null)
                )
            )
        }
        .setNegativeButton("取消", null)
        .show()
}
```

### 3.5 特殊权限：悬浮窗（走设置页，不走权限框）

```kotlin
if (!Settings.canDrawOverlays(this)) {
    startActivity(
        Intent(
            Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
            Uri.parse("package:$packageName")     // 直接定位到本 App 的开关
        )
    )
}
// 用户从设置页回来后再查一次 canDrawOverlays，因为这里是异步的
```

其它特殊权限的查询/跳转 API（同样"申请了不代表有，用时必查"）：

| 特殊权限 | 查询 API | 跳转 Action |
| --- | --- | --- |
| 悬浮窗 | `Settings.canDrawOverlays(context)` | `ACTION_MANAGE_OVERLAY_PERMISSION` |
| 修改系统设置 | `Settings.System.canWrite(context)` | `ACTION_MANAGE_WRITE_SETTINGS` |
| 安装未知来源应用 | `packageManager.canRequestPackageInstalls()` | `ACTION_MANAGE_UNKNOWN_APP_SOURCES` |
| 精确闹钟 | `alarmManager.canScheduleExactAlarms()` | `ACTION_REQUEST_SCHEDULE_EXACT_ALARM` |
| 勿扰访问 | `notificationManager.isNotificationPolicyAccessGranted` | `ACTION_NOTIFICATION_POLICY_ACCESS_SETTINGS` |
| 所有文件访问 | `Environment.isExternalStorageManager()` | `ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION` |

### 3.6 拍照与相册：优先"零权限"方案

**① Photo Picker（Android 13+ 原生，低版本由 Activity Result API 兜底）—— 选图完全不需要权限**

```kotlin
val pickImage = registerForActivityResult(
    ActivityResultContracts.PickVisualMedia()
) { uri: Uri? ->
    uri?.let { showPreview(it) }        // 拿到用户主动选中的 Uri，无需 READ_MEDIA_*
}

pickImage.launch(
    PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
)
// 多选：ActivityResultContracts.PickMultipleVisualMedia(maxItems = 9)
```

**② 拍照写到自己指定的 Uri（系统相机 + FileProvider）**

```kotlin
// 事先准备一个 FileProvider 授权的目标 Uri（见文件系统篇）
val photoUri: Uri = FileProvider.getUriForFile(
    this, "$packageName.fileprovider", File(cacheDir, "photo/${System.currentTimeMillis()}.jpg")
)

val takePicture = registerForActivityResult(
    ActivityResultContracts.TakePicture()
) { success: Boolean ->
    if (success) { /* 照片已写入 photoUri，直接读取即可 */ }
}
takePicture.launch(photoUri)
```

**③ 让用户选文件/选目录（SAF，无需权限）**

```kotlin
val createDoc = registerForActivityResult(
    ActivityResultContracts.CreateDocument("application/pdf")
) { uri -> uri?.let { writePdfTo(it) } }
createDoc.launch("report.pdf")
```

## 四、常见坑与排查

### 4.1 权限回调时 Activity / Fragment 可能被系统重建

申请权限时 App 可能被系统回收（低内存），回来时 **Activity 是新的实例**，你的匿名回调如果绑在旧实例上就会丢。

- **用 Activity Result API 会自动处理**：它在 `onCreate` 前重新注册，回调能正确派发；
- 自己用 `onRequestPermissionsResult` 时，要**把"申请了什么、拿到后要干什么"存进 `savedInstanceState` 或 ViewModel**，回调里做幂等处理，别假设内存里的临时变量还在。

### 4.2 Fragment 里注册时机不对会直接崩

`registerForActivityResult` **必须在 Fragment/Activity 进入 STARTED 之前完成注册**（通常在属性初始化或 `onCreate` 中）。如果在点击事件或 `onStart` 之后才注册，会抛：

```
LifecycleOwner is attempting to register while current state is RESUMED.
```

**在 Fragment 里不要用 `requireActivity().registerForActivityResult(...)`**（容易和宿主 Activity 的注册时机打架），直接用 Fragment 自己的。

### 4.3 永久拒绝后 `launch()` 直接回调 denied

第 3.3 节说过，永久拒绝时再申请，系统**不再弹框**，直接回调 `false`。很多人误以为"再点一次就好"，结果用户狂点没反应。**必须检测并引导去设置页。**

### 4.4 升级 targetSdk 后权限没适配 → 功能静默失败 / 上架被拒

- 升到 **33**：`POST_NOTIFICATIONS` 没申请，通知全不发（不崩溃，静默失败）；
- 升到 **33**：还在用 `READ_EXTERNAL_STORAGE` 读图，拿不到数据；
- 升到 **31**：还在用 `BLUETOOTH`，扫描设备直接失败；
- 升到 **34**：前台服务没声明类型 / 没申请 `FOREGROUND_SERVICE_*`，起服务抛异常；
- 上架层面：Google Play 会检查**权限用途说明**（如 `QUERY_ALL_PACKAGES`、`MANAGE_EXTERNAL_STORAGE`、`SYSTEM_ALERT_WINDOW`），理由不充分会被拒。

**排查手段**：`adb shell dumpsys package <包名> | grep -i permission` 看授予状态；真机逐个版本（尤其 11 / 12 / 13 / 14）回归。

### 4.5 Android 10+ 后台启动 Activity 受限

从 Android 10 起，App 在后台**不能随意直接 `startActivity` 弹页面**（会被拦截）。常见踩坑：

- 通知点击跳转：**只能用 `PendingIntent`**，系统认为这是"用户主动触发"，才放行；
- 悬浮窗里想直接拉起一个 Activity：**同样受限**，通常要结合 `SYSTEM_ALERT_WINDOW` 权限或让用户先点悬浮窗（视为用户交互）；
- 定时任务里 `startActivity`：基本必失败，改用通知。

### 4.6 通知权限被拒要有降级策略

Android 13+ 用户拒绝 `POST_NOTIFICATIONS` 后，`notify()` 不报错、也不显示。**不能"死等用户开权限"**：

- 关键提醒改走**应用内横幅 / 红点 / 弹窗**；
- 在合适的时机（如用户开启某个提醒开关时）**再解释并引导申请**，别一进 App 就硬要；
- 判断是否被拒：`NotificationManagerCompat.areNotificationsEnabled()`。

### 4.7 "申请了就一定有"是错觉

用户**随时**能在设置里关掉权限。所以：

- **每次使用前都要 `checkSelfPermission`**，不要缓存在一个变量里当永久有效；
- 关权限后系统会**杀掉进程**（iOS 是这样，Android 部分场景也会重建），回来时状态要重新算。

### 4.8 隐私合规（GDPR / CCPA）场景

权限是"技术授权"，合规还要求"可拒绝、可撤回、说得清用途"：

- **同意与拒绝都要能走通**：拒绝定位后 App 核心功能仍可用（不能"不给权限就不让用"）；
- **要能撤回**：设置里关掉权限后，App 内采集行为要立即停止；
- **用途要明示**：申请前的说明、隐私政策里的描述要和实际行为一致；
- **最小必要**：只申请真正用到的权限（这也是上架审核的硬要求）。

## 五、面试追问速答

**Q1：为什么 Android 6.0 要改成运行时权限？**
安装时授权的问题有三个：一是用户根本看不懂那一屏权限也不看，等于没同意；二是"全给或全不给"太粗暴，用户因为一个定位权限要被迫同意所有权限；三是 App 一旦装上就长期持有全部权限，用户后续无法收回。改成运行时权限后，App 在真正用到某个能力时才申请，用户能结合当下场景判断是否合理，而且能随时在设置里撤销。技术上它依赖的是 Linux uid/gid 沙箱加包管理器的权限记账，Android 6 之前只是"安装时一次性发牌"，之后变成"用一张发一张、可随时收回"。

**Q2：为什么 Android 10 要分区存储（Scoped Storage）？**
Android 10 之前，App 拿到 `WRITE_EXTERNAL_STORAGE` 就能对整个共享存储读写，于是出现了大量乱建目录、把用户 SD 卡塞满、卸载后留一堆垃圾、甚至偷偷读取其它 App 文件的问题；同时很多 App 滥用存储权限采集用户数据。分区存储的核心是：App 默认只能自由访问自己的应用专属目录和通过 MediaStore 写入自己的媒体文件，访问别人的共享文件必须走 MediaStore 或 SAF，从"给一把全盘钥匙"改成"按需申请具体文件"。它同时解决了隐私和存储混乱两个问题。

**Q3：Photo Picker 解决了什么问题？**
它解决了"只想让用户挑一张图，却要申请整个相册读取权限"的权限过高问题。以前要用 `ACTION_GET_CONTENT` 或 `READ_EXTERNAL_STORAGE`，前者在部分版本仍会暴露权限，后者等于把用户整个相册都交出去。Photo Picker 由系统提供，用户选中的那个文件通过 Uri 临时授权给 App，**App 全程不需要任何存储/媒体权限**，只拿到用户主动选的内容。Android 13 起它是系统原生组件，低版本由 Activity Result API 自动兜底（走 OpenDocument 或 Play 服务提供的 backport）。

**Q4：永久拒绝（勾了不再询问 / 拒两次）怎么处理？**
先要能识别出来：`shouldShowRequestPermissionRationale` 在"从未申请"和"永久拒绝"时都返回 false，所以必须自己用一个标记记录是否申请过，标记为 true 且 rationale 为 false 且权限未授予，就是永久拒绝。处理方式是**不再调用 `launch()`**（调了也只会立刻回 denied），而是弹一个说明弹窗，引导用户点击跳到 `ACTION_APPLICATION_DETAILS_SETTINGS` 应用详情页手动开启；用户从设置页回来后在 `onResume` 里重新 `checkSelfPermission` 验证结果，成功则继续原流程，失败则给出降级方案。

**Q5：targetSdk 升级要注意哪些权限变更？**
33 起要加通知权限 `POST_NOTIFICATIONS`；33 起读图/视频/音频的媒体权限拆成 `READ_MEDIA_IMAGES/VIDEO/AUDIO`；34 起前台服务必须声明类型并申请对应 `FOREGROUND_SERVICE_*` 权限；31 起蓝牙改用 `BLUETOOTH_CONNECT/SCAN` 运行时权限、定位区分精确/大致、精确闹钟改走 `SCHEDULE_EXACT_ALARM` 设置页；30 起请求后台定位必须先有前台定位再跳设置页、分区存储强制生效。核心心法：**每次升级先跑一遍"权限变更清单"，真机回归通知、定位、媒体、蓝牙、前台服务这五类功能。**

**Q6：用户拒绝通知权限后你怎么办？**
首先接受现实——通知权限被拒不代表功能坏了。策略分三步：一是**降级**，把重要提醒改成应用内横幅、红点、角标或弹窗，保证用户仍能感知；二是**找时机再请求**，不要立刻再弹，等用户主动开启某个提醒开关、或完成一次关键操作时，结合具体收益再解释一次；三是**兜底引导**，如果确实需要，提供"去设置开启通知"的入口。全程要注意用 `NotificationManagerCompat.areNotificationsEnabled()` 判断状态，而不是假设申请过就有。

**Q7：权限和隐私合规是什么关系？**
权限是系统层面的"技术开关"，合规是法律层面的"用户对个人信息处理有知情权、同意权和撤回权"，两者是**手段与目标**的关系，但权限到手不等于合规。合规要求在权限之外还要做到：获取前有清晰告知（隐私政策 + 场景内说明）、目的正当且最小必要（只收真正用得上的数据和权限）、同意和拒绝都能正常使用核心功能、用户能随时撤回（撤回后采集必须立即停止）、数据跨境/共享要额外告知。所以"我申请到权限了"只是合法处理的第一步，不等于用户就一定合规地同意了你采集数据。
