---
title: Android 文件系统：原理 / 存储位置 / 常用 API / 常见坑
date: 2026-10-21 00:26:32
---

# Android 文件系统：原理 / 存储位置 / 常用 API / 常见坑

> 一句话定位：Android 的文件系统就是 Linux 目录树加"每个 App 一个沙箱目录"，而外部存储其实是"模拟出来的共享分区"；Android 10 起分区存储把"随便用路径读写共享盘"改成了"自己的随便写、别人的得走 MediaStore 或 SAF 拿授权"。
> 版本口径：重点覆盖 Android 10（API 29）分区存储到 Android 14（API 34）媒体权限的变化，老版本行为会写明"哪个版本之前"。

---

## 一、Android 文件系统是什么（原理）

### 1.1 底座：Linux 目录树 + 应用沙箱

Android 的文件系统完全是 Linux 那一套：一棵从 `/` 开始的目录树，权限靠 uid/gid 控制。对 App 来说只有两类存储：

- **内部存储（internal storage）**：App 私有目录，路径 `/data/data/<包名>/`（真实路径是 `/data/user/0/<包名>/`，`/data/data` 只是指向它的软链接，`0` 代表主用户）。**别的 App 看不到、不需要任何权限、卸载即删。**
- **外部存储（external storage）**：历史上是可插拔的 SD 卡，现在是**模拟出来的共享分区（emulated storage）**，路径 `/storage/emulated/0/`（`/sdcard` 是指向它的软链接，`0` 是用户编号，多用户下是 `/storage/emulated/10`）。**所有 App 都看得见，历史上人人都能读写。**

关键认知：**所谓"外部存储"在现代手机上大多是内部闪存模拟出来的一个共享分区，并不是真的 SD 卡**——但它的"共享、所有人都能看见"这个语义被完整保留了，这正是后面一堆问题的根源。

### 1.2 分区存储（Scoped Storage）：Android 10 的分水岭

在 Android 10 之前，App 只要拿到 `WRITE_EXTERNAL_STORAGE`，就能用文件路径 `/storage/emulated/0/xxx` **随便读写整个共享盘**。结果是目录乱建、垃圾文件残留、隐私数据被别家 App 偷读。

**Android 10（API 29）起引入分区存储（Scoped Storage）**：

- App 默认**只能自由访问自己的应用专属外部目录**（`/storage/emulated/0/Android/data/<包名>/`）和自己的私有目录；
- 要往共享目录（相册、下载等）写自己的媒体，**走 MediaStore**，不再需要存储权限；
- 要读别的 App 写的共享文件，通过 **MediaStore 查询**（配合 `READ_MEDIA_*` 等权限），或让用户用 **SAF** 选文件并授权；
- Android 10 允许用 `requestLegacyExternalStorage` 临时退回旧行为；**Android 11（API 30）起强制分区存储，该开关被忽略**，只有文件管理器类 App 可申请 `MANAGE_EXTERNAL_STORAGE`。

### 1.3 两种视角：`File` 路径 vs `content://` URI

这是理解 Android 文件系统最重要的一次思维切换：

| 视角 | 长这样 | 能力 | 适用 |
| --- | --- | --- | --- |
| **文件路径（File）** | `/storage/emulated/0/DCIM/a.jpg` | 直接 `File` 读写，但**只有自己沙箱和专属目录才保证有权限** | App 私有目录、应用专属外部目录 |
| **内容 URI（content://）** | `content://media/external/images/media/123` | 通过 `ContentResolver` 读写，**权限由 URI 授权（grant）决定，不需要存储权限** | 共享媒体、SAF 用户选的文件、跨 App 分享 |

一句话记忆：**在分区存储时代，"能拿到路径"不代表"能读写"，"拿到一个被授权的 content:// URI"才真正代表你有权限。** 所以新代码里凡是涉及共享文件，第一反应应该是"用 Uri 而不是用路径"。

### 1.4 几个容易搞混的路径

