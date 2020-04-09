## greenDao、Room对比

### CURD效率对比

十万条数据进行批量的插入、查询、删除操作，取五次操作平均值，得出如下统计数据

操作(单位: ms)|greenDao|Room
:-:|:-:|:-:
insertAll|1651.2|1603.8
selectAll|874|1287.4
deleteAll|669.8|49.5

### 数据库版本升级对比

#### greenDao

greenDao的数据库版本可以在app的build.gradle中指定，如：

```groovy
android {
 
    greendao {
        //指定数据库schema版本号，迁移等操作会用到
        schemaVersion 1
        //DaoSession、DaoMaster以及所有实体类的dao生成的目录
        //daoPackage 包名
        daoPackage 'com.example.liaoshouwang.dbdemo.greendaodb'
        //工程路径
        targetGenDir 'src/main/java'
    }
}
```

处理升级需继承 DaoMaster.OpenHelper，重写onUpgrade方法，在onUpgrade方法中实现所需的升级操作。 greenDao中有默认的OpenHelper实现类，其内部实现如下：

```java
//默认每次升级会把数据库中的数据清除
/** WARNING: Drops all table on Upgrade! Use only during development. */
    public static class DevOpenHelper extends OpenHelper {
        public DevOpenHelper(Context context, String name) {
            super(context, name);
        }

        public DevOpenHelper(Context context, String name, CursorFactory factory) {
            super(context, name, factory);
        }

        @Override
        public void onUpgrade(Database db, int oldVersion, int newVersion) {
            Log.i("greenDAO", "Upgrading schema from version " + oldVersion + " to " + newVersion + " by dropping all tables");
            dropAllTables(db, true);
            onCreate(db);
        }
    }
```

#### Room

Room的数据库升级也需要手动添加代码处理，否则也是会默认清除数据库里的数据。

```java
Room.databaseBuilder(getApplicationContext(), MyDb.class, "database-name")
        .addMigrations(MIGRATION_1_2, MIGRATION_2_3).build();

//1，2是对应的旧、新的数据库版本
static final Migration MIGRATION_1_2 = new Migration(1, 2) {
    @Override
    public void migrate(SupportSQLiteDatabase database) {
        database.execSQL("CREATE TABLE `Fruit` (`id` INTEGER, "
                + "`name` TEXT, PRIMARY KEY(`id`))");
    }
};

static final Migration MIGRATION_2_3 = new Migration(2, 3) {
    @Override
    public void migrate(SupportSQLiteDatabase database) {
        database.execSQL("ALTER TABLE Book "
                + " ADD COLUMN pub_year INTEGER");
    }
};
```

### 自定义SQL语句的支持程度

#### greenDao

greenDao有两种办法嵌入自定义的SQL语句，分别是通过queryRaw和queryRawCreate

```java
/** A raw-style query where you can pass any WHERE clause and arguments. */
    public List<T> queryRaw(String where, String... selectionArg) {
        Cursor cursor = db.rawQuery(statements.getSelectAll() + where, selectionArg);
        return loadAllAndCloseCursor(cursor);
    }

    /**
     * Creates a repeatable {@link Query} object based on the given raw SQL where you can pass any WHERE clause and
     * arguments.
     */
    public Query<T> queryRawCreate(String where, Object... selectionArg) {
        List<Object> argList = Arrays.asList(selectionArg);
        return queryRawCreateListArgs(where, argList);
    }
```

可以看到的是，这两个方法对于自定义的SQL语句，其实是WHERE后面的那部分，让我们来看看statements.getSelectAll()返回的是什么。

```sql
SELECT T."_id",T."NAME",T."AGE" FROM "GREEN_USER" T 
```

GREEN_USER是Demo中的表名，也就是说greenDao会在我们的SQL语句前面**自动**拼接上上面的这段SQL，这段SQL其实是相对于SELECT * FROM tablename

#### Room

Room对于自定义SQL可以说是支持得非常完美，因为Room的Dao中所有的CRUD方法都是通过我们自己写SQL去实现的，比如Demo中的RoomUserDao:

```java
@Dao
interface RoomUserDao {

    @Query("SELECT * FROM ROOM_USER")
    fun getRoomUsers(): List<RoomUser>

    @Insert
    fun insertAll(users:List<RoomUser>)

    @Query("DELETE FROM ROOM_USER")
    fun deleteAll()
}
```

更复杂一点的SQL，也是没问题的：

```java
@Dao
interface MyAppReportDataDao {

    //本周每天的点击次数，intervalDay栗子：-7 day 指定的日期往前回滚7天，
    //datetime('now','start of day',:intervalDay)，当前日期往前回滚intervalDay天
    @Query("SELECT count(*) as times ,time_date as date " +
            "FROM ( SELECT * FROM MY_APP_REPORT_DATA WHERE time_date>=datetime('now','start of day',:intervalDay) " +
            "AND time_date<=datetime('now','localtime')) WHERE time_date>=datetime(time_date,'start of day','+0 day') " +
            "AND time_date<datetime(time_date,'start of day','+1 day') group by date(time_date)")
    fun getLineDataByWeek(intervalDay:String):LiveData<List<ReportLineData>>
```

### 总结

操作类型|greenDao|Room
:-:|:-:|:-:
批量插入|==|==
批量选择|✅|
批量删除||✅
支持LiveData||✅