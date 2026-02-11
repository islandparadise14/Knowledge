---
tags: [CS, 데이터베이스, ORM, Room, SQLDelight, Exposed]
---

# ORM

## 💡 핵심 개념

ORM(Object-Relational Mapping)은 객체 지향 프로그래밍의 객체와 관계형 데이터베이스의 테이블을 자동으로 매핑하는 기술이다. SQL을 직접 작성하지 않고 객체를 다루듯 DB를 조작할 수 있게 해준다. Android에서는 Room이 사실상 표준 ORM이며, SQLDelight와 Realm도 대안으로 사용된다.

## 📌 왜 필요한가?

Raw SQLite API는 커서 관리, 타입 변환, 스레드 안전성 등을 직접 처리해야 해서 번거롭고 에러가 발생하기 쉽다. Room 같은 ORM을 사용하면 컴파일 타임 검증, 코루틴 지원, 자동 매핑으로 안전하고 생산적인 DB 개발이 가능하다.

## 🔍 자세히

### ORM이 해결하는 문제

```kotlin
// Raw SQLite API (ORM 없이)
fun getUsers(db: SQLiteDatabase): List<User> {
    val users = mutableListOf<User>()
    val cursor = db.rawQuery("SELECT * FROM user", null)
    try {
        while (cursor.moveToNext()) {
            users.add(User(
                id = cursor.getLong(cursor.getColumnIndexOrThrow("id")),
                name = cursor.getString(cursor.getColumnIndexOrThrow("name")),
                email = cursor.getString(cursor.getColumnIndexOrThrow("email"))
                // 컬럼 이름 오타? → 런타임 크래시
                // 타입 불일치? → 런타임 크래시
            ))
        }
    } finally {
        cursor.close()  // 닫지 않으면 메모리 릭
    }
    return users
}

// Room ORM 사용
@Query("SELECT * FROM user")
suspend fun getUsers(): List<User>
// → 컴파일 타임 검증, 자동 매핑, 커서 관리 불필요
```

### Room - Android 표준 ORM

**3가지 핵심 컴포넌트:**

```kotlin
// 1. Entity - 테이블 정의
@Entity(
    tableName = "user",
    indices = [Index(value = ["email"], unique = true)]
)
data class User(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val name: String,
    val email: String,
    @ColumnInfo(name = "created_at") val createdAt: Long = System.currentTimeMillis()
)

// 2. DAO - 데이터 접근 객체
@Dao
interface UserDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(user: User)

    @Query("SELECT * FROM user WHERE id = :id")
    suspend fun getById(id: Long): User?

    @Query("SELECT * FROM user")
    fun observeAll(): Flow<List<User>>

    @Delete
    suspend fun delete(user: User)
}

// 3. Database - DB 인스턴스
@Database(entities = [User::class], version = 1)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```

**TypeConverter - 커스텀 타입 매핑:**

```kotlin
class Converters {
    @TypeConverter
    fun fromTimestamp(value: Long?): Date? = value?.let { Date(it) }

    @TypeConverter
    fun dateToTimestamp(date: Date?): Long? = date?.time

    @TypeConverter
    fun fromStringList(value: List<String>): String = Json.encodeToString(value)

    @TypeConverter
    fun toStringList(value: String): List<String> = Json.decodeFromString(value)
}
```

**마이그레이션:**

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE user ADD COLUMN age INTEGER NOT NULL DEFAULT 0")
    }
}

Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .addMigrations(MIGRATION_1_2)
    // .fallbackToDestructiveMigration() // 개발 중에만
    .build()
```

### SQLDelight - SQL 우선 접근

```kotlin
// SQLDelight: .sq 파일에 SQL을 먼저 작성 → Kotlin 코드 자동 생성

// user.sq 파일
/*
CREATE TABLE user (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE
);

selectAll:
SELECT * FROM user;

selectById:
SELECT * FROM user WHERE id = ?;

insert:
INSERT INTO user (name, email) VALUES (?, ?);
*/

// 자동 생성된 코드 사용
class UserRepository(private val db: Database) {
    fun getAll(): Flow<List<User>> =
        db.userQueries.selectAll()
            .asFlow()
            .mapToList(Dispatchers.IO)

    suspend fun insert(name: String, email: String) {
        db.userQueries.insert(name, email)
    }
}

