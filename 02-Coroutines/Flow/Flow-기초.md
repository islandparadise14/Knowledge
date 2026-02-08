---
tags: [coroutines, flow, stream, reactive]
---

# Flow 기초

## 💡 핵심 개념

**Flow**는 비동기로 값을 연속적으로 방출하는 스트림이다. Cold stream으로 collect될 때만 시작되고, 여러 연산자로 변환 가능하다.

## 📌 왜 필요한가?

단일 값이 아닌 연속적인 값(검색, 센서 데이터, DB 변경)을 처리할 때 필요하다. RxJava보다 간단하고 코루틴과 잘 통합된다.

## 🔍 자세히

### 기본 사용법

```kotlin
// Flow 생성
fun simpleFlow(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)
        emit(i)  // 값 방출
    }
}

// Flow 수집
viewModelScope.launch {
    simpleFlow().collect { value ->
        println(value)  // 1, 2, 3
    }
}
```

### Flow 빌더

```kotlin
// flow { } - 가장 기본
val flow1 = flow {
    emit(1)
    emit(2)
}

// flowOf - 고정된 값
val flow2 = flowOf(1, 2, 3)

// asFlow - 컬렉션을 Flow로
val flow3 = listOf(1, 2, 3).asFlow()

// channelFlow - 동시성 필요 시
val flow4 = channelFlow {
    launch { send(1) }
    launch { send(2) }
}
```

### Cold vs Hot

```kotlin
// Cold - collect할 때마다 새로 시작
val coldFlow = flow {
    println("Started")
    emit(1)
}

coldFlow.collect { }  // "Started" 출력
coldFlow.collect { }  // "Started" 다시 출력

// Hot - StateFlow, SharedFlow (항상 활성)
val hotFlow = MutableStateFlow(1)
hotFlow.collect { }  // 즉시 1 받음
hotFlow.collect { }  // 즉시 1 받음
```

### 중간 연산자

**map**

```kotlin
flow { emit(1); emit(2) }
    .map { it * 2 }
    .collect { println(it) }  // 2, 4
```

**filter**

```kotlin
(1..10).asFlow()
    .filter { it % 2 == 0 }
    .collect { println(it) }  // 2, 4, 6, 8, 10
```

**transform**

```kotlin
(1..3).asFlow()
    .transform { value ->
        emit("String $value")
        emit(value)
    }
    .collect { println(it) }
    // String 1, 1, String 2, 2, String 3, 3
```

**take**

```kotlin
(1..10).asFlow()
    .take(3)
    .collect { println(it) }  // 1, 2, 3
```

### 종단 연산자

**collect**

```kotlin
flow.collect { value ->
    println(value)
}
```

**toList**

```kotlin
val list = flow.toList()
```

**first / firstOrNull**

```kotlin
val first = flow.first()
val firstOrNull = flow.firstOrNull()
```

**reduce / fold**

```kotlin
val sum = (1..5).asFlow().reduce { acc, value -> acc + value }  // 15
val sum2 = (1..5).asFlow().fold(10) { acc, value -> acc + value }  // 25
```

### 예외 처리

**catch**

```kotlin
flow {
    emit(1)
    throw Exception("Error")
    emit(2)
}.catch { e ->
    println("Caught: $e")
    emit(-1)
}.collect { println(it) }
// 1, Caught: java.lang.Exception: Error, -1
```

**onCompletion**

```kotlin
flow { emit(1) }
    .onCompletion { cause ->
        if (cause != null) println("Error: $cause")
        else println("Completed")
    }
    .collect { println(it) }
```

### Context 변경

**flowOn**

```kotlin
fun getData(): Flow<Int> = flow {
    // IO 스레드에서 실행
    emit(fetchFromDb())
}.flowOn(Dispatchers.IO)

// Main에서 collect
viewModelScope.launch {
    getData().collect { data ->
        // Main 스레드
        updateUI(data)
    }
}
```

### 동작 원리

**Flow는 suspend 함수의 시퀀스**

```kotlin
interface Flow<out T> {
    suspend fun collect(collector: FlowCollector<T>)
}

fun interface FlowCollector<in T> {
    suspend fun emit(value: T)
}
```

**Flow 빌더 구현**

```kotlin
fun <T> flow(block: suspend FlowCollector<T>.() -> Unit): Flow<T> {
    return object : Flow<T> {
        override suspend fun collect(collector: FlowCollector<T>) {
            collector.block()
        }
    }
}
```

### 주의사항

**1. collect는 suspend 함수**

```kotlin
// 나쁨 - 일반 함수에서 호출 불가
fun bad() {
    // flow.collect { }  // ERROR
}

// 좋음 - 코루틴 안에서
viewModelScope.launch {
    flow.collect { }
}
```

**2. Flow는 순차적**

```kotlin
flow {
    emit(1)
    delay(100)
    emit(2)
}
.map {
    delay(100)
    it * 2
}
.collect { println(it) }
// 총 400ms (순차 실행)
```

**3. Context 보존**

```kotlin
// 나쁨 - Flow 빌더 안에서 withContext 금지
flow {
    withContext(Dispatchers.IO) {  // 예외 발생!
        emit(1)
    }
}

// 좋음 - flowOn 사용
flow {
    emit(1)
}.flowOn(Dispatchers.IO)
```

## 💻 실전 예제

### Room with Flow

```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    fun getAllUsers(): Flow<List<User>>
}

class UserRepository(private val dao: UserDao) {
    val users: Flow<List<User>> = dao.getAllUsers()
}

// ViewModel
class UserViewModel : ViewModel() {
    val users = repository.users
        .map { it.filter { user -> user.isActive } }
        .stateIn(
            viewModelScope,
            SharingStarted.WhileSubscribed(5000),
            emptyList()
        )
}
```

### 검색 기능

```kotlin
val searchQuery = MutableStateFlow("")

val searchResults = searchQuery
    .debounce(300)  // 300ms 대기
    .filter { it.length >= 2 }  // 2글자 이상
    .distinctUntilChanged()  // 중복 제거
    .flatMapLatest { query ->
        searchRepository.search(query)
    }
    .catch { emit(emptyList()) }
    .stateIn(
        viewModelScope,
        SharingStarted.WhileSubscribed(),
        emptyList()
    )
```

### 여러 Flow 결합

```kotlin
// combine - 둘 중 하나라도 방출하면 결합
val combined = combine(flow1, flow2) { a, b ->
    "$a $b"
}

// zip - 쌍으로 결합
val zipped = flow1.zip(flow2) { a, b ->
    "$a $b"
}
```

### Retry

```kotlin
fun fetchData(): Flow<Data> = flow {
    emit(api.getData())
}.retry(3) { cause ->
    (cause is IOException).also {
        if (it) delay(1000)
    }
}
```

### SharedFlow 사용

```kotlin
class EventBus {
    private val _events = MutableSharedFlow<Event>()
    val events: SharedFlow<Event> = _events.asSharedFlow()
    
    suspend fun post(event: Event) {
        _events.emit(event)
    }
}
```

## 🔗 관련 개념

- [[02-Coroutines/Flow/StateFlow]]
- [[02-Coroutines/Flow/SharedFlow]]
- [[02-Coroutines/Flow/Flow-연산자-변환]]

## 📚 더 보기

- [Kotlin Docs - Flow](https://kotlinlang.org/docs/flow.html)

---

**핵심 요약:** 비동기 스트림. Cold stream. collect로 수집. 다양한 연산자. flowOn으로 context 변경.
