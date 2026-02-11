---
tags: [CS, 타입시스템, 정적타이핑, 동적타이핑, Kotlin]
---

# 정적 vs 동적 타이핑

## 💡 핵심 개념

정적 타이핑(Static Typing)은 변수의 타입이 컴파일 타임에 결정되고 검사되는 방식이고, 동적 타이핑(Dynamic Typing)은 변수의 타입이 런타임에 결정되는 방식이다. Kotlin은 정적 타이핑 언어이지만, 타입 추론(Type Inference)을 통해 동적 타이핑 언어 수준의 간결한 코드를 작성할 수 있다.

## 📌 왜 필요한가?

안드로이드는 Kotlin/Java(정적 타이핑) 기반이며, 이 덕분에 IDE 자동완성, 리팩토링 안전성, 컴파일 타임 오류 감지가 가능하다. 반면 Gradle 스크립트(Groovy는 동적, KTS는 정적)나 서버 JSON 응답 파싱 등에서 두 패러다임의 차이를 체감하게 된다.

## 🔍 자세히

### 비교 표

| 특성 | 정적 타이핑 | 동적 타이핑 |
|------|------------|------------|
| 타입 결정 시점 | 컴파일 타임 | 런타임 |
| 타입 선언 | 필요 (추론 가능) | 불필요 |
| 대표 언어 | Kotlin, Java, Swift | Python, JavaScript, Ruby |
| 에러 감지 | 빌드 시 | 실행 시 |
| IDE 지원 | 강력 (자동완성, 리팩토링) | 제한적 |
| 성능 | 최적화 용이 | 런타임 오버헤드 |
| 유연성 | 제한적 | 높음 |

### Kotlin의 정적 타이핑 + 타입 추론

```kotlin
// 명시적 타입 선언
val name: String = "Android"
val count: Int = 42

// 타입 추론 - 컴파일러가 타입을 추론
val name = "Android"     // String으로 추론
val count = 42           // Int로 추론
val pi = 3.14            // Double로 추론
val users = listOf("A")  // List<String>으로 추론

// 타입 추론이 있어도 정적 타이핑
val name = "Android"
// name = 42  // 컴파일 에러! String 변수에 Int 대입 불가

// 반환 타입 추론
fun double(x: Int) = x * 2  // 반환 타입 Int로 추론
```

### 동적 타이핑의 위험성 (JavaScript 예시와 Kotlin 비교)

```javascript
// JavaScript (동적 타이핑) - 런타임에야 에러 발견
function greet(user) {
    return "Hello, " + user.name; // user에 name이 없으면?
}
greet(42); // 런타임에 undefined → "Hello, undefined"
```

```kotlin
// Kotlin (정적 타이핑) - 컴파일 타임에 에러 차단
data class User(val name: String)

fun greet(user: User): String {
    return "Hello, ${user.name}"
}
// greet(42) // 컴파일 에러: Type mismatch
```

### Gradle: Groovy(동적) vs KTS(정적)

```groovy
// build.gradle (Groovy - 동적 타이핑)
android {
    compileSdk 34
    defaultConfig {
        minSdk 24       // 오타: minSdkk 24 → 런타임에야 발견
        targetSdk 34
    }
}
```

```kotlin
// build.gradle.kts (Kotlin Script - 정적 타이핑)
android {
    compileSdk = 34
    defaultConfig {
        minSdk = 24       // 오타: minSdkk = 24 → 컴파일 에러!
        targetSdk = 34
    }
}
// KTS는 IDE 자동완성, 에러 하이라이팅 지원
```

### JSON 파싱에서 정적/동적의 만남

```kotlin
// 서버 JSON (타입 정보 없음 - 본질적으로 동적)
// {"name": "홍길동", "age": 25, "isPremium": true}

// 정적 타이핑으로 안전하게 매핑
@JsonClass(generateAdapter = true)
data class UserResponse(
    val name: String,
    val age: Int,
    val isPremium: Boolean
)

// Moshi가 런타임에 JSON → 정적 타입 객체로 변환
val moshi = Moshi.Builder().add(KotlinJsonAdapterFactory()).build()
val adapter = moshi.adapter(UserResponse::class.java)
val user: UserResponse = adapter.fromJson(jsonString)!!

// 서버에서 예상과 다른 타입이 오면?
// {"name": null, "age": "twenty-five"}
// → 런타임 에러 발생 → null 안전성으로 방어
```

### 타입 안전한 빌더 패턴 (DSL)

Kotlin의 정적 타이핑은 DSL에서도 타입 안전성을 보장한다.

```kotlin
// 타입 안전한 Navigation 그래프
@Serializable
data class UserProfile(val userId: String)

@Serializable
object Settings

// NavHost에서 타입 안전한 라우팅
NavHost(navController, startDestination = Settings) {
    composable<Settings> {
        SettingsScreen(
            onNavigateToProfile = { userId ->
                navController.navigate(UserProfile(userId))
            }
        )
    }
    composable<UserProfile> { backStackEntry ->
        val profile: UserProfile = backStackEntry.toRoute()
        ProfileScreen(userId = profile.userId) // 타입 안전
    }
}
```

### Duck Typing vs Structural Typing

```kotlin
// Kotlin은 명목적 타이핑(Nominal Typing) - 이름으로 타입 구분
interface Drawable {
    fun draw()
}

class Circle : Drawable {
    override fun draw() { /* ... */ }
}

class Square {
    fun draw() { /* ... */ } // Drawable을 구현하지 않음
}

// val shape: Drawable = Square() // 컴파일 에러! draw()가 있어도 안됨
// → 명시적으로 인터페이스를 구현해야 함
```

## 🔗 관련 개념

- [[00-CS-기초/컴파일과-런타임/타입-시스템|타입 시스템]]
- [[00-CS-기초/컴파일과-런타임/타입-소거|타입 소거]]
- [[00-CS-기초/컴파일과-런타임/컴파일-타임-vs-런타임|컴파일 타임 vs 런타임]]
- [[00-CS-기초/컴파일과-런타임/인터프리터-vs-컴파일러|인터프리터 vs 컴파일러]]

## 📚 더 보기

- [Kotlin Type System 공식 문서](https://kotlinlang.org/docs/basic-types.html)
- [Migrate Gradle build scripts to KTS](https://developer.android.com/build/migrate-to-kotlin-dsl)

---

**핵심 요약:** 정적 타이핑은 컴파일 타임에 타입을 검사하여 안전성을 보장하고, Kotlin은 타입 추론으로 정적 타이핑의 안전성과 동적 타이핑의 간결함을 동시에 제공한다.