// SQLDelight 장점:
// - KMP(Kotlin Multiplatform) 지원
// - SQL 먼저 → 더 정밀한 쿼리 제어
// - 마이그레이션 자동 검증
```

### Room vs SQLDelight vs Realm

| 항목 | Room | SQLDelight | Realm |
|------|------|------------|-------|
| 접근 방식 | ORM (객체 우선) | SQL 우선 | 객체 DB |
| 플랫폼 | Android 전용 | KMP 지원 | KMP 지원 |
| 쿼리 검증 | 컴파일 타임 | 컴파일 타임 | 런타임 |
| 학습 곡선 | 낮음 | 중간 | 중간 |
| 코루틴 | 네이티브 지원 | 지원 | 지원 |
| Flow | 네이티브 지원 | 지원 | 지원 |
| Google 지원 | 공식 권장 | 커뮤니티 | MongoDB |
| 기반 DB | SQLite | SQLite/기타 | 자체 엔진 |

### ORM의 N+1 문제

```kotlin
// N+1 문제: 1번의 목록 쿼리 + N번의 관계 쿼리
// Room의 @Relation은 내부적으로 2번의 쿼리로 해결 (1+1)

// 비효율적: 각 유저마다 별도 쿼리
suspend fun getUsersWithPosts(): List<UserWithPosts> {
    val users = userDao.getAll()       // 1번 쿼리
    return users.map { user ->
        val posts = postDao.getByUserId(user.id)  // N번 쿼리!
        UserWithPosts(user, posts)
    }
}

// Room @Relation 사용 (효율적: 2번의 쿼리)
data class UserWithPosts(
    @Embedded val user: User,
    @Relation(parentColumn = "id", entityColumn = "userId")
    val posts: List<Post>
)

@Transaction
@Query("SELECT * FROM user")
suspend fun getUsersWithPosts(): List<UserWithPosts>
// 내부: SELECT * FROM user + SELECT * FROM post WHERE userId IN (1,2,3...)
```

### ORM 사용 시 주의사항

```kotlin
// 1. 메인 스레드 접근 금지
// ❌ allowMainThreadQueries()는 프로덕션에서 사용 금지
Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .allowMainThreadQueries()  // 개발/테스트에서만!
    .build()

// ✅ suspend 함수 또는 Flow 사용
@Query("SELECT * FROM user")
suspend fun getAll(): List<User>     // IO 스레드에서 실행
@Query("SELECT * FROM user")
fun observeAll(): Flow<List<User>>   // 자동 백그라운드 실행

// 2. 대량 데이터 삽입 시 트랜잭션 사용
@Transaction
suspend fun insertAll(users: List<User>) {
    users.forEach { insert(it) }
    // 트랜잭션 없이 1000건 삽입: ~3초
    // 트랜잭션으로 1000건 삽입: ~0.1초
}

// 3. Entity와 도메인 모델 분리
// Entity는 DB 구조, 도메인 모델은 비즈니스 로직
fun User.toDomain(): DomainUser = DomainUser(id, name, email)
fun DomainUser.toEntity(): User = User(id, name, email)
```

## 🔗 관련 개념

- [[00-CS-기초/데이터베이스/SQL-기초|SQL 기초]]
- [[00-CS-기초/데이터베이스/트랜잭션-ACID|트랜잭션 ACID]]
- [[00-CS-기초/데이터베이스/인덱스|인덱스]]
- [[00-CS-기초/소프트웨어-공학/의존성-주입-개념|의존성 주입]]
- [[00-CS-기초/소프트웨어-공학/추상화와-캡슐화|추상화와 캡슐화]]

## 📚 더 보기

- [Room 공식 문서](https://developer.android.com/training/data-storage/room)
- [SQLDelight 공식 사이트](https://cashapp.github.io/sqldelight/)
- [Realm Kotlin SDK](https://www.mongodb.com/docs/realm/sdk/kotlin/)
- [Room with Coroutines & Flow](https://developer.android.com/training/data-storage/room/async-queries)

---

**핵심 요약:** ORM은 객체와 DB 테이블 간의 매핑을 자동화하며, Android에서는 Room이 컴파일 타임 검증, 코루틴/Flow 지원으로 표준 선택이고, KMP가 필요하면 SQLDelight를 고려한다.
