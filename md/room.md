# Room Persistence Library

Room提供了一个SQLite之上的抽象层，使得在充分利用SQLite功能的前提下顺畅的访问数据库。


## 目录

* [概述](#概述)
* [添加依赖](#添加依赖)
* [定义Entity](#定义Entity)
* [创建DatabaseView](#创建DatabaseView)
  * [创建一个视图](#创建一个视图)
  * [将视图与数据库关联](#将视图与数据库关联)
* [使用DAO访问数据](#使用DAO访问数据)
  * [和LiveData一起使用](#和LiveData一起使用)
  * [和RxJava一起使用](#和RxJava一起使用)
  * [直接游标访问](#直接游标访问)
  * [查询多个表](#查询多个表)
* [实现RoomDatabase类](#实现RoomDatabase类)
* [迁移数据库](#迁移数据库)
  * [测试迁移](#测试迁移)
  * [导出模式](#导出模式)
  * [优雅地处理丢失的迁移路径](#优雅地处理丢失的迁移路径)
* [测试数据库](#测试数据库)
* [使用类型转换器引用复杂数据](#使用类型转换器引用复杂数据)


## 概述

SQLite是Android内置的轻量级关系型数据库，但直接使用SQLite核心包做数据库操作有以下劣势：

* 需要编写长且重复的代码，这会很耗时且容易出错。
* 管理SQL困难，特别对于复杂的数据库结构。

Room是在这样的背景下应运而生。Room充当现有SQLite API的抽象层。Room的一些特点：

* 编译时 sql 语句检查。相信大家都有过 app 跑起来，执行到 db 语句的时候 crash，检查之后发现原来是 sql 语句少了一个 ) 或者其它符号之类的经历。
  Room 会在编译阶段检查你的 DAO 中的 sql 语句，如果写错了（包括 sql 语法错误跟表名、字段名等等错误），会直接编译失败并提醒你哪里不对。
* sql 查询直接关联到 Java 对象。这个应该不用详细解释了，很多第三方 db 库早已经实现。
* 耗时操作主动要求异步处理。这一点还是挺值得注意的，Room 会在执行 db 操作时判断是不是在 UI 线程。
  比如当你需要插入一条记录到数据库时，Room 会让你放到异步线程去做，否则会直接 crash 掉 app 来告诉你不这样做容易阻塞 UI 线程。
* 基于注解编译时自动生成代码。这个应该算是 Room 工作原理的核心所在了。
* API 设计符合 sql 标准。方便扩展进行各种 db 操作。

Room有3个主要组成部分:

* Entity:表示数据库中的表。
* DAO:包含用于访问数据库的方法。
* Database:包含数据库持有者，并作为应用程序持久关系数据的底层连接的主要访问点。

Room的不同组件之间的关系如图所示：

![room_architecture](images/room_architecture.png)


## 添加依赖

在项目build.gradle文件中添加以下依赖

```
dependencies {
    def room_version = "1.1.1"

    implementation "android.arch.persistence.room:runtime:$room_version"
    // For Kotlin use kapt instead of annotationProcessor
    annotationProcessor "android.arch.persistence.room:compiler:$room_version"

    // 可选 - RxJava support for Room
    implementation "android.arch.persistence.room:rxjava2:$room_version"

    // 可选 - Guava support for Room, including Optional and ListenableFuture
    implementation "android.arch.persistence.room:guava:$room_version"

    // 测试
    testImplementation "android.arch.persistence.room:testing:$room_version"
}
```

AndroidX

```
dependencies {
    def room_version = "2.1.0-alpha07"

    implementation "androidx.room:room-runtime:$room_version"
    // For Kotlin use kapt instead of annotationProcessor
    annotationProcessor "androidx.room:room-compiler:$room_version"

    // 可选 - Kotlin Extensions and Coroutines support for Room
    implementation "androidx.room:room-ktx:$room_version"

    // 可选 - RxJava support for Room
    implementation "androidx.room:room-rxjava2:$room_version"

    // 可选 - Guava support for Room, including Optional and ListenableFuture
    implementation "androidx.room:room-guava:$room_version"

    // 测试
    testImplementation "androidx.room:room-testing:$room_version"
}
```


## 定义Entity

使用Room持久性库时，可以将相关字段集定义为实体。对于每个实体，在关联Database对象内创建一个表来保存项目。您必须通过Database类中的entities数组引用实体类。

下面的代码片段展示了如何定义一个实体:

```
import android.arch.persistence.room.ColumnInfo;
import android.arch.persistence.room.Embedded;
import android.arch.persistence.room.Entity;
import android.arch.persistence.room.ForeignKey;
import android.arch.persistence.room.Ignore;
import android.arch.persistence.room.Index;
import android.arch.persistence.room.PrimaryKey;
import android.graphics.Bitmap;

class Address {
    public String street;
    public String state;
    public String city;

    @ColumnInfo(name = "post_code")
    public int postCode;
}

@Entity(
        tableName = "user",
        primaryKeys = {"firstName", "lastName"},
        indices = {@Index("name"), @Index(value = {"first_name", "last_name"}, unique = true)}
)
public class User {
    @PrimaryKey
    public int uid;

    @ColumnInfo(name = "first_name")
    public String firstName;

    @ColumnInfo(name = "last_name")
    public String lastName;

    @Embedded
    public Address address;

    @Ignore
    Bitmap picture;
}

@Entity(foreignKeys = @ForeignKey(entity = User.class, parentColumns = "uid", childColumns = "user_id"))
class Book {

    @PrimaryKey
    public int bookId;

    public String title;

    @ColumnInfo(name = "user_id")
    public int userId;
}
```

要持久化字段，Room必须能够访问它。您可以将字段公开，或者为其提供getter和setter。如果您使用getter和setter方法，请记住它们基于Room中的JavaBeans约定。

重点说明:

* @Entity:顶部使用@Entity声明这是一个实体类。你必须在Database类中的entities数组中引用这些entity类。entity中的每一个field都将被持久化到数据库，除非使用了@Ignore注解。
  * tableName:表名。tableName如果不写，那么默认类名就是表名。(注意:SQLite中的表名不区分大小写。)
  * primaryKeys:组合主键。
  * indices:定义索引，列出希望包含在索引或复合索引中的列的名称。有时，数据库中的某些字段或字段组必须是惟一的，您可以通过将@Index注释的unique属性设置为true来强制执行此唯一性属性。

    Index索引注解可选参数：

    ```
    public @interface Index {
        // 定义需要添加索引的字段
        String[] value();
        // 定义索引的名称
        String name() default "";
        // true-设置唯一键，标识value数组中的索引字段必须是唯一的，不可重复
        boolean unique() default false;
    }
    ```

  * foreignKeys:定义外键。

    ForeignKey外键注解可选参数：

    ```
    public @interface ForeignKey {
        // 引用外键的表的实体
        Class entity();
        // 要引用的外键列
        String[] parentColumns();
        // 要关联的列
        String[] childColumns();
        // 当父类实体(关联的外键表)从数据库中删除时执行的操作
        @Action int onDelete() default NO_ACTION;
        // 当父类实体(关联的外键表)更新时执行的操作
        @Action int onUpdate() default NO_ACTION;
        // 在事务完成之前，是否应该推迟外键约束
        boolean deferred() default false;
        // 给onDelete，onUpdate定义的操作
        int NO_ACTION = 1;
        int RESTRICT = 2;
        int SET_NULL = 3;
        int SET_DEFAULT = 4;
        int CASCADE = 5;
        @IntDef({NO_ACTION, RESTRICT, SET_NULL, SET_DEFAULT, CASCADE})
        @interface Action {
        }
    }
    ```

* @PrimaryKey:标识属性为表的主键。
  * 每个实体必须至少定义一个字段作为主键。即使只有一个字段，仍然需要使用@PrimaryKey注释该字段。
  * 如果您希望为实体分配自动id，可以设置@PrimaryKey的autoGenerate属性，可以理解成就是AUTOINCRESEMENT。
* @ColumnInfo:与tableName属性类似，使用字段名作为数据库中的列名。如果希望列具有不同的名称，请将@ColumnInfo注释添加到字段中。
* @Embedded:有时，您希望将一个实体或普通的以前的Java对象(POJO)作为数据库逻辑中的一个完整的整体来表示，即使该对象包含几个字段。
  在此情况下，您可以使用@Embedded来表示一个对象，您希望将其分解为表中的子字段。然后可以像对其他单个列一样查询嵌入式字段。
  * 嵌入式字段还可以包含其他嵌入式字段。
  * 如果一个实体具有多个相同类型的嵌入式字段，则可以通过设置prefix属性使每个列保持惟一(@Embedded(prefix = "a_"))。然后，Room将提供的值添加到嵌入对象中每个列名的开头。
* @Ignore:需要忽略的字段或方法。默认情况下，Room为实体中定义的每个字段创建一个列。如果一个实体有您不想持久化的字段，您可以使用@Ignore注释它们。


## 创建DatabaseView

Room持久库的2.1.0及更高版本提供了对SQLite数据库视图的支持，允许将查询封装到类中。Room将这些查询支持的类引用为视图，当在DAO中使用时，它们的行为与简单的数据对象相同。

> **注意**:与实体一样，您可以对视图运行SELECT语句。但是，不能对视图运行INSERT、UPDATE或DELETE语句。

### 创建一个视图

要创建视图，请将@DatabaseView注释添加到类中。将注释的值设置为类应该表示的查询。

```
@DatabaseView("SELECT user.id, user.name, user.departmentId," +
              "department.name AS departmentName FROM user " +
              "INNER JOIN department ON user.departmentId = department.id")
public class UserDetail {
    public long id;
    public String name;
    public long departmentId;
    public String departmentName;
}
```

### 将视图与数据库关联

要将视图包含在应用程序的数据库中，请在应用程序的@Database注释中包含views属性:

```
@Database(entities = {User.class}, views = {UserDetail.class}, version = 1)
public abstract class AppDatabase extends RoomDatabase {
    public abstract UserDao userDao();
}
```


## 使用DAO访问数据

要使用Room持久性库访问应用程序的数据，需要使用数据访问对象(Data Access Objects，简称DAOs)。这组Dao对象构成了Room的主要组件，因为每个DAO都包含提供对应用程序数据库的抽象访问的方法。

通过使用DAO类而不是查询构造器或直接查询访问数据库，可以分离数据库体系结构的不同组件。此外，DAOs允许您在测试应用程序时轻松模拟数据库访问。

```
class NameTuple {
    @ColumnInfo(name = "first_name")
    public String firstName;

    @ColumnInfo(name = "last_name")
    public String lastName;
}

@Dao
interface UserDao {

    // 如果@Insert方法只接收一个参数，它可以返回一个long，这是插入项的新rowId。如果参数是数组或集合，则应该返回long[]或List<long>。
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    void insertUsers(User... users);

    @Insert
    void insertBothUsers(User user1, User user2);

    @Insert
    void insertUsersAndFriends(User user, List<User> friends);

    // 虽然通常没有必要，但是您可以让这个方法返回一个int值，以指示数据库中更新的行数。
    @Update
    void updateUsers(User... users);

    // 虽然通常没有必要，但是您可以让这个方法返回一个int值，表示从数据库中删除的行数。
    @Delete
    void deleteUsers(User... users);

    // @Query是DAO类中使用的主要注释。它允许您对数据库执行读/写操作。每个@Query方法都在编译时进行验证，因此如果查询有问题，就会发生编译错误，而不是运行时失败。
    @Query("SELECT * FROM user")
    User[] loadAllUsers();

    @Query("SELECT * FROM user WHERE age > :minAge")
    User[] loadAllUsersOlderThan(int minAge);

    @Query("SELECT * FROM user WHERE age BETWEEN :minAge AND :maxAge")
    User[] loadAllUsersBetweenAges(int minAge, int maxAge);

    @Query("SELECT * FROM user WHERE first_name LIKE :search " + "OR last_name LIKE :search")
    List<User> findUserWithName(String search);

    @Query("SELECT first_name, last_name FROM user")
    List<NameTuple> loadFullName();

    @Query("SELECT first_name, last_name FROM user WHERE region IN (:regions)")
    List<NameTuple> loadUsersFromRegions(List<String> regions);
}
```

重点说明:

* @Dao：DAO可以是接口，也可以是抽象类。如果它是一个抽象类，它可以选择有一个构造函数，该构造函数将RoomDatabase作为它唯一的参数。
* @Insert：插入到表的数据。
* @Update：更新到表数据。
* @Delete：删除表的数据。
* @Query：执行SQL查询。

这个接口不需要写实现类，Room在编译时创建每个DAO实现。

@insert，@Update都可以执行事务操作，定义在OnConflictStrategy注解类中。

```
public @interface Insert {
    // 定义处理冲突的操作
    @OnConflictStrategy
    int onConflict() default OnConflictStrategy.ABORT;
}
```

```
public @interface OnConflictStrategy {
    // 策略冲突就替换旧数据
    int REPLACE = 1;
    // 策略冲突就回滚事务
    int ROLLBACK = 2;
    // 策略冲突就退出事务
    int ABORT = 3;
    // 策略冲突就使事务失败
    int FAIL = 4;
    // 忽略冲突
    int IGNORE = 5;
}
```

> 注意:除非在构建器上调用allowMainThreadQueries()，否则Room不支持在主线程上访问数据库，因为它可能会长期锁定UI。
异步查询返回LiveData或Flowable实例的查询不受此规则约束，因为它们在需要时在后台线程上异步运行查询。

### 和LiveData一起使用

要使用此功能，请在应用程序的构建中包含最新版本的livedata构件。

```
implementation "android.arch.lifecycle:livedata:1.1.1"
```

下面的代码片段展示了几个示例，说明如何使用这些返回类型:

```
@Dao
interface UserDao {
    @Query("SELECT * FROM user")
    public LiveData<List<User>> getAll();

    @Query("SELECT first_name, last_name FROM user WHERE region IN (:regions)")
    LiveData<List<NameTuple>> loadUsersFromRegionsSync(List<String> regions);
}
```

在执行查询时，您通常希望应用程序的UI在数据更改时自动更新。为此，在查询方法描述中使用LiveData类型的返回值。Room生成更新数据库时更新LiveData所需的所有代码。

> 注意:从1.0版开始，Room使用查询中访问的表列表来决定是否更新LiveData的实例。

### 和RxJava一起使用

Room为RxJava2类型的返回值提供了以下支持:

* @Query方法: Room支持类型为Publisher、Flowable和Observable的返回值。
* @Insert、@Update和@Delete方法:Room 2.1.0及更高版本返回类型的值Completable，Single<T>和Maybe<T>。

要使用此功能，请在应用程序的构建中包含最新版本的rxjava2构件。

```
implementation 'android.arch.persistence.room:rxjava2:1.1.1'
```

下面的代码片段展示了几个示例，说明如何使用这些返回类型:

```
@Dao
interface UserDao {
    @Query("SELECT * FROM user")
    public Flowable<List<User>> getAll();

    @Query("SELECT * from user where uid = :id LIMIT 1")
    public Flowable<User> loadUserById(int id);

    // Emits the number of users added to the database.
    @Insert
    public Maybe<Integer> insertLargeNumberOfUsers(List<User> users);

    // Makes sure that the operation finishes successfully.
    @Insert
    public Completable insertLargeNumberOfUsers(User... users);

    // Emits the number of users removed from the database. Always emits at least one user.
    @Delete
    public Single<Integer> deleteUsers(List<User> users);
}
```

### 直接游标访问

如果应用程序的逻辑要求直接访问返回行，则可以从查询中返回Cursor对象，如下面的代码片段所示:

```
@Dao
interface UserDao {
    @Query("SELECT * FROM user")
    public Cursor getAll();
}
```

### 查询多个表

有些查询可能需要访问多个表来计算结果。Room允许您编写任何查询，因此您还可以连接表。此外，如果响应是可观察的数据类型，例如Flowable或LiveData，则Room将监视查询中引用的所有表，以确定是否无效。

下面的代码片段展示了如何执行表连接，以合并包含借书用户的表和包含当前借书数据的表之间的信息:

```
@Dao
interface MyDao {
    @Query("SELECT * FROM book " +
            "INNER JOIN loan ON loan.book_id = book.id " +
            "INNER JOIN user ON user.id = loan.user_id " +
            "WHERE user.name LIKE :userName")
    List<Book> findBooksBorrowedByNameSync(String userName);
}
```

您还可以从这些查询中返回POJO。例如，您可以编写一个加载用户及其宠物名称的查询，如下所示：

```
@Dao
interface MyDao {
    @Query("SELECT user.name AS userName, pet.name AS petName " +
            "FROM user, pet " +
            "WHERE user.id = pet.user_id")
    public LiveData<List<UserPet>> loadUserAndPetNames();

    // You can also define this class in a separate file, as long as you add the "public" access modifier.
    static class UserPet {
        public String userName;
        public String petName;
    }
}
```

## 实现RoomDatabase类

@Database：使用此注释的类提供了一系列DAOs，它也是下层的主要连接的访问点。注解的类应该是一个抽象的继承RoomDatabase的类。
在运行时，你能获得一个实例通过调用Room.databaseBuilder()或者Room.inMemoryDatabaseBuilder()。

```
@Database(entities = {User.class}, version = 1)
public abstract class AppDatabase extends RoomDatabase {
    public abstract UserDao userDao();
}
```

在创建上述文件之后，您将使用以下代码获得所创建数据库的一个实例:

```
AppDatabase db = Room.databaseBuilder(getApplicationContext(), AppDatabase.class, "database-name").build();
```

> 注：在实例化AppDatabase对象的时候你应该遵循单例模式，因为每个Database实例都是相当昂贵的，而且你也很少需要多个实例。

```
@Database(entities = {User.class}, version = 1)
public abstract class AppDatabase extends RoomDatabase {

    private static volatile AppDatabase INSTANCE;

    private static final Object lock = new Object();

    static TaskDataBase getInstance(Context context) {
        synchronized (lock) {
            if (INSTANCE == null) {
                INSTANCE = Room.databaseBuilder(context.getApplicationContext(), AppDatabase.class, "database-name").build();
            }
        }
        return INSTANCE;
    }

    abstract TasksDAO tasksDAO();
}
```


## 迁移数据库

在应用程序中添加和更改特性时，需要修改实体类来反映这些更改。当用户更新到您的应用程序的最新版本时，您不希望他们丢失所有现有的数据，尤其是在您无法从远程服务器恢复数据的情况下。

Room持久性库允许您以这种方式编写迁移类来保存用户数据。每个迁移类都指定一个startVersion和endVersion。在运行时，Room运行每个迁移类的migrate()方法，使用正确的顺序将数据库迁移到较晚的版本。

```
private static final Migration MIGRATION_1_2 = new Migration(1, 2) {
    @Override
    public void migrate(SupportSQLiteDatabase database) {
        database.execSQL("CREATE TABLE book (id INTEGER PRIMARY KEY, name TEXT, owner INTEGER)");
    }
};

private static final Migration MIGRATION_2_3 = new Migration(2, 3) {
    @Override
    public void migrate(SupportSQLiteDatabase database) {
        database.execSQL("ALTER TABLE Book ADD COLUMN pub_year INTEGER");
    }
};
```

```
Room.databaseBuilder(context.getApplicationContext(), AppDatabase.class, "database-name").addMigrations(MIGRATION_1_2, MIGRATION_2_3).build();
```

### 测试迁移

编写迁移并不简单，如果不能正确地编写迁移，可能会在应用程序中导致崩溃循环。为了保持应用程序的稳定性，应该事先测试迁移。Room提供了一个测试Maven构件来帮助完成这个测试过程。然而，要使此构件工作，您需要导出数据库的模式。

### 导出模式

编译后，Room会将数据库的架构信息导出到JSON文件中。要导出架构，请在build.gradle文件中配置room.schemaLocation的导出路径，如以下代码段所示：

```
android {
    ...
    defaultConfig {
        ...
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = ["room.schemaLocation": "$projectDir/schemas".toString()]
            }
        }
    }
}
```

如果不需要导出Schema，可以在@Database中配置：

```
@Database(entities = {User.class}, version = 1, exportSchema = false)
```

您应该在版本控制系统中存储导出的JSON文件（表示数据库的模式历史记录），因为它允许Room创建数据库的旧版本以进行测试。

要测试这些迁移，添加android.arch.persistence.room:testing到你的测试依赖当中，并且把schema位置当做一个asset文件添加，如下所示：

```
android {
    ...
    sourceSets {
        androidTest.assets.srcDirs += files("$projectDir/schemas".toString())
    }
}
```

测试包提供了一个MigrationTestHelper类，该类可以读取这些模式文件。它还实现了JUnit4 TestRule接口，因此可以管理创建的数据库。

下面的代码片段显示了一个示例迁移测试:

```
@RunWith(AndroidJUnit4.class)
public class MigrationTest {
    private static final String TEST_DB = "migration-test";

    @Rule
    public MigrationTestHelper helper;

    public MigrationTest() {
        helper = new MigrationTestHelper(InstrumentationRegistry.getInstrumentation(),
                MigrationDb.class.getCanonicalName(), new FrameworkSQLiteOpenHelperFactory());
    }

    @Test
    public void migrate1To2() throws IOException {
        SupportSQLiteDatabase db = helper.createDatabase(TEST_DB, 1);

        // db has schema version 1. insert some data using SQL queries.
        // You cannot use DAO classes because they expect the latest schema.
        db.execSQL(...);

        // Prepare for the next version.
        db.close();

        // Re-open the database with version 2 and provide
        // MIGRATION_1_2 as the migration process.
        db = helper.runMigrationsAndValidate(TEST_DB, 2, true, MIGRATION_1_2);

        // MigrationTestHelper automatically verifies the schema changes,
        // but you need to validate that the data was migrated properly.
    }
}
```

### 优雅地处理丢失的迁移路径

更新了数据库的模式之后，一些设备上的数据库可能仍然使用旧的模式版本。如果Room无法找到将该设备的数据库从旧版本升级到当前版本的迁移规则，则会发生IllegalStateException。

为了防止应用程序在这种情况下崩溃，在创建数据库时调用fallbackToDestructiveMigration()构建方法:

```
Room.databaseBuilder(getApplicationContext(), AppDatabase.class, "database-name").fallbackToDestructiveMigration().build();
```

通过在应用程序的数据库构建逻辑中包含这个子句，您可以告诉Room在模式版本之间缺少迁移路径的情况下破坏性地重新创建应用程序的数据库表。

> **警告**:通过在应用程序的数据库生成器中配置此选项，当缺少迁移路径时，Room将永久删除数据库表中的所有数据。

破坏性的重新创建回退逻辑包括几个额外的选项:

* 如果错误发生在您的模式历史的特定版本中，而您无法使用迁移路径解决这些错误，那么使用fallbackToDestructiveMigrationFrom()。
  此方法表明，您希望仅在数据库试图从这些有问题的版本之一迁移时才有使用回退逻辑的空间。
* 要仅在尝试降级模式时执行破坏性的重新创建，请使用fallbackToDestructiveMigrationOnDowngrade()降级()。


## 测试数据库

在使用Room持久库创建数据库时，验证应用程序数据库和用户数据的稳定性非常重要。

当运行你app的测试时，你不应该创建一个完全的数据库如果你不测试数据库本身。Room允许你轻松的模仿数据访问层在测试当中。
这个过程是可能的因为你的DAOs不暴漏任何你数据库的细节。当测试你的应用，你应该创建模仿你的DAO类的假的实例。

测试你的数据库推荐的方法实现是写一个单元测试在Android设备上。因为这些测试不需要创建一个activity，所以它们的执行速度应该比UI测试快。

在设置测试时，你应该创建一个内存版本的数据库，使你的测试更加封闭，如下面的例子所示:

```
@RunWith(AndroidJUnit4.class)
public class SimpleEntityReadWriteTest {
    private UserDao userDao;
    private TestDatabase db;

    @Before
    public void createDb() {
        Context context = ApplicationProvider.getApplicationContext();
        db = Room.inMemoryDatabaseBuilder(context, TestDatabase.class).build();
        userDao = db.getUserDao();
    }

    @After
    public void closeDb() throws IOException {
        db.close();
    }

    @Test
    public void writeUserAndReadInList() throws Exception {
        User user = TestUtil.createUser(3);
        user.setName("george");
        userDao.insert(user);
        List<User> byName = userDao.findUsersByName("george");
        assertThat(byName.get(0), equalTo(user));
    }
}
```


## 使用类型转换器引用复杂数据

有时，应用程序需要使用自定义数据类型，希望将其值存储在单个数据库列中。要添加对自定义类型的这种支持，您需要提供TypeConverter，它将自定义类转换为Room可以持久存储的已知类型，并将其转换为其他类型。

例如，如果我们想要保存Date的实例，我们可以编写下面的TypeConverter来在数据库中存储等价的Unix时间戳:

```
public class Converters {
    @TypeConverter
    public static Date fromTimestamp(Long value) {
        return value == null ? null : new Date(value);
    }

    @TypeConverter
    public static Long dateToTimestamp(Date date) {
        return date == null ? null : date.getTime();
    }
}
```

上面的示例定义了两个函数，一个函数将Date对象转换为Long对象，另一个函数执行Long到Date的逆转换。由于Room已经知道如何持久化长对象，所以它可以使用这个转换器来持久化Date类型的值。

接下来，将@TypeConverters注释添加到AppDatabase类中，这样Room就可以使用为AppDatabase中的每个实体和DAO定义的转换器:

```
@Database(entities = {User.class}, version = 1)
@TypeConverters({Converters.class})
public abstract class AppDatabase extends RoomDatabase {
    ...
}
```

使用这些转换器，您可以在其他查询中使用自定义类型，就像使用基本类型一样，如下面的代码片段所示:

```
@Entity
public class User {
    private Date birthday;
}
```

```
@Dao
public interface UserDao {
    @Query("SELECT * FROM user WHERE birthday BETWEEN :from AND :to")
    List<User> findUsersBornBetweenDates(Date from, Date to);
}
```

您还可以将@TypeConverters限制在不同的范围内，包括单个实体、DAO和DAO方法。有关详细信息，请参阅[@TypeConverters](https://developer.android.google.cn/reference/androidx/room/TypeConverters.html)注释的参考文档。

> Room提供了在基本类型和装箱类型之间转换的功能，但不允许在实体之间引用对象。