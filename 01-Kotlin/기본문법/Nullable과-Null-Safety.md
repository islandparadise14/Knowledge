---
tags: [kotlin, basics, null-safety]
---

# Nullable과 Null Safety

## 💡 핵심 개념

Kotlin은 **Null Safety**를 언어 차원에서 지원한다. `?`를 붙여 nullable 타입을 명시하고, 컴파일 타임에 NPE를 방지한다.

## 📌 왜 필요한가?

Java의 최대 문제 중 하나가 NullPointerException(NPE)이다. Kotlin은 타입 시스템으로 이를 해결해서 런타임 크래시를 크게 줄인다.

## 🔍 자세히

### 기본 사용법

```kotlin
// Non-null 타입 (기본)
val name: String = "민트"
// name = null  // ERROR: Null can not be a value of a non-null type

// Nullable 타입
val nullableName: String? = null  // OK
val age: Int? = null  // OK
```

### Safe Call (?.)

```kotlin
val name: String? = null

// Safe call - null이면 null 반환
val length: Int? = name?.length
val upper: String? = name?.uppercase()

// 체이닝 가능
val firstChar: Char? = name?.uppercase()?.first()
```

### Elvis 연산자 (?:)

```kotlin
val name: String? = null

// null이면 기본값 사용
val length = name?.length ?: 0
val displayName = name ?: "Unknown"

// 함수와 함께
fun getName(): String? = null
val result = getName() ?: throw IllegalStateException("Name required")
```

### Non-null Assertion (!!)

```kotlin
val name: String? = "민트"

// null이 아니라고 확신할 때 (위험!)
val length: Int = name!!.length

// null이면 NPE 발생
val nullName: String? = null
val length2 = nullName!!.length  // KotlinNullPointerException
```

**사용 금지 상황:**
```kotlin
// 이렇게 쓰지 마!
user!!.profile!!.name!!.length

// 대신 이렇게
user?.profile?.name?.length ?: 0
```

### Safe Cast (as?)

```kotlin
val obj: Any = "Hello"

// Safe cast - 실패하면 null 반환
val str: String? = obj as? String  // OK
val num: Int? = obj as? Int  // null
```

### let 함수와 활용

```kotlin
val name: String? = "민트"

// null이 아닐 때만 실행
name?.let {
    println("Name length: ${it.length}")
}

// null일 때는 실행 안 됨
val nullName: String? = null
nullName?.let {
    println("This won't be printed")
}
```

### 동작 원리

**플랫폼 타입**

```kotlin
// Java 코드
public String getName() {
    return null;  // @Nullable이 없음
}

// Kotlin에서
val name = javaClass.getName()  // String! (플랫폼 타입)
// null일 수도 있으니 주의!
```

**@Nullable, @NonNull 어노테이션**

```kotlin
// Java
public @Nullable String getName() { ... }
public @NonNull String getTitle() { ... }

// Kotlin에서
val name: String? = javaClass.getName()  // Nullable로 인식
val title: String = javaClass.getTitle()  // Non-null로 인식
```

### 주의사항

**1. lateinit 체크**

```kotlin
lateinit var viewModel: MyViewModel

// 초기화 전 접근하면 UninitializedPropertyAccessException
// viewModel.doSomething()  // 크래시!

// 초기화 여부 확인
if (::viewModel.isInitialized) {
    viewModel.doSomething()
}
```

**2. 컬렉션의 nullable**

```kotlin
// nullable 타입의 리스트
val list: List<String?> = listOf("A", null, "B")

// nullable한 리스트 (리스트 자체가 null 가능)
val nullableList: List<String>? = null

// 둘 다 nullable
val both: List<String?>? = null

// null 제거
val filtered = list.filterNotNull()  // List<String>
```

**3. 스마트 캐스트**

```kotlin
fun process(name: String?) {
    if (name != null) {
        // 이 블록 안에서는 String으로 자동 캐스트
        println(name.length)
    }
}

// when과 함께
fun process2(value: Any?) {
    when (value) {
        is String -> println(value.length)  // String으로 캐스트
        null -> println("Null value")
    }
}
```

## 💻 실전 예제

### ViewModel에서 활용

```kotlin
class UserViewModel : ViewModel() {
    private val _user = MutableStateFlow<User?>(null)
    val user: StateFlow<User?> = _user.asStateFlow()
    
    fun loadUser(id: String) {
        viewModelScope.launch {
            _user.value = repository.getUser(id)
        }
    }
}

// Compose에서
@Composable
fun UserScreen(viewModel: UserViewModel) {
    val user by viewModel.user.collectAsStateWithLifecycle()
    
    user?.let { 
        Text("Welcome, ${it.name}")
    } ?: Text("Loading...")
}
```

### Repository 패턴

```kotlin
class UserRepository {
    suspend fun getUser(id: String): User? {
        return try {
            api.getUser(id)
        } catch (e: Exception) {
            null
        }
    }
    
    // Result 패턴 활용 (더 좋음)
    suspend fun getUserSafe(id: String): Result<User> {
        return runCatching {
            api.getUser(id)
        }
    }
}
```

### Safe navigation in chains

```kotlin
data class Company(val address: Address?)
data class Address(val street: Street?)
data class Street(val name: String?)

fun getStreetName(company: Company?): String {
    return company?.address?.street?.name ?: "Unknown"
}
```

## 🔗 관련 개념

- [[01-Kotlin/기본문법/변수와-타입]]
- [[01-Kotlin/함수형-프로그래밍/Scope-Functions]]

## 📚 더 보기

- [Kotlin Docs - Null Safety](https://kotlinlang.org/docs/null-safety.html)

---

**핵심 요약:** ?로 nullable 명시, ?.로 안전 호출, ?:로 기본값, !!는 가급적 쓰지 말 것.