| 你以为的路径 | 实际含义 |
| --- | --- |
| `/data/data/<包名>/` | 软链接，真实是 `/data/user/0/<包名>/`；`0` 是主用户，多用户是 `/data/user/10/...` |
| `/sdcard/` | 软链接，指向 `/storage/emulated/0/`（或当前用户的 self 目录） |
| `/storage/emulated/0/Android/data/<包名>/` | **应用专属外部目录**，其他 App 在较新版本上访问受限，卸载即删 |
| `/storage/emulated/0/DCIM`、`Pictures`、`Download` | **共享目录**，属于用户，卸载 App 不会删，通过 MediaStore 访问 |

## 二、存储位置对照表（必背）

| 目录 | 获取方式 | 路径 | 需要权限吗 | 卸载是否删除 | 系统是否会清理 | 备份行为（Auto Backup） | 适合放什么 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **内部私有文件** | `context.filesDir` | `/data/user/0/<pkg>/files` | 不需要 | 删除 | 不清理 | **默认包含** | 用户数据、配置、小文件 |
| **内部缓存** | `context.cacheDir` | `/data/user/0/<pkg>/cache` | 不需要 | 删除 | **空间不足时系统可能清理** | **不包含** | 可重建的缓存（图片/网络缓存） |
| **不备份私有目录** | `context.noBackupFilesDir` | `/data/user/0/<pkg>/no_backup` | 不需要 | 删除 | 不清理 | **不包含** | 不希望被备份的敏感/可重建数据（token、临时态） |
| **代码缓存** | `context.codeCacheDir` | `/data/user/0/<pkg>/code_cache` | 不需要 | 删除 | 可能清理 | 不包含 | JIT/编译产物（一般由运行时管理，App 很少直接用） |
| **外部专属文件** | `getExternalFilesDir(type)` | `/storage/emulated/0/Android/data/<pkg>/files/<type>` | **Android 4.4 起不需要**（专属目录） | 删除 | 不清理 | 不包含 | 大文件、媒体、不想占内部空间的数据 |
| **外部专属缓存** | `getExternalCacheDir()` | `/storage/emulated/0/Android/data/<pkg>/cache` | 同上，不需要 | 删除 | 可能清理 | 不包含 | 大体积缓存 |
| **共享媒体目录** | MediaStore | `/storage/emulated/0/Pictures`、`DCIM`、`Download` 等 | 写自己的媒体：**Android 10+ 不需要**；读别人的：需 `READ_MEDIA_*`（13+）或 `READ_EXTERNAL_STORAGE`（≤12） | **不删除**（属于用户） | 不清理 | 不包含 | 保存到相册、导出下载文件 |
| **SAF 用户选择的位置** | `ACTION_OPEN_DOCUMENT` / `ACTION_CREATE_DOCUMENT` / `OPEN_DOCUMENT_TREE` | 任意位置，返回 `content://` URI | **不需要权限**（用户主动授权 Uri） | 不删除 | 不清理 | 不包含 | 用户选目录备份、导入导出任意文件 |

补充说明：

- **`getExternalFilesDir(type)` 里的 `type`** 常用 `Environment.DIRECTORY_PICTURES` / `MOVIES` / `MUSIC` / `DOCUMENTS` / `DOWNLOAD` 等；传 `null` 得到 files 目录本身；
- **`getExternalFilesDirs(type)`**（带 s）返回**多卷**，可以用来在外置 SD 卡上找空间；
- **Android 4.4（API 19）之前**，连应用专属外部目录都要 `WRITE_EXTERNAL_STORAGE` 才能写，之后就不需要了。

## 三、常用 API 与使用方法

### 3.1 File API 与 Context 各目录

```kotlin
// 内部私有目录：用户数据
val file = File(context.filesDir, "config.json")
file.writeText("""{"theme":"dark"}""")          // 小文件直接 writeText/readText
val text = file.readText()

// 内部缓存：可被系统清理，不要放重要数据
val cache = File(context.cacheDir, "images/cover.png")
cache.parentFile?.mkdirs()                       // 父目录不存在要先建，否则写会失败

// 不参与 Auto Backup 的私有目录
val token = File(context.noBackupFilesDir, "session.bin")

// 应用专属外部目录（Android 4.4+ 无需权限）
val picDir = context.getExternalFilesDir(Environment.DIRECTORY_PICTURES)
```

