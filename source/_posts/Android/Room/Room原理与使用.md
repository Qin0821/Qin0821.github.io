---
title: Room：原理 / 使用场景 / 常用 API / 常见坑
date: 2026-10-21 00:26:03
---

# Room：原理 / 使用场景 / 常用 API / 常见坑

> 一句话定位：Room 是 Jetpack 提供的**SQLite 之上的 ORM 与类型安全封装**——SQL 写在注解里，编译期就校验语法和字段映射，运行时通过 `SupportSQLiteOpenHelper` 操作 SQLite，查询返回 `Cursor` 再由生成的代码映射成实体。
> 提示：本文讲通用原理与常用写法，不绑定具体版本号；`@Upsert`、`autoMigrations` 等能力需要较新的 Room 版本，用前请确认你项目里的版本。

---

## 一、Room 是什么（原理）

**一句话：Room = 编译期注解处理 + 生成的样板代码 + 一层 SQLite 封装，把"写 SQL 容易写错"这件事尽量堵在编译期。**

### 1.1 核心链路

```
@Entity / @Dao / @Query 注解
        ↓  编译期（KSP 或 KAPT）注解处理器解析
生成 XxxDao_Impl、XxxDatabase_Impl          ← SQL 语法、字段映射、返回类型在这里校验
        ↓  运行时
RoomDatabase → SupportSQLiteOpenHelper → SQLite（最终落在 .db 文件）
        ↓
查询返回 Cursor → 生成的代码把它映射成实体对象 / 列表
        ↓
@Query 返回 Flow / LiveData → 靠 InvalidationTracker 感知表变化后重新查询
```

### 1.2 四个必须记住的点

**① 编译期校验是 Room 最大的价值**

- SQL 语句**语法错误**、引用了不存在的表/列、返回类型和查询列对不上……这些在**编译期**就报错（生成 `Dao_Impl` 时失败），不用等到运行时崩。
- 生成代码里的列名和实体字段的映射也是编译期确定的，避免了手写 `Cursor.getColumnIndex` 拼错字段的下场。
- 这就是为什么 Room 能取代大量手写 `SQLiteOpenHelper` 样板代码。

**② 运行时到底怎么跑**

Room 并不自己实现数据库引擎，底层还是 Android 系统的 SQLite。它通过 `SupportSQLiteOpenHelper` 这一层接口访问，好处是：

- 能屏蔽不同 Android 版本的差异；
- 能接入加密（SQLCipher）等第三方实现；
- `RoomDatabase.Builder` 里的 `openHelperFactory` 就是换这一层的入口。

**③ `Flow` / `LiveData` 查询怎么感知变化（面试高频）**

`@Query` 返回 `Flow<List<User>>` 时，Room 不是在做"增量推送"，而是**失效重查**：

1. Room 用 `InvalidationTracker` 监听相关表（`@Query` 涉及的表会被登记）；
2. 任何写操作（Insert/Update/Delete）提交后，该表被标记为"失效"；
3. `InvalidationTracker` 触发重新执行原来的查询 SQL；
4. 新结果通过 `Flow` 发射出来。

所以本质上：**"写 → 表失效 → 重跑查询"**，不是数据库变了只推变化的那一行。这也解释了为什么"一次写可能触发多次 emit"——后面常见坑里说。

**④ WAL 提升并发**

- 默认（rollback journal）模式下，写会阻塞读；
- 开启 **WAL（Write-Ahead Logging）** 后，读写可以并发，只有"写与写"会互斥。Android 上可以通过给 `RoomDatabase.Builder` 设置 `setJournalMode(JournalMode.WRITE_AHEAD_LOGGING)`（Android 9+ 上 WAL 是默认行为之一）来获得更好的并发读写表现。
- 结论：**读多写多、并发访问频繁的场景，WAL 基本是必开项。**

---

## 二、使用场景

| 场景 | 为什么用它 | 关键点 |
| --- | --- | --- |
| 本地缓存 / 离线可用 | 强类型 + 编译期校验，改字段不容易漏 | 实体和网络 DTO 分开，避免耦合 |
| Feed 本地分页缓存 | 用 `Paging` + Room 的 `PagingSource` 做本地分页 | 注意列表更新用 `PagingSource.invalidate()` |
| 消息本地存储 | 需要事务、索引、按会话查询 | `@Transaction` + 索引（会话 ID、时间） |
| 用户设置 / 配置 | 结构简单的键值或小表 | 量小也可考虑 DataStore，视需求 |
| 埋点暂存（离线补传） | 先落库、联网后批量上传 | 加"已上传"标记 + 定期清理 |
| 关联查询（一对多） | `@Relation` + `@Transaction` 一次查出对象树 | 必须配 `@Transaction` 保证一致 |
| 全文检索 | FTS（`@Fts4`）支持关键词检索 | 有条件使用，注意 FTS 表结构独立 |

