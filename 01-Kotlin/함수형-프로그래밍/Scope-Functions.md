---
tags: [kotlin, scope-functions, let, run, with, apply, also]
---

# Scope Functions

## 💡 핵심 개념

**Scope functions**은 객체의 컨텍스트 내에서 코드 블록을 실행하는 함수들이다. `let`, `run`, `with`, `apply`, `also` 5가지가 있다.

## 📌 왜 필요한가?

객체 초기화, null 체크, 임시 스코프 생성 등을 간결하게 처리할 수 있다. 하지만 과용하면 가독성이 떨어지니 적절히 사용해야 한다.

## 🔍 자세히

### 비교표

| 함수 | 객체 참조 | 반환값 | 용도 |
|------|----------|--------|------|
| **let** | it | 람다 결과 | null 체크 & 변환 |
| **run** | this | 람다 결과 | 객체 설정 & 결과 계산 |
| **with** | this | 람다 결과 | non-null 객체 여러 호출 |
| **apply** | this | 객체 자신 | 객체 초기화 |
| **also** | it | 객체 자신 | 추가 작업 (로깅 등) |

### let

```kotlin
// null이 아닐 때만 실행
val name: String? = getName()
name?.let {
    println("Name length: ${it.length}")
}

// 변환
val length = name?.let {
    it.uppercase().length
}

// 여러 nullable 체크
val user = getUser()
val address = getAddress()
user?.let { u ->
    address?.let { a ->
        println("${u.name} lives at ${a.city}")
    }
}
```

**언제 사용?**
- nullable 값에 대한 안전한 작업
- 지역 변수 스코프 제한
- 체이닝

### run

```kotlin
// 객체 설정 후 결과 반환
val result = service.run {
    port = 8080
    host = "localhost"
    start()  // 반환값
}

// 식 그룹화
val formatted = user.run {
    val fullName = "$firstName $lastName"
    val age = calculateAge(birthDate)
    "$fullName ($age years old)"
}
```

**언제 사용?**
- 객체 초기화 + 결과 계산
- 여러 메서드 호출 후 결과 필요

### with

```kotlin
// non-null 객체의 여러 메서드 호출
val result = with(user) {
    println(name)
    println(age)
    println(email)
    "Processed: $name"
}

// StringBuilder 같은 경우
val message = with(StringBuilder()) {
    append("Hello")
    append(" ")
    append("World")
    toString()
}
```

**언제 사용?**
- 리시버 객체의 여러 메서드 호출
- 결과값 필요
- nullable이 아닌 경우

### apply

```kotlin
// 객체 초기화 (자기 자신 반환)
val user = User().apply {
    name = "민트"
    age = 27
    email = "mint@example.com"
}

// Intent 초기화
val intent = Intent(this, DetailActivity::class.java).apply {
    putExtra("id", userId)
    putExtra("name", userName)
}
startActivity(intent)

// View 초기화
val textView = TextView(context).apply {
    text = "Hello"
    textSize = 16f
    setTextColor(Color.BLACK)
}
```

**언제 사용?**
- 객체 초기화 (빌더 패턴 대체)
- 설정 후 객체 자체가 필요한 경우

### also

```kotlin
// 추가 작업 (로깅, 검증 등)
val user = getUser().also {
    println("User fetched: ${it.name}")
}

// 체이닝 중간에 디버깅
val result = numbers
    .filter { it > 0 }
    .also { println("Filtered: $it") }
    .map { it * 2 }
    .also { println("Mapped: $it") }

// 검증
val validated = user.also {
    require(it.age >= 18) { "User must be adult" }
}
```

**언제 사용?**
- 로깅, 디버깅
- 객체에 대한 추가 작업
- 부수 효과 (side effect)

### 동작 원리

**구현**

```kotlin
// let
public inline fun <T, R> T.let(block: (T) -> R): R {
    return block(this)
}

// run
public inline fun <T, R> T.run(block: T.() -> R): R {
    return block()
}

// with
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    return receiver.block()
}

// apply
public inline fun <T> T.apply(block: T.() -> Unit): T {
    block()
    return this
}

// also
public inline fun <T> T.also(block: (T) -> Unit): T {
    block(this)
    return this
}
```

**this vs it**

