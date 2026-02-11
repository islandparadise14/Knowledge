---
tags: [CS, 동시성, 동기, 비동기, Coroutine, Android]
---

# 동기 vs 비동기

## 💡 핵심 개념

**동기(Synchronous)**는 작업이 순차적으로 실행되어 하나의 작업이 완료될 때까지 다음 작업이 대기하는 방식이다. **비동기(Asynchronous)**는 작업의 완료를 기다리지 않고 다음 작업을 즉시 실행하며, 결과는 나중에 콜백이나 코루틴 등을 통해 받는 방식이다. Android에서는 `suspend` 함수, `Flow`, `callback` 등이 비동기 처리의 핵심이다.

## 📌 왜 필요한가?

Android 메인 스레드는 UI를 그리는 역할을 하므로 네트워크, DB, 파일 I/O 같은 시간이 오래 걸리는 작업을 동기적으로 처리하면 화면이 멈추고 ANR이 발생한다. 비동기 프로그래밍을 통해 사용자 경험을 해치지 않으면서 백그라운드 작업을 처리할 수 있다.

## 🔍 자세히

### 동기와 비동기 비교

| 구분 | 동기 (Synchronous) | 비동기 (Asynchronous) |
|------|---------------------|------------------------|
| 실행 흐름 | 순차적, 블로킹 | 비순차적, 논블로킹 |
| 대기 여부 | 결과를 받을 때까지 대기 | 결과를 기다리지 않음 |
| 코드 복잡도 | 단순하고 직관적 | 콜백 지옥 가능 (코루틴으로 해결) |
| Android 사용처 | 간단한 계산, SharedPreferences 읽기 | 네트워크, DB, 파일 I/O |

### 동기 방식의 문제점

```kotlin
// ❌ 메인 스레드에서 동기 호출 - ANR 발생!
fun loadUserSync(): User {
    val response = api.getUser() // 네트워크 응답까지 스레드가 블로킹됨
    return response.body()
}

// 메인 스레드에서 호출하면 UI가 멈춤
val user = loadUserSync()
textView.text = user.name
```

### 콜백 기반 비동기 (전통적 방식)

```kotlin
// 콜백 패턴 - 중첩되면 "콜백 지옥" 발생
fun loadUserAsync(callback: (User) -> Unit) {
    api.getUser().enqueue(object : Callback<User> {
        override fun onResponse(call: Call<User>, response: Response<User>) {
            response.body()?.let { user ->
                // 사용자 정보를 가져온 후 추가 요청
                api.getPosts(user.id).enqueue(object : Callback<List<Post>> {
                    override fun onResponse(call: Call<List<Post>>, response: Response<List<Post>>) {
                        callback(user) // 콜백 중첩 -> 가독성 저하
                    }
                    override fun onFailure(call: Call<List<Post>>, t: Throwable) { }
                })
            }
        }
        override fun onFailure(call: Call<User>, t: Throwable) { }
    })
}
```

### suspend 함수 (코루틴 비동기)

```kotlin
// ✅ 코루틴 suspend 함수 - 비동기지만 동기처럼 읽힌다
suspend fun loadUser(): User {
    return withContext(Dispatchers.IO) {
        api.getUser() // suspend 함수: 스레드를 블로킹하지 않고 일시 중단
    }
}

// 호출부
lifecycleScope.launch {
    val user = loadUser()          // 일시 중단 후 재개
    val posts = loadPosts(user.id) // 순차적으로 읽히지만 비동기
    updateUI(user, posts)          // 메인 스레드에서 UI 업데이트
}
```

### 병렬 비동기 처리 (async/await)

```kotlin
lifecycleScope.launch {
    // 두 요청을 동시에 실행
    val userDeferred = async(Dispatchers.IO) { api.getUser() }
    val configDeferred = async(Dispatchers.IO) { api.getConfig() }

    // 두 결과가 모두 완료될 때까지 대기
    val user = userDeferred.await()
    val config = configDeferred.await()

    updateUI(user, config)
}
```

### Flow를 활용한 비동기 스트림

```kotlin
// 연속적인 데이터 스트림을 비동기로 처리
fun observeLocationUpdates(): Flow<Location> = callbackFlow {
    val callback = object : LocationCallback() {
        override fun onLocationResult(result: LocationResult) {
            trySend(result.lastLocation)
        }
    }
    locationClient.requestLocationUpdates(request, callback, Looper.getMainLooper())
    awaitClose { locationClient.removeLocationUpdates(callback) }
}

// 수집
lifecycleScope.launch {
    observeLocationUpdates()
        .flowOn(Dispatchers.IO)
        .collect { location ->
            updateMapUI(location) // 메인 스레드에서 UI 업데이트
        }
}
```

### 동기 vs 비동기 실행 흐름

```
동기:    [작업A ██████████] -> [작업B ██████████] -> [작업C ██████████]
비동기:  [작업A ██████████]
         [작업B ██████████]    -> 모두 완료 후 결과 처리
         [작업C ██████████]
```

## 🔗 관련 개념

- [[00-CS-기초/동시성/블로킹-vs-논블로킹|블로킹 vs 논블로킹]]
- [[00-CS-기초/동시성/동시성-vs-병렬성|동시성 vs 병렬성]]
- [[00-CS-기초/동시성/프로세스-vs-스레드|프로세스 vs 스레드]]
- [[00-CS-기초/동시성/스레드-풀|스레드 풀]]

## 📚 더 보기

- [Kotlin Coroutines on Android](https://developer.android.com/kotlin/coroutines)
- [Kotlin Flow 공식 문서](https://kotlinlang.org/docs/flow.html)
- [Suspend Functions](https://kotlinlang.org/docs/composing-suspending-functions.html)

---

**핵심 요약:** 동기는 작업 완료를 기다리는 순차 실행이고, 비동기는 결과를 기다리지 않고 진행하는 방식으로, Android에서는 코루틴의 suspend 함수와 Flow를 통해 비동기 코드를 동기처럼 직관적으로 작성할 수 있다.