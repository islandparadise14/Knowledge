---
tags: [CS, 데이터베이스, SQL, SQLite, Room]
---

# SQL 기초

## 💡 핵심 개념

SQL(Structured Query Language)은 관계형 데이터베이스에서 데이터를 정의, 조작, 제어하기 위한 표준 언어다. Android에서는 SQLite를 내장 DB로 사용하며, Room 라이브러리를 통해 SQL을 안전하게 작성한다. DDL(데이터 정의), DML(데이터 조작), DCL(데이터 제어)로 크게 나뉜다.

## 📌 왜 필요한가?

Room의 `@Query` 어노테이션 안에 직접 SQL을 작성해야 하므로, 기본 SQL 문법을 반드시 알아야 한다. 복잡한 조인이나 서브쿼리를 작성할 때 SQL 지식이 핵심이 된다.

## 🔍 자세히

### DDL - 테이블 정의

```sql
-- Raw SQL
CREATE TABLE user (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE,
    created_at INTEGER DEFAULT 0
);
```

Room에서는 Entity로 정의한다:

```kotlin
@Entity(tableName = "user")
data class User(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "name") val name: String,
    @ColumnInfo(name = "email") val email: String?,
    @ColumnInfo(name = "created_at") val createdAt: Long = 0
)
```

### DML - 데이터 조작 (CRUD)

**INSERT**

```kotlin
@Dao
interface UserDao {
    // Room 방식
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(user: User)

    // Raw SQL 방식
    @Query("INSERT INTO user (name, email) VALUES (:name, :email)")
    suspend fun insertRaw(name: String, email: String)
}
```

**SELECT**

```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM user")
    fun getAll(): Flow<List<User>>

    @Query("SELECT * FROM user WHERE id = :userId")
    suspend fun getById(userId: Long): User?

    // LIKE 검색
    @Query("SELECT * FROM user WHERE name LIKE '%' || :keyword || '%'")
    fun search(keyword: String): Flow<List<User>>

    // 정렬과 제한
    @Query("SELECT * FROM user ORDER BY created_at DESC LIMIT :count")
    suspend fun getRecent(count: Int): List<User>
}
```

**UPDATE / DELETE**

```kotlin
@Dao
interface UserDao {
    @Update
    suspend fun update(user: User)

    @Delete
    suspend fun delete(user: User)

    @Query("UPDATE user SET name = :newName WHERE id = :userId")
    suspend fun updateName(userId: Long, newName: String)

    @Query("DELETE FROM user WHERE created_at < :timestamp")
    suspend fun deleteOldUsers(timestamp: Long)
}
```

### JOIN - 테이블 결합

```kotlin
data class UserWithPosts(
    @Embedded val user: User,
    @Relation(
        parentColumn = "id",
        entityColumn = "user_id"
    )
    val posts: List<Post>
)

@Dao
interface UserDao {
    // Room @Relation 방식
    @Transaction
    @Query("SELECT * FROM user")
    fun getUsersWithPosts(): Flow<List<UserWithPosts>>

    // Raw JOIN 방식
    @Query("""
        SELECT user.name, post.title
        FROM user
        INNER JOIN post ON user.id = post.user_id
        WHERE user.id = :userId
    """)
    suspend fun getUserPosts(userId: Long): List<UserPostTuple>
}
```

### 집계 함수

```kotlin
@Dao
interface UserDao {
    @Query("SELECT COUNT(*) FROM user")
    suspend fun getUserCount(): Int

    @Query("SELECT AVG(age) FROM user WHERE active = 1")
    suspend fun getAverageAge(): Double

    @Query("""
        SELECT department, COUNT(*) as count
        FROM user
        GROUP BY department
        HAVING count > :minCount
    """)
    suspend fun getDepartmentStats(minCount: Int): List<DepartmentCount>
}
```

### SQLite vs 표준 SQL 차이점

| 항목 | 표준 SQL | SQLite |
|------|----------|--------|
| 타입 시스템 | 엄격한 타입 | 동적 타입 (Type Affinity) |
| BOOLEAN | BOOLEAN | INTEGER (0/1) |
| 날짜/시간 | DATE, TIMESTAMP | TEXT 또는 INTEGER |
| ALTER TABLE | 모든 변경 가능 | 컬럼 추가/이름변경만 가능 |
| 동시성 | 행 수준 잠금 | 파일 수준 잠금 |

## 🔗 관련 개념

- [[00-CS-기초/데이터베이스/인덱스|인덱스]]
- [[00-CS-기초/데이터베이스/정규화|정규화]]
- [[00-CS-기초/데이터베이스/트랜잭션-ACID|트랜잭션 ACID]]
- [[00-CS-기초/데이터베이스/ORM|ORM]]

## 📚 더 보기

- [Room 공식 문서](https://developer.android.com/training/data-storage/room)
- [SQLite 공식 문서](https://www.sqlite.org/lang.html)
- [Room DAO 가이드](https://developer.android.com/training/data-storage/room/accessing-data)

---

**핵심 요약:** SQL은 관계형 DB의 표준 언어이며, Android에서는 Room의 `@Query`를 통해 컴파일 타임에 검증되는 안전한 SQL을 작성한다.
