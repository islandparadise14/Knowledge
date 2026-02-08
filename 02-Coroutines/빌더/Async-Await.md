---
tags: [coroutines, async, await, deferred]
---

# Async-Await

## 💡 핵심 개념

**async**는 결과를 반환하는 코루틴을 시작한다. Deferred 객체를 반환하고, `await()`로 결과를 받는다. 병렬 실행에 유용하다.

## 📌 왜 필요한가?

여러 비동기 작업을 병렬로 실행하고 모든 결과를 모을 때 사용한다. launch보다 결과값이 필요할 때 적합하다.

## 🔍 자세히

### 기본 사용법

```kotlin
val deferred = async {
    delay(1000)
    "Result"
}

val result = deferred.await()  // "Result"
```

### async vs launch

```kotlin
// launch - 결과 X
launch {
    doSomething()
}

// async - 결과 O
val result = async {
    fetchData()
}.await()
```

### 병렬 실행

```kotlin
suspend fun loadData() = coroutineScope {
    // 순차 실행 - 3초
    val user = fetchUser()  // 1초
    val posts = fetchPosts()  // 1초
    val comments = fetchComments()  // 1초
    
    // 병렬 실행 - 1초
    val userDeferred = async { fetchUser() }
    val postsDeferred = async { fetchPosts() }
    val commentsDeferred = async { fetchComments() }
    
    val user = userDeferred.await()
    val posts = postsDeferred.await()
    val comments = commentsDeferred.await()
}
```

### awaitAll

```kotlin
suspend fun fetchUsers(ids: List<String>): List<User> = coroutineScope {
    ids.map { id ->
        async { api.getUser(id) }
    }.awaitAll()
}
```

### 지연 시작 (Lazy)

```kotlin
val deferred = async(start = CoroutineStart.LAZY) {
    expensiveComputation()
}

// await() 호출 시 시작
val result = deferred.await()

// 또는 명시적 시작
deferred.start()
val result = deferred.await()
```

### 동작 원리

**Deferred는 Job + 결과**

```kotlin
public interface Deferred<out T> : Job {
    suspend fun await(): T
    val onAwait: SelectClause1<T>
}

// 사용
val deferred: Deferred<String> = async { "Result" }
println(deferred.isActive)  // Job 기능
val result: String = deferred.await()  // 결과 반환
```

**예외 처리**

```kotlin
val deferred = async {
    throw Exception("Error")
}

// await() 호출 시 예외 발생
try {
    deferred.await()
} catch (e: Exception) {
    println("Caught: $e")
}
```

### 주의사항

**1. coroutineScope 필수**

```kotlin
// 나쁨 - Deferred가 누수될 수 있음
suspend fun bad() {
    val deferred = async { fetchData() }
    // await() 안 하면 취소 안 됨
}

// 좋음 - coroutineScope 사용
suspend fun good() = coroutineScope {
    val deferred = async { fetchData() }
    deferred.await()
}
```

**2. async는 즉시 시작**

```kotlin
// async는 바로 시작됨
val deferred = async {
    println("Started")  // 즉시 출력
    delay(1000)
}

// await()는 결과만 기다림
delay(500)
deferred.await()  // 나머지 500ms만 대기
```

**3. 순서대로 await 권장**

```kotlin
// 좋음 - 동시 실행
val d1 = async { fetch1() }
val d2 = async { fetch2() }
d1.await()
d2.await()

// 나쁨 - 순차 실행됨
val r1 = async { fetch1() }.await()  // 기다림
val r2 = async { fetch2() }.await()  // 그 다음 기다림
```

**4. GlobalScope 금지**

```kotlin
// 나쁨
GlobalScope.async {
    // 생명주기 관리 안 됨
}

// 좋음
viewModelScope.async {
    // ViewModel 종료되면 자동 취소
}
```

## 💻 실전 예제

### 병렬 API 호출

```kotlin
suspend fun loadUserProfile(id: String) = coroutineScope {
    val userDeferred = async { api.getUser(id) }
    val postsDeferred = async { api.getUserPosts(id) }
    val followersDeferred = async { api.getUserFollowers(id) }
    
    UserProfile(
        user = userDeferred.await(),
        posts = postsDeferred.await(),
        followers = followersDeferred.await()
    )
}
```

### 여러 URL 다운로드

```kotlin
suspend fun downloadFiles(urls: List<String>): List<ByteArray> = coroutineScope {
    urls.map { url ->
        async(Dispatchers.IO) {
            downloadFile(url)
        }
    }.awaitAll()
}
```

### 타임아웃과 함께

```kotlin
suspend fun fetchWithTimeout() = coroutineScope {
    val deferred = async {
        delay(5000)
        "Result"
    }
    
    try {
        withTimeout(3000) {
            deferred.await()
        }
    } catch (e: TimeoutCancellationException) {
        deferred.cancel()
        "Timeout"
    }
}
```

### 조건부 병렬 실행

```kotlin
suspend fun loadData(loadPosts: Boolean) = coroutineScope {
    val user = async { fetchUser() }
    val posts = if (loadPosts) {
        async { fetchPosts() }
    } else {
        null
    }
    
    UserData(
        user = user.await(),
        posts = posts?.await()
    )
}
```

### Repository 패턴

```kotlin
class DataRepository {
    suspend fun syncData() = coroutineScope {
        val usersDeferred = async { syncUsers() }
        val postsDeferred = async { syncPosts() }
        val commentsDeferred = async { syncComments() }
        
        SyncResult(
            users = usersDeferred.await(),
            posts = postsDeferred.await(),
            comments = commentsDeferred.await()
        )
    }
}
```

### 에러 처리

```kotlin
suspend fun loadData() = coroutineScope {
    try {
        val d1 = async { fetch1() }
        val d2 = async { fetch2() }
        
        listOf(d1.await(), d2.await())
    } catch (e: Exception) {
        // 하나라도 실패하면 여기로
        emptyList()
    }
}
```

### Compose에서

```kotlin
@Composable
fun DataScreen() {
    var data by remember { mutableStateOf<List<Data>>(emptyList()) }
    
    LaunchedEffect(Unit) {
        data = loadDataParallel()
    }
}

suspend fun loadDataParallel() = coroutineScope {
    listOf(
        async { fetchData1() },
        async { fetchData2() },
        async { fetchData3() }
    ).awaitAll().flatten()
}
```

### 결과 조합

```kotlin
suspend fun getRecommendations(userId: String) = coroutineScope {
    val (user, history, trending) = awaitAll(
        async { getUser(userId) },
        async { getUserHistory(userId) },
        async { getTrendingItems() }
    )
    
    calculateRecommendations(user, history, trending)
}
```

## 🔗 관련 개념

- [[02-Coroutines/빌더/Launch]]
- [[02-Coroutines/실전-패턴/병렬-처리]]
- [[02-Coroutines/취소/취소-메커니즘]]

## 📚 더 보기

- [Kotlin Docs - async](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html)

---

**핵심 요약:** 결과 반환. Deferred로 제어. 병렬 실행에 유용. awaitAll()로 여러 결과 수집.