### 不适合的场景

- **存大二进制（图片、音视频）**：应该存文件、数据库只存路径/URI，否则 db 文件爆炸、读写变慢。
- **超大规模数据的复杂分析查询**：SQLite 不是分析型数据库，量级上去要考虑分表或换存储。
- **只需要简单键值**：DataStore 更轻，杀鸡不必用牛刀。
- **需要多进程同时写**：Room 默认不保证多进程安全，多进程场景要额外处理（或用单进程架构）。

---

## 三、常用 API 与使用方法

### 3.1 三件套：Entity + Dao + Database

```kotlin
// ① Entity：表结构 + 索引 + 外键
@Entity(
    tableName = "user",
    indices = [
        Index(value = ["userId"]),                       // 普通索引
        Index(value = ["email"], unique = true)          // 唯一索引
    ]
)
data class UserEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "userId") val userId: String,
    val name: String,
    val email: String,
    @Ignore val isSelected: Boolean = false              // 不落库的字段
)

// 外键 + 外键索引（外键列一定要建索引，否则查询慢）
@Entity(
    tableName = "post",
    foreignKeys = [
        ForeignKey(
            entity = UserEntity::class,
            parentColumns = ["id"],
            childColumns = ["authorId"],
            onDelete = ForeignKey.CASCADE                    // 父删子也删
        )
    ],
    indices = [Index(value = ["authorId"])]
)
data class PostEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val authorId: Long,
    val title: String,
    val content: String
)
```

```kotlin
// ② Dao：查询 / 插入 / 更新 / 删除
@Dao
interface UserDao {

    @Query("SELECT * FROM user WHERE userId = :userId LIMIT 1")
    suspend fun getUser(userId: String): UserEntity?

    // 返回 Flow：表变化后自动重新查询（失效重查）
    @Query("SELECT * FROM user ORDER BY name ASC")
    fun observeUsers(): Flow<List<UserEntity>>

    // 分页
    @Query("SELECT * FROM post ORDER BY id DESC")
    fun pagingPosts(): PagingSource<Int, PostEntity>

    @Insert(onConflict = OnConflictStrategy.REPLACE)     // 冲突策略：先删后插
    suspend fun insert(user: UserEntity): Long

    @Upsert                                               // 存在就更新、不存在就插入
    suspend fun upsert(users: List<UserEntity>)

    @Update
    suspend fun update(user: UserEntity)

    @Delete
    suspend fun delete(user: UserEntity)

    // 事务：多步操作原子执行
    @Transaction
    suspend fun replaceAll(users: List<UserEntity>) {
        deleteAll()
        insertAll(users)
    }

    @Query("DELETE FROM user")
    suspend fun deleteAll()

    @Insert
    suspend fun insertAll(users: List<UserEntity>)
}
```

```kotlin
// ③ Database：数据库入口 + 版本管理
@Database(
    entities = [UserEntity::class, PostEntity::class],
    version = 1,
    exportSchema = true            // 建议开启，产物用于比对 schema 变化
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
    abstract fun postDao(): PostDao
}
```

### 3.2 一对多：`@Embedded` / `@Relation`

```kotlin
// 「用户 + 他的所有帖子」：一次查出对象树
data class UserWithPosts(
    @Embedded val user: UserEntity,
    @Relation(parentColumn = "id", entityColumn = "authorId")
    val posts: List<PostEntity>
)

@Dao
interface UserWithPostsDao {
    // ⚠️ 必须加 @Transaction，否则两次查询之间数据可能被改动导致不一致
    @Transaction
    @Query("SELECT * FROM user")
    suspend fun getUsersWithPosts(): List<UserWithPosts>
}
```

> `@Relation` 会分两条 SQL 查（父查询 + 子查询），再在内存里拼装，所以**它不会自动是原子的**，靠 `@Transaction` 保证。

### 3.3 `@TypeConverter`：复杂类型转换

```kotlin
class Converters {
    private val fmt = SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.getDefault())

    @TypeConverter
    fun dateToLong(date: Date?): Long? = date?.time

    @TypeConverter
    fun longToDate(value: Long?): Date? = value?.let { Date(it) }

    @TypeConverter
    fun listToString(list: List<String>?): String? = list?.joinToString(",")

    @TypeConverter
    fun stringToList(value: String?): List<String>? =
        value?.takeIf { it.isNotEmpty() }?.split(",")
}

@Database(entities = [UserEntity::class, PostEntity::class], version = 1, exportSchema = true)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() { /* ... */ }
```

### 3.4 构建数据库 + 迁移 + 单例

