---
tags: [coroutines, launch, job]
---

# Launch

## 💡 핵심 개념

**launch**는 결과를 반환하지 않는 코루틴을 시작한다. Job 객체를 반환해서 코루틴을 제어(취소, 대기)할 수 있다.

## 📌 왜 필요한가?

백그라운드 작업을 실행하고 결과가 필요없을 때 사용한다. UI 업데이트, 로깅, 데이터 동기화 등에 적합하다.

## 🔍 자세히

### 기본 사용법

```kotlin
// 기본 launch
val job = launch {
    println("Coroutine started")
    delay(1000)
    println("Coroutine finished")
}

// Job 제어
job.cancel()  // 취소
job.join()  // 완료 대기
```

### launch vs async

```kotlin
// launch - 결과 반환 X
launch {
    updateUI()
    saveToDb()
}

// async - 결과 반환 O
val deferred = async {
    fetchData()
}
val result = deferred.await()
```

### Job 상태

```kotlin
val job = launch {
    delay(1000)
}

println(job.isActive)  // true
println(job.isCompleted)  // false
println(job.isCancelled)  // false

job.cancel()
println(job.isCancelled)  // true
```

### Job 계층

```kotlin
val parentJob = launch {
    val childJob1 = launch {
        delay(1000)
        println("Child 1")
    }
    
    val childJob2 = launch {
        delay(2000)
        println("Child 2")
    }
    
    println("Parent waiting")
}

// 부모는 모든 자식이 끝날 때까지 대기
parentJob.join()
```

### 취소 처리

```kotlin
val job = launch {
    try {
        repeat(1000) { i ->
            println("Job: $i")
            delay(500)
        }
    } catch (e: CancellationException) {
        println("Cancelled")
        // 정리 작업
    } finally {
        println("Finally")
    }
}

delay(1300)
job.cancel()
```

### 동작 원리

**Job은 상태 머신**

```
                  New
                   |
             [start/join]
                   |
              Active ──> Cancelling ──> Cancelled
                   |          |              |
              [complete]  [cancel]     [finish]
                   |          |              |
             Completing ──────┴──────> Completed
```

**부모-자식 관계**

```kotlin
val parent = launch {
    launch { /* child 1 */ }
    launch { /* child 2 */ }
}

// parent.cancel() → 모든 자식도 취소
// 자식 하나 실패 → parent도 취소 (일반 Job)
// 자식 하나 실패 → parent는 계속 (SupervisorJob)
```

### 주의사항

**1. GlobalScope 금지**

```kotlin
// 나쁨 - 생명주기 관리 안 됨
GlobalScope.launch {
    // Activity 종료돼도 계속 실행
}

// 좋음
viewModelScope.launch {
    // ViewModel 종료되면 자동 취소
}
```

**2. 취소 가능하게 작성**

```kotlin
// 나쁨 - 취소 안 됨
launch {
    while (true) {
        doWork()  // delay 없음
    }
}

// 좋음 - isActive 체크
launch {
    while (isActive) {
        doWork()
        delay(100)
    }
}
```

**3. Exception 전파**

```kotlin
// launch는 예외를 전파
val job = launch {
    throw Exception("Error")
}
// 예외가 부모로 전파 → 부모도 취소
```

## 💻 실전 예제

### ViewModel에서

```kotlin
class UserViewModel : ViewModel() {
    fun loadUser(id: String) {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val user = repository.getUser(id)
                _uiState.value = UiState.Success(user)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message)
            }
        }
    }
}
```

### 병렬 실행

```kotlin
suspend fun loadData() = coroutineScope {
    // 두 작업 동시 실행
    launch { loadUsers() }
    launch { loadPosts() }
    // coroutineScope는 모든 자식이 끝날 때까지 대기
}
```

### Job 저장 및 취소

```kotlin
class DataLoader {
    private var loadJob: Job? = null
    
    fun startLoading() {
        loadJob?.cancel()  // 이전 작업 취소
        loadJob = viewModelScope.launch {
            loadData()
        }
    }
    
    fun stopLoading() {
        loadJob?.cancel()
    }
}
```

### 주기적 작업

```kotlin
fun startPeriodicUpdate() {
    viewModelScope.launch {
        while (isActive) {
            updateData()
            delay(5000)  // 5초마다
        }
    }
}
```

### Compose에서

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel) {
    val scope = rememberCoroutineScope()
    
    Button(onClick = {
        scope.launch {
            viewModel.loadUser()
        }
    }) {
        Text("Load User")
    }
}
```

### 여러 Job 관리

```kotlin
class TaskManager {
    private val jobs = mutableListOf<Job>()
    
    fun addTask(task: suspend () -> Unit) {
        val job = viewModelScope.launch {
            task()
        }
        jobs.add(job)
    }
    
    fun cancelAll() {
        jobs.forEach { it.cancel() }
        jobs.clear()
    }
}
```

## 🔗 관련 개념

- [[02-Coroutines/빌더/Async-Await]]
- [[02-Coroutines/기초/Job]]
- [[02-Coroutines/취소/취소-메커니즘]]

## 📚 더 보기

- [Kotlin Docs - launch](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html)

---

**핵심 요약:** 결과 반환 X. Job으로 제어. fire-and-forget 스타일. GlobalScope 금지.