### 3.2 `openFileOutput` / `openFileInput`（返回的是私有 files 目录下的流）

```kotlin
// 等价于在 filesDir 下开一个文件流，适合写二进制
context.openFileOutput("data.bin", Context.MODE_PRIVATE).use { out ->
    out.write(payload)
}
context.openFileInput("data.bin").use { input ->
    val bytes = input.readBytes()
}
// MODE_PRIVATE：仅本 App 可读写（新项目已不再支持 MODE_WORLD_*，不要用）
```

### 3.3 临时文件与流式拷贝（java.nio）

```kotlin
// 在指定目录创建唯一临时文件（会真正创建空文件）
val tmp = File.createTempFile("download", ".tmp", context.cacheDir)

// 流式拷贝：适合大文件，避免一次性读进内存
context.contentResolver.openInputStream(uri)?.use { input ->
    FileOutputStream(tmp).use { output ->
        input.copyTo(output, bufferSize = 8 * 1024)     // Kotlin 扩展，内部循环读写
    }
}

// 用 java.nio 拷贝（API 26+ 可用 Files；API 26 以下用 FileInputStream 通道）
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    java.nio.file.Files.copy(inputStream, tmp.toPath(), java.nio.file.StandardCopyOption.REPLACE_EXISTING)
}
```

### 3.4 FileProvider：把私有文件安全地分享给别的 App

**为什么需要**：直接传 `file://` URI 给别的 App 会触发 `FileUriExposedException`（Android 7.0 起），必须用 `content://` + FileProvider 授权。

Manifest：

```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

`res/xml/file_paths.xml`（声明哪些目录可以对外暴露，`path="."` 表示该目录全部）：

```xml
<paths>
    <files-path name="files" path="." />               <!-- filesDir -->
    <cache-path name="cache" path="." />               <!-- cacheDir -->
    <external-files-path name="ext_files" path="." />  <!-- getExternalFilesDir -->
    <external-cache-path name="ext_cache" path="." />  <!-- getExternalCacheDir -->
</paths>
```

分享：

```kotlin
val file = File(cacheDir, "share/report.pdf")
val uri = FileProvider.getUriForFile(this, "$packageName.fileprovider", file)

val intent = Intent(Intent.ACTION_SEND).apply {
    type = "application/pdf"
    putExtra(Intent.EXTRA_STREAM, uri)
    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)    // 临时把读权限授给接收方 App
}
startActivity(Intent.createChooser(intent, null))
```

### 3.5 MediaStore 往相册插入图片（Android 10+ 的正确写法）

```kotlin
// ---------- Android 10（API 29）及以上：不需要存储权限 ----------
val values = ContentValues().apply {
    put(MediaStore.Images.Media.DISPLAY_NAME, "IMG_${System.currentTimeMillis()}.jpg")
    put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg")
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        // 用 RELATIVE_PATH 指定"相对共享目录的路径"，不要再写绝对路径
        put(MediaStore.Images.Media.RELATIVE_PATH, "${Environment.DIRECTORY_PICTURES}/MyApp")
        // IS_PENDING=1：先占位，写完再置 0，避免半成品被别人（相册）看到
        put(MediaStore.Images.Media.IS_PENDING, 1)
    }
}

val collection = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
    MediaStore.Images.Media.getContentUri(MediaStore.VOLUME_EXTERNAL_PRIMARY)
} else {
    MediaStore.Images.Media.EXTERNAL_CONTENT_URI
}

