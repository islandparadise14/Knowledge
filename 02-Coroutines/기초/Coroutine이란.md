---
tags: [kotlin, coroutines, async, suspend]
---

# Coroutine이란

## 💡 핵심 개념

**Coroutine**은 중단 가능한 계산(suspendable computation)이다. 함수 실행을 중단했다가 나중에 재개할 수 있다. 경량 스레드처럼 동작하지만 실제 스레드보다 훨씬 가볍다.

## 📌 왜 필요한가?

비동기 작업을 동기 코드처럼 작성할 수 있다. 콜백 지옥을 피하고 코드 가독성을 높인다. 스레드보다 훨씬 적은 비용으로 수천 개의 동시 작업 가능하다.

## 🔍 자세히

### 기본 사용법

```kotlin
// suspend 함수 - 중단 가능
suspend fun fetchUser(id: String): User {
    delay(1000)  // 1초 중단 (스레드는 블록 안 됨)
    return api.getUser(id)
}

// Coroutine 실행
fun loadUser() {
    viewModelScope.launch {  // 코루틴 시작
        val user = fetchUser("123")  // 중단 지점
        updateUI(user)
    }
}
```

### 스레드 vs 코루틴

```kotlin
// 스레드 - 무거움
thread {
    Thread.sleep(1000)  // 스레드 블록
    println("Done")
}
// 10,000개 스레드 → 메모리 부족

// 코루틴 - 가벼움
GlobalScope.launch {
    delay(1000)  // 중단 (스레드 블록 X)
    println("Done")
}
// 10,000개 코루틴 → 문제 없음
```

### suspend 함수

```kotlin
// suspend 키워드 - 코루틴 안에서만 호출 가능
suspend fun fetchData(): String {
    delay(1000)
    return "Data"
}

// 일반 함수에서는 호출 불가
fun normalFunction() {
    // val data = fetchData()  // ERROR
    
    // 코루틴 빌더 필요
    viewModelScope.launch {
        val data = fetchData()  // OK
    }
}

// suspend 함수는 suspend 함수에서 호출 가능
suspend fun processData() {
    val data = fetchData()  // OK
    println(data)
}
```

### 코루틴 빌더

```kotlin
// launch - 결과 반환 안 함
val job = launch {
    // 작업 수행
}

// async - 결과 반환
val deferred = async {
    fetchUser("123")
}
val user = deferred.await()

// runBlocking - 현재 스레드 블록 (메인 함수, 테스트에서만)
runBlocking {
    val result = fetchData()
    println(result)
}

// withContext - 컨텍스트 전환
val result = withContext(Dispatchers.IO) {
    // IO 작업
}
```

### 동작 원리

**Continuation — 실제 인터페이스**

`Continuation`은 개념이 아니라 Kotlin 표준 라이브러리에 실제로 존재하는 인터페이스다.

```kotlin
// Kotlin 표준 라이브러리 실제 코드
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}
```

컴파일러가 `suspend` 함수마다 이걸 구현한 클래스를 자동 생성한다.

```kotlin
// 네가 작성한 코드
suspend fun fetchUser(id: String): User {
    delay(1000)
    return api.getUser(id)
}

// 컴파일러가 실제로 만드는 것 (대략)
class FetchUserContinuation(
    val id: String,
    var label: Int = 0,
    var result: Any? = null
) : Continuation<User> {
    override fun resumeWith(result: Result<User>) {
        this.result = result
        // label 보고 다음 단계 실행
    }
}
```

**State Machine**

```kotlin
// 컴파일러가 변환
suspend fun example() {
    println("Before")
    delay(1000)
    println("After")
}

// 대략적으로 이렇게 변환됨
fun example(continuation: Continuation<Unit>): Any {
    when (continuation.label) {
        0 -> {
            println("Before")
            continuation.label = 1
            return delay(1000, continuation)  // COROUTINE_SUSPENDED
        }
        1 -> {
            println("After")
            return Unit
        }
    }
}
```

```kotlin
suspend fun multi() {
    println("1")
    delay(100)
    println("2")
    delay(100)
    println("3")
}

// State machine으로 변환:
// State 0: println("1") → delay
// State 1: println("2") → delay
// State 2: println("3") → done
```

**스레드 블록이 없는 이유 — 스택 vs 힙**

스레드는 모든 상태를 자신의 스택에 들고 있어서 기다리는 동안 스레드 자체가 묶인다.
코루틴은 상태를 힙의 Continuation 객체로 꺼내놓기 때문에 스레드를 즉시 반환할 수 있다.

```
스레드
  [스택]
    fetchUser()
      userId = "123"
      result = ???    ← 상태가 스택에 있음
      sleep(1000) 대기중
                      ← 스택을 다른 스레드에 못 넘김
                         스레드 자체가 들고 있어야 함


코루틴
  [스택]              [힙]
    fetchUser()
    delay(1000)  →   Continuation {
    리턴!              label = 1
                       userId = "123"  ← 상태를 힙으로 이사
                     }
  [스택 비워짐]        ↓
                     메시지 큐에 등록
                     "1초 후 이 Continuation 실행해줘"

                     ↓ 1초 후

  [스택]              [힙]
    fetchUser()  ←   Continuation     ← 힙에서 꺼내서
    label=1          label = 1          스택에 다시 올림
    재개!
```

