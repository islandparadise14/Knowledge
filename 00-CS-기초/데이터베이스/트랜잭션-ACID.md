---
tags: [CS, 데이터베이스, 트랜잭션, ACID, Room, SQLite]
---

# 트랜잭션과 ACID

## 💡 핵심 개념

트랜잭션(Transaction)은 데이터베이스에서 하나의 논리적 작업 단위를 구성하는 연산들의 집합이다. "모두 성공하거나 모두 실패"하는 원자적 실행을 보장한다. ACID는 트랜잭션이 지켜야 할 4가지 속성(원자성, 일관성, 격리성, 지속성)을 의미하며, Room에서는 `@Transaction` 어노테이션으로 이를 지원한다.

## 📌 왜 필요한가?

Android 앱에서 여러 테이블을 동시에 수정하거나, 관계 데이터를 한 번에 조회할 때 트랜잭션이 필수다. 트랜잭션 없이는 중간에 앱이 종료되면 데이터 불일치가 발생한다.

## 🔍 자세히

### ACID 속성

```
A - Atomicity (원자성)
    트랜잭션의 모든 연산이 완전히 수행되거나, 전혀 수행되지 않는다.
    → 송금 중 앱 크래시 → 전체 롤백

C - Consistency (일관성)
    트랜잭션 전후로 DB가 일관된 상태를 유지한다.
    → 잔액이 음수가 되는 송금은 거부

I - Isolation (격리성)
    동시에 실행되는 트랜잭션들이 서로 영향을 주지 않는다.
    → 두 스레드가 동시에 잔액을 읽어도 올바른 결과

D - Durability (지속성)
    트랜잭션이 커밋되면 결과가 영구적으로 저장된다.
    → 커밋 후 앱 종료 → 데이터 유지
```

### Room의 @Transaction

```kotlin
@Dao
interface TransferDao {

    // 단순한 읽기 트랜잭션: @Relation 사용 시 필수
    @Transaction
    @Query("SELECT * FROM user WHERE id = :userId")
    suspend fun getUserWithPosts(userId: Long): UserWithPosts

    // @Transaction이 필요한 이유:
    // 1. user 테이블 조회
    // 2. post 테이블 조회 (user_id 기준)
    // → 두 쿼리 사이에 데이터가 변경되면 불일치 발생
    // → @Transaction으로 묶으면 스냅샷 일관성 보장
}
```

**복합 쓰기 트랜잭션:**

```kotlin
@Dao
abstract class BankDao {

    @Query("UPDATE account SET balance = balance - :amount WHERE id = :fromId")
    abstract suspend fun withdraw(fromId: Long, amount: Long)

    @Query("UPDATE account SET balance = balance + :amount WHERE id = :toId")
    abstract suspend fun deposit(toId: Long, amount: Long)

    @Insert
    abstract suspend fun insertLog(log: TransferLog)

    // 세 연산을 하나의 트랜잭션으로 묶음
    @Transaction
    open suspend fun transfer(fromId: Long, toId: Long, amount: Long) {
        withdraw(fromId, amount)
        deposit(toId, amount)
        insertLog(TransferLog(fromId, toId, amount, System.currentTimeMillis()))
        // 하나라도 실패하면 전체 롤백
    }
}
```

### 수동 트랜잭션 관리

```kotlin
// RoomDatabase의 runInTransaction 사용
class TransferRepository(
    private val db: AppDatabase
) {
    suspend fun transfer(fromId: Long, toId: Long, amount: Long) {
        db.withTransaction {
            // 이 블록 안의 모든 DB 작업이 하나의 트랜잭션
            val from = db.accountDao().getById(fromId)
            if (from.balance < amount) {
                throw InsufficientFundsException()
                // 예외 발생 시 자동 롤백
            }
            db.accountDao().withdraw(fromId, amount)
            db.accountDao().deposit(toId, amount)
        }
        // 블록을 정상적으로 빠져나오면 자동 커밋
    }
}
```

### SQLite의 트랜잭션 모드