val resolver = contentResolver
val uri = resolver.insert(collection, values) ?: return
try {
    resolver.openOutputStream(uri)?.use { out ->
        bitmap.compress(Bitmap.CompressFormat.JPEG, 90, out)
    }
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        values.clear()
        values.put(MediaStore.Images.Media.IS_PENDING, 0)   // 写入完成，解除 pending
        resolver.update(uri, values, null, null)
    }
} catch (e: Exception) {
    resolver.delete(uri, null, null)     // 失败要清理掉这条空记录，否则相册里会出现坏图
    throw e
}
```

Android 9 及以下的老写法（**需要 `WRITE_EXTERNAL_STORAGE`，且要自己给出绝对路径**）：

```kotlin
val dir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_PICTURES)
val file = File(dir, "IMG_${System.currentTimeMillis()}.jpg")
val values = ContentValues().apply {
    put(MediaStore.Images.Media.DISPLAY_NAME, file.name)
    put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg")
    put(MediaStore.Images.Media.DATA, file.absolutePath)   // DATA 列已废弃，仅 ≤28 使用
}
contentResolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values)
```

### 3.6 SAF：让用户选文件/选目录，并持久化授权

```kotlin
// 选单个文件：ACTION_OPEN_DOCUMENT
val openDoc = registerForActivityResult(ActivityResultContracts.OpenDocument()) { uri ->
    uri ?: return@registerForActivityResult
    // 想让 Uri 在 App 重启后仍可用，必须申请持久化读权限
    contentResolver.takePersistableUriPermission(uri, Intent.FLAG_GRANT_READ_URI_PERMISSION)
    contentResolver.openInputStream(uri)?.use { input ->
        File(context.filesDir, "imported.dat").outputStream().use { input.copyTo(it) }
    }
}
openDoc.launch(arrayOf("image/*", "application/pdf"))   // 传 MIME 数组，传 */* 表示任意类型

// 选目录（用于备份）：ACTION_OPEN_DOCUMENT_TREE，返回的是目录树 Uri
val openTree = registerForActivityResult(
    ActivityResultContracts.OpenDocumentTree()
) { treeUri ->
    treeUri ?: return@registerForActivityResult
    contentResolver.takePersistableUriPermission(
        treeUri,
        Intent.FLAG_GRANT_READ_URI_PERMISSION or Intent.FLAG_GRANT_WRITE_URI_PERMISSION
    )
    // 用 DocumentFile 把树 Uri 当"目录"来操作，别拼字符串路径
    val root = DocumentFile.fromTreeUri(context, treeUri)
    val out = root?.createFile("application/json", "backup.json")
    out?.uri?.let { contentResolver.openOutputStream(it)?.use { os -> os.write(jsonBytes) } }
}
openTree.launch(null)

// 导出新文件：ACTION_CREATE_DOCUMENT（ActivityResultContracts.CreateDocument(mime)）
```

**要点**：`takePersistableUriPermission` 只能对 SAF 返回的（带 `FLAG_GRANT_PERSISTABLE_URI_PERMISSION` 的）Uri 生效，对 Photo Picker 返回的 Uri 通常不能持久化；重新读取持久化过的目录时用 `contentResolver.persistedUriPermissions` 查。

### 3.7 缓存统计与清理（只清自己能清的目录）

```kotlin
// 递归统计一个目录的大小
fun dirSize(dir: File?): Long =
    dir?.listFiles()?.sumOf { if (it.isDirectory) dirSize(it) else it.length() } ?: 0L

// 只清理 cacheDir / externalCacheDir；绝不能去删 filesDir 或共享目录
fun clearCache(dir: File?) {
    dir?.listFiles()?.forEach { f -> if (f.isDirectory) clearCache(f) else f.delete() }
}

val cacheSize = dirSize(context.cacheDir) + dirSize(context.externalCacheDir)
```

### 3.8 查询可用存储空间

```kotlin
// 方式一：StatFs 查某个路径所在分区的剩余空间
val stat = StatFs(context.filesDir.path)
val availableBytes = stat.availableBytes
val totalBytes = stat.totalBytes