```kotlin
// 单例：用 Koin / Hilt 提供，避免多处 new（多实例写冲突）
val db = Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .addMigrations(MIGRATION_1_2)
    .setJournalMode(RoomDatabase.JournalMode.WRITE_AHEAD_LOGGING)  // WAL
    // .fallbackToDestructiveMigration()   // ⚠️ 只在开发期用，会清库丢数据
    .build()

// 手写 Migration
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE user ADD COLUMN nickname TEXT")
    }
}
```

> 较新版本 Room 支持 `@AutoMigration(from = 1, to = 2)` 自动生成简单迁移（如加列），但涉及数据搬移、删列、改名等复杂变更仍需手写 `Migration`。

### 3.5 注入单例并接入 Repository + Flow

```kotlin
// Koin
val dataModule = module {
    single { Room.databaseBuilder(androidContext(), AppDatabase::class.java, "app.db").build() }
    single { get<AppDatabase>().userDao() }
    single { UserRepository(get()) }
}

class UserRepository(private val dao: UserDao) {
    fun observeUsers(): Flow<List<UserEntity>> = dao.observeUsers()

    suspend fun refresh() {
        // 网络请求 → 落库（IO 调度在 Repository 或更外层切换）
        val remote = api.fetchUsers()
        dao.upsert(remote.map { it.toEntity() })
    }
}

class HomeViewModel(private val repo: UserRepository) : ViewModel() {
    val users: StateFlow<List<UserEntity>> =
        repo.observeUsers().stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
}
```

### 3.6 注解 / API 速查表

| 注解 / API | 作用 |
| --- | --- |
| `@Entity` / `@PrimaryKey` / `@ColumnInfo` / `@Ignore` | 定义表、主键、列名映射、忽略字段 |
| `@Entity(indices = [...])` / `ForeignKey` | 索引 / 外键约束 |
| `@Dao` / `@Query` / `@Insert` / `@Update` / `@Delete` / `@Upsert` | 数据访问与增删改 |
| `@Transaction` | 多步操作原子化（`@Relation` 必须配） |
| `@Embedded` / `@Relation` | 内嵌对象 / 一对多关系 |
| `@TypeConverter` / `@TypeConverters` | 复杂类型 ↔ SQLite 支持类型的转换 |
| `OnConflictStrategy.REPLACE` / `IGNORE` / `ABORT` | 冲突策略 |
| `@Database` / `version` / `exportSchema` | 数据库定义与版本 |
| `RoomDatabase.Builder` / `databaseBuilder` / `addMigrations` / `setJournalMode` / `openHelperFactory` | 构建、迁移、WAL、加密 |
| `PagingSource` | 配合 Paging 做本地分页 |

---

## 四、常见坑与排查

### 4.1 主线程访问崩溃

```
Cannot access database on the main thread...
```

- 别用 `allowMainThreadQueries()`（那只是把炸弹藏起来，会卡 UI）；
- 正确姿势：`@Dao` 方法声明成 `suspend`，或用 `withContext(Dispatchers.IO)`；Room 的 suspend 查询会自动切到内部执行器，不会阻塞主线程。

### 4.2 数据库必须单例

`RoomDatabase` 实例**应该全局唯一**。多处 `databaseBuilder` 会创建多个连接池，WAL 下多写者会产生锁竞争甚至写冲突。做法：用 Koin `single` / Hilt `@Singleton` 提供同一个实例。

### 4.3 漏写 Migration 导致升级崩溃

改了 `version` 却没给对应 `Migration`，启动就抛 `IllegalStateException: A migration from X to Y was required but not found`。

- **千万不要**随手加 `fallbackToDestructiveMigration()` 图省事——它会**删库重建、数据全丢**，只能用于开发期；
- 正确做法：改 schema 就补迁移；有条件的话对每个版本写迁移测试（Room 提供 `MigrationTestHelper`）。

### 4.4 `REPLACE` 冲突策略的隐藏行为

`OnConflictStrategy.REPLACE` 在遇到冲突时，是**先 DELETE 再 INSERT**，不是 UPDATE。后果：

- 如果这张表被别的表用 `ForeignKey.CASCADE` 引用，**删旧行会级联删掉子表数据**；
- 主键会重新分配（`autoGenerate` 情况下可能变化）。

需要"存在则更新"的语义，优先用 `@Upsert` 或 `@Update`，别乱用 `REPLACE`。

### 4.5 缺索引导致慢查询

`WHERE` / `JOIN` / `ORDER BY` 用到的列没建索引，数据量一大就是全表扫描。排查方法：

```sql
EXPLAIN QUERY PLAN SELECT * FROM post WHERE authorId = 1;
```

看输出是 `SCAN TABLE`（全表扫）还是 `SEARCH TABLE ... USING INDEX`（走索引）。外键列**务必**建索引，否则父表删改时会全表扫子表。

### 4.6 `Flow` 查询可能重复 emit

