---
tags: [kotlin, coroutines, structured-concurrency]
---

# Structured Concurrency

## 💡 핵심 개념

**구조화된 동시성(Structured Concurrency)**은 코루틴의 생명주기를 구조적으로 관리하는 원칙이다. 부모가 취소되면 자식도 자동으로 취소된다.

## 📌 왜 필요한가?

코루틴 누수를 방지하고, 예측 가능한 동작을 보장한다. 메모리 누수와 리소스 낭비를 막는다.

## 🔍 자세히

### 부모-자식 관계

```kotlin
viewModelScope.launch {  // 부모
    launch {  // 자식 1
        delay(1000)
        println("Child 1")
    }
    
    launch {  // 자식 2
        delay(2000)
        println("Child 2")
    }
    
    println("Parent")
}

// viewModelScope가 취소되면 자식들도 모두 취소됨
```

### Job vs SupervisorJob

**Job - 하나 실패하면 전체 취소**

```kotlin
val job = Job()
val scope = CoroutineScope(job)

scope.launch {
    launch {
        throw Exception("Error")  // 예외 발생
    }
    
    launch {
        delay(5000)
        println("This won't print")  // 취소됨
    }
}
```

**SupervisorJob - 독립적 실패**

```kotlin
val supervisorJob = SupervisorJob()
val scope = CoroutineScope(supervisorJob)

scope.launch {
    launch {
        throw Exception("Error")  // 예외 발생
    }
    
    launch {
        delay(1000)
        println("This will print")  // 계속 실행
    }
}
```

### coroutineScope vs supervisorScope

**coroutineScope - 하나 실패하면 전체 취소**

```kotlin
suspend fun loadData() = coroutineScope {
    val user = async { loadUser() }
    val posts = async { loadPosts() }
    
    // posts가 실패하면 user도 취소됨
    Pair(user.await(), posts.await())
}
```

**supervisorScope - 독립적 실패**

```kotlin
suspend fun loadData() = supervisorScope {
    val user = async { 
        try {
            loadUser()
        } catch (e: Exception) {
            null  // 실패해도 계속
        }
    }
    
    val posts = async {
        try {
            loadPosts()
        } catch (e: Exception) {
            emptyList()  // 실패해도 계속
        }
    }
    
    Pair(user.await(), posts.await())
}
```

### ViewModelScope

```kotlin
// viewModelScope = Dispatchers.Main.immediate + SupervisorJob()
class UserViewModel : ViewModel() {
    fun loadUser() {
        viewModelScope.launch {
            // ViewModel이 clear되면 자동 취소
        }
    }
}

// onCleared()에서 자동으로 취소됨
```

### LifecycleScope

```kotlin
class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // lifecycleScope - Lifecycle에 종속
        lifecycleScope.launch {
            // Fragment destroy되면 자동 취소
        }
        
        // viewLifecycleOwner.lifecycleScope
        viewLifecycleOwner.lifecycleScope.launch {
            // View destroy되면 자동 취소
        }
    }
}
```

### 주의사항

**1. GlobalScope 사용 금지**

```kotlin
// 나쁜 예 - 메모리 누수!
GlobalScope.launch {
    // 이것은 앱이 종료될 때까지 실행됨
}

// 좋은 예
viewModelScope.launch {
    // ViewModel이 clear되면 자동 취소
}
```

**2. 직접 Job 만들기**

```kotlin
class MyRepository {
    private val scope = CoroutineScope(
        Dispatchers.IO + SupervisorJob()
    )
    
    fun loadData() {
        scope.launch {
            // ...
        }
    }
    
    fun cleanup() {
        scope.cancel()  // 모든 작업 취소
    }
}
```

## 💻 실전 예제

### ViewModel with SupervisorJob

```kotlin
class UserViewModel : ViewModel() {
    // viewModelScope는 이미 SupervisorJob 사용
    
    fun loadAll() {
        viewModelScope.launch {
            // 각각 독립적으로 실패 가능
            launch { loadUser() }
            launch { loadPosts() }
            launch { loadComments() }
        }
    }
}
```

### Repository with Custom Scope

```kotlin
class UserRepository {
    private val ioScope = CoroutineScope(
        Dispatchers.IO + SupervisorJob()
    )
    
    suspend fun syncData() = supervisorScope {
        val user = async { 
            try { syncUser() } 
            catch (e: Exception) { null }
        }
        
        val posts = async {
            try { syncPosts() }
            catch (e: Exception) { emptyList() }
        }
        
        SyncResult(user.await(), posts.await())
    }
    
    fun cleanup() {
        ioScope.cancel()
    }
}
```

### 부분 실패 허용

```kotlin
suspend fun loadDashboard(): Dashboard = supervisorScope {
    val userData = async {
        try {
            loadUserData()
        } catch (e: Exception) {
            Log.e("Dashboard", "User data failed", e)
            UserData.EMPTY
        }
    }
    
    val statsData = async {
        try {
            loadStatsData()
        } catch (e: Exception) {
            Log.e("Dashboard", "Stats failed", e)
            StatsData.EMPTY
        }
    }
    
    Dashboard(
        user = userData.await(),
        stats = statsData.await()
    )
}
```

## 🔗 관련 개념

- [[02-Coroutines/기초/Job]]
- [[02-Coroutines/구조화된-동시성/Job-vs-SupervisorJob]]
- [[02-Coroutines/구조화된-동시성/ViewModelScope-LifecycleScope]]

## 📚 더 보기

- [Kotlin Docs - Structured Concurrency](https://kotlinlang.org/docs/coroutines-basics.html#structured-concurrency)
- [Android Docs - Coroutines Best Practices](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)

---

**핵심 요약:** 부모 취소 → 자식 취소. Job은 전체 실패, SupervisorJob은 독립 실패. viewModelScope 사용.
