---
tags: [coroutines, context, job, dispatcher]
---

# CoroutineContext

## 💡 핵심 개념

**CoroutineContext**는 코루틴의 실행 환경을 정의한다. Job, Dispatcher, CoroutineName, ExceptionHandler 등의 요소들로 구성된다.

## 📌 왜 필요한가?

코루틴이 어디서(Dispatcher), 어떻게(Job), 무슨 이름으로(CoroutineName) 실행될지 결정한다. 부모-자식 관계, 예외 처리, 취소 등을 관리한다.

## 🔍 자세히

### 기본 구조

```kotlin
// CoroutineContext는 요소들의 집합
launch(Dispatchers.Main + CoroutineName("MyCoroutine")) {
    println(coroutineContext[Job])  // Job 요소 접근
    println(coroutineContext[CoroutineDispatcher])  // Dispatcher 접근
}
```

### 주요 요소

**1. Job**

```kotlin
val job = Job()  // 코루틴의 생명주기 관리

launch(job) {
    // ...
}

job.cancel()  // 취소
job.join()  // 완료 대기
```

**2. Dispatcher**

```kotlin
// Main 스레드
launch(Dispatchers.Main) { }

// IO 작업용
launch(Dispatchers.IO) { }

// CPU 집약적 작업
launch(Dispatchers.Default) { }

// 현재 스레드
launch(Dispatchers.Unconfined) { }
```

**3. CoroutineName**

```kotlin
launch(CoroutineName("DataLoader")) {
    println(coroutineContext[CoroutineName])  // CoroutineName(DataLoader)
}
```

**4. CoroutineExceptionHandler**

```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
    println("Caught $exception")
}

launch(handler) {
    throw Exception("Error!")
}
```

### Context 결합 (+)

```kotlin
val context = Dispatchers.IO + CoroutineName("Worker") + Job()

launch(context) {
    // Dispatcher.IO에서 실행, Worker라는 이름, 새 Job
}

// 덮어쓰기
val context1 = Dispatchers.Main + CoroutineName("A")
val context2 = context1 + CoroutineName("B")  // "A"가 "B"로 변경
```

### Context 상속

```kotlin
// 부모 컨텍스트 상속
val parentContext = Dispatchers.Main + CoroutineName("Parent")

launch(parentContext) {
    println(coroutineContext[CoroutineName])  // Parent
    
    // 자식은 부모 컨텍스트 상속
    launch {
        println(coroutineContext[CoroutineName])  // Parent (상속)
    }
    
    // 일부만 오버라이드
    launch(CoroutineName("Child")) {
        println(coroutineContext[CoroutineName])  // Child
        println(coroutineContext[CoroutineDispatcher])  // Main (상속)
    }
}
```

### withContext

```kotlin
// 임시로 컨텍스트 전환
suspend fun loadData() {
    // Main에서 실행 중이라고 가정
    val data = withContext(Dispatchers.IO) {
        // IO 스레드로 전환
        fetchFromNetwork()
    }
    // 다시 Main으로 복귀
    updateUI(data)
}
```

### 동작 원리

**Element 인터페이스**

```kotlin
public interface CoroutineContext {
    public operator fun <E : Element> get(key: Key<E>): E?
    public operator fun plus(context: CoroutineContext): CoroutineContext
    
    public interface Element : CoroutineContext {
        public val key: Key<*>
    }
}

// 사용
val job: Job? = coroutineContext[Job]
val dispatcher: CoroutineDispatcher? = coroutineContext[CoroutineDispatcher]
```

**Context 병합**

```kotlin
// + 연산자로 병합
val context = EmptyCoroutineContext +
    Dispatchers.IO +
    CoroutineName("Worker") +
    Job()

// 같은 Key는 덮어씀
val c1 = CoroutineName("A")
val c2 = CoroutineName("B")
val combined = c1 + c2  // c2가 c1을 덮어씀
```

### 주의사항

**1. Context 상속과 Job**

```kotlin
val parentJob = Job()

launch(parentJob) {
    val childJob = coroutineContext[Job]
    println(childJob?.parent == parentJob)  // true
    
    // 부모가 취소되면 자식도 취소
    parentJob.cancel()
}
```

**2. withContext vs launch**

```kotlin
// withContext - 순차 실행, 같은 코루틴
suspend fun sequential() {
    val result1 = withContext(Dispatchers.IO) { fetch1() }
    val result2 = withContext(Dispatchers.IO) { fetch2() }
}

// launch - 병렬 실행, 새 코루틴
suspend fun parallel() = coroutineScope {
    val job1 = launch(Dispatchers.IO) { fetch1() }
    val job2 = launch(Dispatchers.IO) { fetch2() }
    job1.join()
    job2.join()
}
```

**3. Unconfined Dispatcher 주의**

```kotlin
launch(Dispatchers.Unconfined) {
    println("Thread: ${Thread.currentThread().name}")  // main
    delay(100)
    println("Thread: ${Thread.currentThread().name}")  // kotlinx.coroutines.DefaultExecutor
    // 첫 suspend 후 스레드가 바뀔 수 있음!
}
```

## 💻 실전 예제

### Custom CoroutineScope

```kotlin
class MyRepository {
    // 커스텀 스코프
    private val scope = CoroutineScope(
        Dispatchers.IO +
        SupervisorJob() +
        CoroutineName("RepositoryScope")
    )
    
    fun loadData() {
        scope.launch {
            // ...
        }
    }
    
    fun cleanup() {
        scope.cancel()
    }
}
```

### ThreadLocal 전달

```kotlin
val threadLocal = ThreadLocal<String>()

val context = threadLocal.asContextElement(value = "data")

launch(context) {
    println(threadLocal.get())  // "data"
    
    withContext(Dispatchers.Default) {
        println(threadLocal.get())  // "data" (전달됨)
    }
}
```

### Debugging

```kotlin
// 디버깅 정보 추가
launch(CoroutineName("DataLoader")) {
    println(coroutineContext)
    // [CoroutineName(DataLoader), StandaloneCoroutine{Active}@..., Dispatchers.Default]
}

// 프로덕션에서는 -Dkotlinx.coroutines.debug JVM 옵션
// Thread[main @DataLoader#1,5,main]
```

### Context Element 추가

```kotlin
class UserIdContext(val userId: String) : AbstractCoroutineContextElement(UserIdContext) {
    companion object Key : CoroutineContext.Key<UserIdContext>
}

launch(UserIdContext("user123")) {
    val userId = coroutineContext[UserIdContext]?.userId
    println("User: $userId")
}
```

### Android ViewModel

```kotlin
class MyViewModel : ViewModel() {
    // viewModelScope = ViewModel의 생명주기 + Main dispatcher
    fun loadData() {
        viewModelScope.launch {
            println(coroutineContext[Job])  // ViewModelCoroutineScope의 Job
            println(coroutineContext[CoroutineDispatcher])  // Dispatchers.Main
        }
    }
}
```

### 에러 처리와 Context

```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
    logError(exception)
}

viewModelScope.launch(handler + CoroutineName("DataLoader")) {
    // 에러 발생 시 handler로 전달
}
```

## 🔗 관련 개념

- [[02-Coroutines/기초/Job]]
- [[02-Coroutines/기초/Dispatcher]]
- [[02-Coroutines/구조화된-동시성/Structured-Concurrency]]
- [[02-Coroutines/예외-처리/CoroutineExceptionHandler]]

## 📚 더 보기

- [Kotlin Docs - Coroutine Context](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)

---

**핵심 요약:** Job + Dispatcher + Name + ExceptionHandler. + 연산자로 결합. 자식은 부모 컨텍스트 상속.