前面说过，`Flow` 查询是"失效重查"，**不会自动去重**：一次事务里连续写多行，可能触发多次 emit，但查询结果是同一个。UI 层如果对相同数据也重建，会闪。

解决：`distinctUntilChanged()`（配合 `data class` 的 `equals`），或者用 `conflate()` 合并高频发射。

### 4.7 `@TypeConverter` 只管简单类型

它适合 `Date`、`List<String>`、枚举这类能映射到 `TEXT`/`INTEGER`/`REAL`/`BLOB` 的类型。**不要**用它序列化整个复杂对象树——那是反模式（查询、索引都没法用）。复杂对象应该拆成多表 + 关系。

### 4.8 `@Relation` 没配 `@Transaction`

`@Relation` 内部是"先查父、再查子"两条 SQL，中间若有写操作，两次结果可能不一致（查到的父子对不上）。**加 `@Transaction`** 保证同一个事务内读写一致。

### 4.9 实体不要直接当网络 DTO

- 后端字段一变，数据库 schema 也得跟着升版本，改动面爆炸；
- 后端下发的字段可能不需要落库、落库的字段可能需要额外结构；
- 正确做法是三层模型分离：**网络 DTO（Retrofit）→ 领域模型（Domain）→ 数据库 Entity（Room）**，在不同层做映射。别图省事一个类打天下。

### 4.10 数据库加密

Room 本身**不提供加密**。需要对 db 文件加密，要引入 SQLCipher，通过 `openHelperFactory(SupportFactory(password))` 把 Room 的 open helper 换成 SQLCipher 的实现。

### 4.11 其它高频杂项

- `exportSchema = true` 时记得配置 schema 输出目录，否则会有编译警告；
- 大列表 `@Insert` 建议分批（`insertAll` 里 chunk），避免单个事务过大；
- 计数用 `SELECT COUNT(*)` 而不是查全表再 `.size`。

---

## 五、面试追问速答

**Q1：Room 相对原生 SQLite 好在哪？**
原生 SQLite 要自己写 `SQLiteOpenHelper`、拼 SQL 字符串、手写 `Cursor` 到对象的映射，而且 SQL 写错、字段拼错只有运行时才崩。Room 把这些变成**编译期校验**：注解处理器生成 `Dao_Impl` 时会验证 SQL 语法、表/列是否存在、返回类型是否匹配，错了直接编译失败；同时自动生成 Cursor 映射、事务、`Flow`/`LiveData` 支持。代价是多了一层注解和生成代码。

**Q2：编译期校验怎么做到的？**
Room 用 KSP（或 KAPT）注解处理器在编译期解析 `@Entity`、`@Dao`、`@Query`。处理器内部会真正解析 SQL 语句（判断语法、表名列名、参数绑定），并检查方法返回类型和查询列是否兼容，然后生成 `XxxDao_Impl` 的 Kotlin/Java 代码。任何一步不通过就编译报错。所以校验发生在**编译期**而不是运行时。

**Q3：`Flow` 查询怎么感知数据变化？**
靠 `InvalidationTracker`。Room 会记录每条 `@Query` 依赖了哪些表，当这些表发生写操作并提交后，Tracker 将表标记为失效，然后**重新执行原来的查询 SQL**（失效重查），把新结果通过 `Flow` 发射出去。它不是"只推送变化的那一行"，所以在一次事务里写多行可能触发多次 emit，需要 `distinctUntilChanged` 去重。

**Q4：数据库升级方案怎么设计？**
每次 schema 变更就 `version + 1`，并为旧版本到新版本提供 `Migration`（用 `execSQL` 做 `ALTER TABLE` 或建新表迁数据）。简单加列可以考虑 `@AutoMigration`。要开启 `exportSchema` 保留 schema 历史，并写迁移测试。**严禁**用 `fallbackToDestructiveMigration` 掩盖问题（会丢全部数据）。

**Q5：多线程并发写怎么保证安全？**
一是保持 `RoomDatabase` **单例**，共享同一个连接池；二是开启 **WAL**，让读写可并发（只有写与写互斥）；三是用 `@Transaction`/`withTransaction` 把多步写包成原子事务；四是通过 suspend DAO 或 `Dispatchers.IO` 避免主线程访问。多进程场景 Room 默认不安全，需要单进程架构或额外处理。

**Q6：为什么实体不应直接作为接口 DTO？**
因为两者变化的原因和频率不同：接口 DTO 跟随后端协议变，Entity 跟随本地表和查询需求变。合并成一个类会导致后端一改字段就得改数据库 schema、升版本、写迁移，还会把"仅用于网络的字段"和"仅用于本地的字段"混在一起。所以应该做三层模型（DTO / Domain / Entity）并在边界处映射，用一点样板代码换取两边的独立演进。