// 方式二：StorageManager 的"可分配空间"（会把系统可回收的缓存也算进去，更准）
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    val sm = context.getSystemService(StorageManager::class.java)
    val uuid = sm.getUuidForPath(context.filesDir)
    val allocatable = sm.getAllocatableBytes(uuid)      // 当前还能腾出多少
    if (allocatable >= requiredBytes) {
        sm.allocateBytes(uuid, requiredBytes)           // 预占空间，减少写入中途失败
    }
}
```

## 四、使用场景

| 场景 | 推荐位置 / 做法 | 关键点 |
| --- | --- | --- |
| **图片/视频缓存** | `cacheDir` 或 `getExternalCacheDir()` | 会被系统清理，必须能重新下载；大图放外部专属缓存省内部空间 |
| **用户数据持久化（DB/配置）** | `filesDir`、`databases` 目录、SharedPreferences / DataStore | 不能放 `cacheDir`（会被清）；`filesDir` 默认进 Auto Backup |
| **导出文件给用户** | SAF `ACTION_CREATE_DOCUMENT`，或 MediaStore `Download` | 让用户自己选位置，App 不需要存储权限 |
| **跨应用分享** | FileProvider + `content://` | 必须 `FLAG_GRANT_READ_URI_PERMISSION`；不能用 `file://` |
| **保存到相册** | MediaStore（`RELATIVE_PATH` + `IS_PENDING`） | Android 10+ 不需要权限；写完必须把 `IS_PENDING` 置 0 |
| **用户选择目录备份** | SAF `OPEN_DOCUMENT_TREE` + DocumentFile | `takePersistableUriPermission` 让授权跨重启有效 |
| **日志文件** | `filesDir/logs` 或 `getExternalFilesDir(LOG)` | **不要放 cacheDir**（随时被清）；要自己做轮转和大小上限 |
| **大文件下载临时文件** | 先下到 `cacheDir/xxx.tmp`，完成后再移到目标位置 | 先写临时文件再 `renameTo`，避免用户拿到半成品 |

## 五、常见坑与排查

### 5.1 `file://` 跨 App 传递直接崩：`FileUriExposedException`

Android 7.0（API 24）起，任何把 `file://` URI 通过 Intent 传出去的行为都会抛 `FileUriExposedException`。原因是 `file://` 会让接收方绕过权限体系直接读文件。**统一改用 FileProvider 生成 `content://` URI 并加 `FLAG_GRANT_READ_URI_PERMISSION`。** 要临时放宽也可以用 `StrictMode.setVmPolicy`，但那是掩盖问题，正确解法是 FileProvider。

### 5.2 `WRITE_EXTERNAL_STORAGE` 在 Android 13+ 对媒体已失效

- Android 13（API 33）起，读共享媒体要用 `READ_MEDIA_IMAGES` / `READ_MEDIA_VIDEO` / `READ_MEDIA_AUDIO`，`READ_EXTERNAL_STORAGE` 不再覆盖这些类型；
- **写入相册（自己创建的媒体）从 Android 10 起就不再需要任何存储权限**，之前靠 `WRITE_EXTERNAL_STORAGE` 的写法在 33+ 上是多余的、申请也可能不被授予。

### 5.3 Android 11 起不能直接写 `/sdcard` 根目录

分区存储强制生效后，在 `/storage/emulated/0/` 根目录或共享目录里**随意 `mkdir` / 写文件会失败**（`EACCES`）。自己的东西写进 `getExternalFilesDir`，要放到共享区就走 MediaStore 或让用户用 SAF 选位置。

### 5.4 `cacheDir` 会被系统清理，不能放重要数据

磁盘紧张时系统会清理 App 的 `cacheDir`（外部缓存同样可能被清）。**重要数据（用户配置、未上传的草稿）必须放 `filesDir` 或 `noBackupFilesDir`**，缓存目录只放"丢了能重新拿"的内容。

### 5.5 `filesDir` 与 `noBackupFilesDir` 的备份差异

Auto Backup 默认**包含 `filesDir`、`databases`、SharedPreferences 等**，但**不包含 `cacheDir` 和 `noBackupFilesDir`**。有个常见坑：Auto Backup 有**约 25MB 的每 App 配额**，超出会被截断导致恢复不完整，所以 **Log、大缓存、可重建的媒体文件要排除掉**——用 `android:fullBackupContent`（Android 11 及以下）和 Android 12+ 的 `android:dataExtractionRules` 声明规则，或者干脆放到 `noBackupFilesDir`。

### 5.6 路径拼接必须用 `File.separator` / `File(parent, child)`

Android 的路径分隔符是 `/`，但硬编码 `"$dir/$name"` 在跨平台（如单元测试跑在 JVM 上）时可能出问题。**统一用 `File(parent, child)` 或 `File(parent, child).path`**，不要自己拼字符串。