**delay()와 Handler 메시지 큐**

```
스레드:   [────────── 스레드가 시간을 점유 ──────────]
           작업        멈춤(1초)                작업

코루틴:   [코루틴A]   [코루틴B][msg][코루틴A 재개]
           작업        ← 스레드는 계속 다른 일 함 →
                  ↑
           여기서 스레드 반환 + Continuation을 메시지 큐에 등록
```

`delay(1000)` 내부에서 일어나는 일:

```kotlin
// delay 내부 (단순화)
Handler.postDelayed({
    continuation.resumeWith(Result.success(Unit))  // 그냥 객체 메서드 호출
}, 1000)
return COROUTINE_SUSPENDED  // 스레드 즉시 반환
```

`continuation.resumeWith()`는 **그냥 객체 메서드 호출 하나**다.
스레드를 깨우거나 OS를 건드리는 게 없다. 힙에 있는 Continuation을 꺼내서 다음 label부터 실행할 뿐이다.

### 주의사항

**1. Main Thread 블록 금지**

```kotlin
// 나쁨 - Main Thread 블록
fun loadData() {
    runBlocking {  // Main Thread 멈춤!
        delay(1000)
    }
}

// 좋음
fun loadData() {
    viewModelScope.launch {  // 비동기
        delay(1000)
    }
}
```

**2. suspend는 자동으로 비동기가 아님**

```kotlin
suspend fun fetchUsers(): List<User> {
    // 여전히 순차 실행
    val user1 = api.getUser("1")  // 1초
    val user2 = api.getUser("2")  // 1초
    return listOf(user1, user2)  // 총 2초
}

// 병렬 실행하려면 async 사용
suspend fun fetchUsersParallel(): List<User> {
    return coroutineScope {
        val user1 = async { api.getUser("1") }
        val user2 = async { api.getUser("2") }
        listOf(user1.await(), user2.await())  // 총 1초
    }
}
```

**3. GlobalScope 사용 금지**

```kotlin
// 나쁨 - 생명주기 관리 안 됨
GlobalScope.launch {
    // Activity 종료돼도 계속 실행
}

// 좋음 - 생명주기 관리
viewModelScope.launch {
    // ViewModel 종료되면 자동 취소
}
```

## 💻 실전 예제

### ViewModel에서 사용

```kotlin
class UserViewModel : ViewModel() {
    private val _users = MutableStateFlow<List<User>>(emptyList())
    val users: StateFlow<List<User>> = _users.asStateFlow()
    
    fun loadUsers() {
        viewModelScope.launch {
            try {
                val users = repository.getUsers()
                _users.value = users
            } catch (e: Exception) {
                // 에러 처리
            }
        }
    }
}
```

### Repository 패턴

```kotlin
class UserRepository(
    private val api: UserApi,
    private val dao: UserDao
) {
    suspend fun getUser(id: String): User {
        // IO 스레드로 전환
        return withContext(Dispatchers.IO) {
            try {
                // 네트워크 요청
                api.getUser(id).also {
                    // DB 저장
                    dao.insert(it)
                }
            } catch (e: Exception) {
                // 실패 시 캐시에서
                dao.getUser(id) ?: throw e
            }
        }
    }
}
```

### 순차 vs 병렬 실행

```kotlin
// 순차 실행
suspend fun loadData() {
    val user = fetchUser()  // 1초
    val posts = fetchPosts()  // 1초
    // 총 2초
}

// 병렬 실행
suspend fun loadDataParallel() = coroutineScope {
    val user = async { fetchUser() }
    val posts = async { fetchPosts() }
    
    val userData = user.await()
    val postsData = posts.await()
    // 총 1초
}
```

### 타임아웃 처리

```kotlin
suspend fun fetchWithTimeout() {
    try {
        withTimeout(3000) {
            val data = fetchData()
            println(data)
        }
    } catch (e: TimeoutCancellationException) {
        println("Timeout!")
    }
}
```

### Compose에서 사용

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel) {
    val users by viewModel.users.collectAsStateWithLifecycle()
    
    LaunchedEffect(Unit) {
        // Composable 생명주기에 맞춰 코루틴 실행
        viewModel.loadUsers()
    }
    
    LazyColumn {
        items(users) { user ->
            UserItem(user)
        }
    }
}
```

### 반복 작업

```kotlin
// 주기적 작업
fun startPeriodicTask() {
    viewModelScope.launch {
        while (isActive) {
            fetchLatestData()
            delay(5000)  // 5초마다
        }
    }
}

// Flow로 더 깔끔하게
fun periodicFlow() = flow {
    while (true) {
        emit(fetchLatestData())
        delay(5000)
    }
}
```

## 🔗 관련 개념

- [[02-Coroutines/기초/CoroutineContext]]
- [[02-Coroutines/기초/Dispatcher]]
- [[02-Coroutines/빌더/Launch]]
- [[02-Coroutines/빌더/Async-Await]]

## 📚 더 보기

- [Kotlin Docs - Coroutines](https://kotlinlang.org/docs/coroutines-overview.html)
- [Android Developers - Coroutines](https://developer.android.com/kotlin/coroutines)

---

**핵심 요약:** suspend 함수로 중단 가능한 작업. 가볍고 효율적. 동기 코드처럼 작성 가능. GlobalScope 금지.