```kotlin
// this - 리시버 객체 (생략 가능)
user.run {
    name  // this.name과 동일
    this.name  // 명시적
}

// it - 람다 파라미터 (이름 변경 가능)
user.let { u ->
    u.name
}
```

### 주의사항

**1. 과용하지 말 것**

```kotlin
// 나쁨 - 중첩 과다
user?.let {
    it.address?.let {
        it.city?.let {
            println(it)
        }
    }
}

// 좋음
println(user?.address?.city)
```

**2. 반환값 주의**

```kotlin
val result = user.apply {
    name = "Updated"
}
// result는 user 객체

val result2 = user.run {
    name = "Updated"
}
// result2는 Unit
```

**3. 중첩 사용 시 this 명시**

```kotlin
class Outer {
    val outerProp = "outer"
    
    fun foo() {
        Inner().apply {
            this@Outer.outerProp  // Outer의 프로퍼티
            this.innerProp         // Inner의 프로퍼티
        }
    }
}
```

## 💻 실전 예제

### Android View 초기화

```kotlin
// apply
val textView = TextView(context).apply {
    text = "Title"
    textSize = 18f
    setTextColor(Color.BLACK)
    gravity = Gravity.CENTER
}

// also (로깅 추가)
val button = Button(context).apply {
    text = "Click me"
    setOnClickListener { /* ... */ }
}.also {
    println("Button created")
}
```

### ViewModel 상태 업데이트

```kotlin
// let으로 null 체크
fun loadUser(id: String?) {
    id?.let {
        viewModelScope.launch {
            _uiState.update { state ->
                state.copy(user = repository.getUser(it))
            }
        }
    }
}

// run으로 여러 검증
fun validateAndSubmit(form: Form) {
    form.run {
        require(name.isNotBlank()) { "Name required" }
        require(age >= 18) { "Must be adult" }
        require(email.contains("@")) { "Invalid email" }
        submit()
    }
}
```

### Repository 패턴

```kotlin
suspend fun getUser(id: String): Result<User> {
    return runCatching {
        api.getUser(id)
    }.also {
        it.onSuccess { user -> 
            cache.put(id, user)
            println("Cached user: ${user.name}")
        }
    }.map {
        it.toDomain()
    }
}
```

### Builder 패턴 대체

```kotlin
// 전통적 Builder
class Request private constructor(
    val url: String,
    val method: String,
    val headers: Map<String, String>
) {
    class Builder {
        private var url: String = ""
        private var method: String = "GET"
        private val headers = mutableMapOf<String, String>()
        
        fun url(url: String) = apply { this.url = url }
        fun method(method: String) = apply { this.method = method }
        fun header(key: String, value: String) = apply { 
            headers[key] = value 
        }
        fun build() = Request(url, method, headers)
    }
}

// apply 사용 (더 간결)
data class Request(
    var url: String = "",
    var method: String = "GET",
    val headers: MutableMap<String, String> = mutableMapOf()
)

val request = Request().apply {
    url = "https://api.example.com"
    method = "POST"
    headers["Content-Type"] = "application/json"
}
```

### 조건부 실행

```kotlin
// takeIf/takeUnless와 함께
val validUser = user
    .takeIf { it.age >= 18 }
    ?.also { println("Valid user: ${it.name}") }

// let으로 변환
val userName = user
    .takeIf { it.isActive }
    ?.let { it.name.uppercase() }
    ?: "Unknown"
```

### Intent 빌더

```kotlin
inline fun <reified T : Activity> Context.intentFor(
    extras: Intent.() -> Unit = {}
): Intent {
    return Intent(this, T::class.java).apply(extras)
}

// 사용
val intent = intentFor<DetailActivity> {
    putExtra("id", userId)
    putExtra("name", userName)
    flags = Intent.FLAG_ACTIVITY_NEW_TASK
}
startActivity(intent)
```

## 🔗 관련 개념

- [[01-Kotlin/함수형-프로그래밍/고차함수]]
- [[01-Kotlin/함수형-프로그래밍/람다-표현식]]
- [[01-Kotlin/기본문법/Nullable과-Null-Safety]]

## 📚 더 보기

- [Kotlin Docs - Scope Functions](https://kotlinlang.org/docs/scope-functions.html)

---

**핵심 요약:** let(null 체크), run(계산), with(여러 호출), apply(초기화), also(부수효과). 과용 금지!