### 5.7 `renameTo` 跨文件系统会失败，原子写要靠"同分区临时文件"

`File.renameTo()` 只有在**源和目标在同一文件系统**时才是原子的，跨分区（比如从 `cacheDir` 移到 `/sdcard`）会直接失败并返回 `false`。**正确做法**：在目标目录下先写一个 `.tmp` 文件，写完再 `renameTo` 成正式名，这样要么看到完整的新文件、要么看到旧文件，不会出现半截文件。注意 `renameTo` 返回 `false` 时要回退到"拷贝后删除"。

```kotlin
val target = File(context.filesDir, "config.json")
val tmp = File(context.filesDir, "config.json.tmp")     // 和目标同目录，保证同分区
tmp.writeText(json)
if (!tmp.renameTo(target)) {                            // 兜底：拷贝 + 删除
    tmp.copyTo(target, overwrite = true)
    tmp.delete()
}
```

### 5.8 文件名非法字符与长度限制导致创建失败

- 文件名里带 `/`、`\`、`:`、`*`、`?`、`"`、`<`、`>`、`|` 等字符，或包含控制字符，会导致创建失败；
- 文件名过长（超过文件系统单段名限制，通常 255 字节）也会失败；
- 从网络/用户输入拿到的名字**必须先做清洗**（去掉非法字符、截断长度、去掉首尾空格和 `.`），否则表现为 `FileNotFoundException` 或 `IOException`。

### 5.9 多用户 / 多进程下路径不同

- 多用户设备上私有目录是 `/data/user/<userId>/<包名>/`，主用户是 `0`，工作资料/访客用户可能是 `10` 等；**不要硬编码 `/data/user/0/`**，永远用 `context.filesDir` 等 API 获取；
- 多进程 App（`:remote` 进程）共享同一个 uid，但**各自的 `Context` 指向同一个沙箱目录**，所以文件是可见的，要注意并发写冲突，用文件锁或让单进程负责写。

### 5.10 写之前先检查可用空间

大文件写入前应先查 `StatFs.availableBytes` 或 `StorageManager.getAllocatableBytes`，并**留出余量**（不要卡着剩余空间写），否则写到一半 `ENOSPC` 会留下残缺文件。推荐"先占位（`allocateBytes`）→ 写临时文件 → 成功后 rename"的流程。

### 5.11 Scoped Storage 的心智模型：先问"用路径还是用 URI"

遇到"读/写文件"的需求，先按这个顺序判断：

1. **是不是自己的私有/专属目录？** → 用 `File` 路径，随便读写；
2. **是不是要写共享区（相册/下载）自己的媒体？** → 用 MediaStore，写自己的**不用权限**；
3. **是不是要读/改别人的共享文件，或用户指定位置的文件？** → 用 `content://` URI（MediaStore 查询或 SAF 授权）；
4. **是不是要分享给别的 App？** → FileProvider 生成 `content://` + 授权 flag。

只要记住"**共享区一律用 URI，别指望路径**"，分区存储相关的坑就基本躲开了。

## 六、面试追问速答

**Q1：Android 10 分区存储到底改变了什么？**
它改变了"谁能读写共享存储"的规则。Android 10 之前，拿到 `WRITE_EXTERNAL_STORAGE` 就等于拿到整个共享盘的读写权，App 可以随意建目录、读别人文件、卸载留垃圾。分区存储之后，每个 App 在共享盘上默认只有三块能力：自己私有目录、自己的应用专属外部目录（`Android/data/<包名>/`）、以及通过 MediaStore 写入的、属于自己创建的媒体文件；要访问别人的共享文件必须走 MediaStore 查询或 SAF 授权。它同时解决了隐私泄露和存储混乱，代价是老代码里大量"绝对路径直接读写 sdcard"的写法全部要改。