```kotlin
// SQLite는 3가지 트랜잭션 모드를 지원
// Room은 기본적으로 IMMEDIATE 모드 사용

// DEFERRED: 실제 DB 접근 시까지 잠금 지연 (기본값)
// IMMEDIATE: 트랜잭션 시작 시 쓰기 잠금 획득
// EXCLUSIVE: 트랜잭션 시작 시 읽기+쓰기 잠금 획득
```

### WAL 모드와 동시성

```kotlin
// Room 기본 설정: WAL (Write-Ahead Logging) 모드
// → 읽기와 쓰기를 동시에 수행 가능

@Database(entities = [User::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    companion object {
        fun create(context: Context): AppDatabase {
            return Room.databaseBuilder(
                context, AppDatabase::class.java, "app.db"
            )
            // WAL 모드 (기본 활성화)
            .setJournalMode(JournalMode.WRITE_AHEAD_LOGGING)
            .build()
        }
    }
}

// WAL 모드의 장점:
// - 읽기가 쓰기를 차단하지 않음
// - 여러 스레드에서 동시 읽기 가능
// - UI 스레드에서 Flow 수집 시 쓰기 작업에 영향 안 받음
```

### 트랜잭션과 코루틴

```kotlin
@Dao
interface BatchDao {
    @Insert
    suspend fun insertAll(items: List<Item>)

    @Transaction
    open suspend fun replaceAll(newItems: List<Item>) {
        deleteAll()       // 1. 기존 데이터 삭제
        insertAll(newItems) // 2. 새 데이터 삽입
        // suspend 함수 호출 가능
        // 코루틴 취소 시에도 트랜잭션이 롤백됨
    }

    @Query("DELETE FROM item")
    suspend fun deleteAll()
}
```

### 흔한 실수와 주의사항

```kotlin
// 실수 1: @Transaction 없이 @Relation 사용
// ❌ 두 번의 쿼리 사이에 데이터 변경 가능
@Query("SELECT * FROM user")
fun getUsersWithPosts(): Flow<List<UserWithPosts>>

// ✅ @Transaction 추가
@Transaction
@Query("SELECT * FROM user")
fun getUsersWithPosts(): Flow<List<UserWithPosts>>

// 실수 2: 메인 스레드에서 트랜잭션 실행
// ❌ ANR 발생 가능
db.runInTransaction { /* ... */ }

// ✅ 코루틴 사용
withContext(Dispatchers.IO) {
    db.withTransaction { /* ... */ }
}

// 실수 3: 트랜잭션이 너무 긴 경우
// ❌ 네트워크 호출을 트랜잭션 안에 포함
@Transaction
open suspend fun syncAndSave() {
    val data = api.fetchData()  // 느린 네트워크 호출이 잠금을 유지
    insertAll(data)
}

// ✅ 네트워크 호출은 트랜잭션 밖에서
suspend fun syncAndSave() {
    val data = api.fetchData()  // 먼저 데이터 가져오기
    db.withTransaction {
        deleteAll()
        insertAll(data)  // DB 작업만 트랜잭션으로
    }
}
```

## 🔗 관련 개념

- [[00-CS-기초/데이터베이스/SQL-기초|SQL 기초]]
- [[00-CS-기초/데이터베이스/인덱스|인덱스]]
- [[00-CS-기초/동시성/동기-vs-비동기|동기 vs 비동기]]
- [[00-CS-기초/동시성/교착상태-데드락|교착상태 데드락]]

## 📚 더 보기

- [Room 트랜잭션 문서](https://developer.android.com/reference/androidx/room/Transaction)
- [SQLite 트랜잭션](https://www.sqlite.org/lang_transaction.html)
- [WAL 모드 설명](https://www.sqlite.org/wal.html)

---

**핵심 요약:** 트랜잭션은 여러 DB 연산의 원자적 실행을 보장하며, Room에서는 `@Transaction`이나 `withTransaction`으로 ACID 속성을 지킨 안전한 데이터 처리를 구현한다.
