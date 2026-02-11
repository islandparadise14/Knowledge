---
tags: [CS, 컴파일, 런타임, Kotlin, Android]
---

# 컴파일 타임 vs 런타임

## 💡 핵심 개념

컴파일 타임(Compile Time)은 소스 코드가 실행 가능한 형태(바이트코드, 기계어)로 변환되는 시점이고, 런타임(Runtime)은 프로그램이 실제로 실행되는 시점이다. 에러가 발생하는 시점에 따라 컴파일 타임 에러와 런타임 에러로 구분하며, 가능한 한 많은 오류를 컴파일 타임에 잡는 것이 좋다.

## 📌 왜 필요한가?

Kotlin은 강력한 타입 시스템으로 많은 오류를 컴파일 타임에 감지하며, Android 빌드 시스템(Gradle + KAPT/KSP)의 동작을 이해하려면 컴파일 타임과 런타임의 차이를 명확히 알아야 한다.

## 🔍 자세히

### 비교 표

| 구분 | 컴파일 타임 | 런타임 |
|------|------------|--------|
| 시점 | 빌드 시 | 실행 시 |
| 주체 | 컴파일러 (kotlinc) | JVM / ART |
| 에러 감지 | 타입 불일치, 문법 오류 | NPE, OOM, ANR |
| 비용 | 빌드 시간 증가 | 크래시, 사용자 영향 |
| 도구 | KSP, Annotation Processor | Reflection, Dynamic Proxy |

### 컴파일 타임 에러 예시

```kotlin
// 타입 불일치 → 컴파일 에러
val number: Int = "hello" // Error: Type mismatch

// 존재하지 않는 함수 호출 → 컴파일 에러
"hello".nonExistentMethod() // Error: Unresolved reference

// Null 안전성 → 컴파일 에러
val name: String = null // Error: Null can not be a value of a non-null type

// sealed class에서 when 분기 누락 → 컴파일 경고/에러
sealed class Result {
    data class Success(val data: String) : Result()
    data class Error(val message: String) : Result()
}

fun handle(result: Result): String = when (result) {
    is Result.Success -> result.data
    // Error 분기 누락 → 컴파일 에러 (exhaustive when)
}
```

### 런타임 에러 예시

```kotlin
// NullPointerException - 플랫폼 타입에서 발생
val intent = activity.intent
val data: String = intent.getStringExtra("key")!! // 런타임 NPE 가능

// ClassCastException
val obj: Any = "hello"
val number = obj as Int // 런타임 ClassCastException

// ArrayIndexOutOfBoundsException
val list = listOf(1, 2, 3)
val item = list[5] // 런타임 IndexOutOfBoundsException

// ANR (Application Not Responding)
// 메인 스레드에서 네트워크 호출 → 런타임 ANR
```

### Android 빌드 파이프라인

```
Kotlin/Java 소스코드
    ↓ [컴파일 타임]
KSP / KAPT (어노테이션 처리)  ← Room, Hilt, Moshi 코드 생성
    ↓
kotlinc / javac → .class 바이트코드
    ↓
D8/R8 → .dex 바이트코드 (최적화 + 난독화)
    ↓
APK/AAB 패키징
    ↓ [런타임]
ART에서 .dex 실행 (AOT/JIT 컴파일)
```

### 컴파일 타임으로 에러를 옮기는 Kotlin 기법

```kotlin
// 1. sealed class/interface로 상태를 타입으로 표현
sealed interface UiState<out T> {
    object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val exception: Throwable) : UiState<Nothing>
}

// 2. value class로 타입 안전성 확보
@JvmInline
value class UserId(val value: String)

@JvmInline
value class OrderId(val value: String)

fun getOrder(userId: UserId, orderId: OrderId) { /* ... */ }

// 실수로 순서를 바꾸면 컴파일 에러
// getOrder(orderId, userId) // Error!

// 3. require/check로 런타임 보호 강화
fun setAge(age: Int) {
    require(age in 0..150) { "나이는 0~150 사이여야 합니다: $age" }
}
```

### KSP를 통한 컴파일 타임 코드 생성

```kotlin
// Room: 컴파일 타임에 SQL 검증 + DAO 구현체 생성
@Dao
interface UserDao {
    @Query("SELECT * FROM users WHERE id = :userId")
    suspend fun getUser(userId: String): User?
    // SQL 문법 오류 → 컴파일 타임에 감지됨!

    @Query("SELECT * FROM non_existent_table")
    // 컴파일 에러: 존재하지 않는 테이블
    suspend fun bad(): List<User>
}
```

## 🔗 관련 개념

- [[00-CS-기초/컴파일과-런타임/컴파일-과정|컴파일 과정]]
- [[00-CS-기초/컴파일과-런타임/타입-시스템|타입 시스템]]
- [[00-CS-기초/컴파일과-런타임/정적-vs-동적-타이핑|정적 vs 동적 타이핑]]
- [[00-CS-기초/컴파일과-런타임/바이트코드와-기계어|바이트코드와 기계어]]
- [[00-CS-기초/컴파일과-런타임/kapt-vs-ksp|KAPT vs KSP]]

## 📚 더 보기

- [Kotlin 컴파일러 공식 문서](https://kotlinlang.org/docs/command-line.html)
- [Android 빌드 개요](https://developer.android.com/build)

---

**핵심 요약:** 컴파일 타임은 코드 변환 시점, 런타임은 실행 시점이며, Kotlin의 강력한 타입 시스템과 sealed class 등을 활용하여 가능한 많은 에러를 컴파일 타임에 잡는 것이 안드로이드 앱 안정성의 핵심이다.