**Q2：为什么推荐 `content://` 而不是文件路径？**
因为在分区存储下，**路径只是一个"位置描述"，不携带任何访问授权**——你知道 `/storage/emulated/0/DCIM/a.jpg` 这个路径，不代表你能打开它。而 `content://` URI 是由系统/提供方签发的"访问凭据"，它背后带着授权信息：MediaStore 的 URI 表示"这是你被允许访问的媒体"，SAF 的 URI 表示"用户主动授权了这个文件"，FileProvider 的 URI 表示"我主动授权给你读这一份"。另外 URI 还能跨进程、跨 App、跨用户使用，且不暴露真实路径，安全性和可移植性都更好。

**Q3：FileProvider 原理是什么？**
FileProvider 是一个 `ContentProvider`，它把 App 私有目录（在 `file_paths.xml` 里声明的那些路径）**映射成一棵虚拟的 content 目录树**。当 App 调 `FileProvider.getUriForFile()` 时，它会根据真实文件路径反查出对应的虚拟路径，生成一个 `content://<authority>/<虚拟路径>` 形式的 URI。别的 App 拿到这个 URI 后，通过 `ContentResolver` 请求 FileProvider 打开文件，FileProvider 校验发起方是否被授权（靠 Intent 上带的 `FLAG_GRANT_READ_URI_PERMISSION` 等），通过后代为读写真实文件。所以接收方**永远看不到真实路径**，权限粒度精确到单个文件，且可以随时收回。

**Q4：Android 13 保存图片到相册需要什么权限？**
**不需要任何存储权限。** 从 Android 10（API 29）分区存储开始，App 通过 MediaStore 插入**自己创建的**媒体文件（相册图片、视频）就无需 `WRITE_EXTERNAL_STORAGE`。正确写法是：`ContentValues` 里设置 `DISPLAY_NAME`、`MIME_TYPE`、`RELATIVE_PATH`，把 `IS_PENDING` 置 1 → `insert` 拿到 URI → 写入数据 → 再把 `IS_PENDING` 置 0。只有**读取别的 App 创建的媒体**才需要权限，Android 13+ 用 `READ_MEDIA_IMAGES` / `READ_MEDIA_VIDEO`，Android 12 及以下用 `READ_EXTERNAL_STORAGE`。

**Q5：原子写文件怎么做？**
核心思路是"要么完整的新内容，要么保持旧内容，绝不出现写坏一半的文件"。做法是在**目标文件的同一目录**（保证同一文件系统）先写一个临时文件（如 `config.json.tmp`），把全部内容写完并 `flush`/`close` 后，再用 `renameTo` 把临时文件改名成正式文件名。同一文件系统上的 rename 是原子操作，所以外界要么看到旧文件、要么看到完整新文件。注意两点：临时文件**必须和目标同目录**（跨分区 rename 会失败，那就只能退化成"拷贝再删"，不再是原子的）；`renameTo` 返回 `false` 时要有兜底逻辑。

**Q6：缓存目录和私有目录怎么选？**
判断标准就一条：**这个数据丢了会不会造成损失？** 会损失（用户配置、草稿、数据库、日志）→ 放私有目录 `filesDir`（或 `noBackupFilesDir`、`databases`）；不会损失、能重新获取（图片缓存、网络响应缓存、可重下的临时文件）→ 放 `cacheDir` 或 `getExternalCacheDir()`。区别在于 `cacheDir` 在磁盘紧张时**会被系统主动清理**，而 `filesDir` 不会被清，并且默认参与 Auto Backup。所以"为了省内部空间把用户数据挪到 cacheDir"是典型的错误优化。

**Q7：卸载后哪些数据会保留？**
App 卸载时，**私有目录（`/data/data/<包名>/` 整个）、应用专属外部目录（`/storage/emulated/0/Android/data/<包名>/` 整个）都会随卸载删除**。会**保留**的是那些已经放进"用户共享区"的文件，比如通过 MediaStore 保存到相册的图片、通过 SAF 由用户选择位置导出的文件——这些文件的所有权属于用户，不属于 App，卸载不会动它们。同理，`cacheDir`、`filesDir`、`noBackupFilesDir` 这些都在沙箱内，一并删除。额外的坑是：**Auto Backup 备份过的 `filesDir` 数据，在新设备/重装时可能被恢复**，如果不想被恢复，要放 `noBackupFilesDir` 或用备份规则排除。
