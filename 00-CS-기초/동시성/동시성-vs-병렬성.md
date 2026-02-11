---
tags: [CS, 동시성, 병렬성, Coroutine, Dispatchers, Android]
---

# 동시성 vs 병렬성

## 💡 핵심 개념

**동시성(Concurrency)**은 여러 작업이 논리적으로 동시에 진행되는 것처럼 보이게 하는 구조로, 단일 코어에서도 컨텍스트 스위칭을 통해 구현된다. **병렬성(Parallelism)**은 여러 작업이 물리적으로 동시에 실행되는 것으로, 멀티코어 CPU가 필요하다. 코루틴은 동시성을 제공하며, `Dispatchers.Default`는 멀티코어를 활용한 병렬 실행을 지원한다.

## 📌 왜 필요한가?

Android 기기는 대부분 멀티코어 프로세서를 탑재하고 있어 병렬성을 활용할 수 있지만, 코루틴의 동시성 모델을 이해해야 `Dispatcher` 선택과 구조화된 동시성(Structured Concurrency)을 올바르게 활용할 수 있다.

## 🔍 자세히

### 동시성과 병렬성 비교

| 구분 | 동시성 (Concurrency) | 병렬성 (Parallelism) |
|------|----------------------|----------------------|
| 정의 | 여러 작업을 번갈아 처리 | 여러 작업을 동시에 처리 |
| 필요 조건 | 단일 코어로도 가능 | 멀티코어 필수 |
| 목적 | 구조와 응답성 개선 | 처리량(throughput) 향상 |
| 비유 | 한 요리사가 여러 요리를 번갈아 조리 | 여러 요리사가 각각 요리를 동시에 조리 |
| Android 예시 | 코루틴, Main 스레드의 이벤트 루프 | Dispatchers.Default의 워커 스레드들 |

### 시각적 비교

```
동시성 (단일 코어):
코어1: [작업A] [작업B] [작업A] [작업B] [작업A]
       <- 빠르게 번갈아 실행하여 동시에 보임 ->

병렬성 (멀티 코어):
코어1: [작업A ████████████████████]
코어2: [작업B ████████████████████]
       <- 실제로 동시에 실행됨 ->
```

### 코루틴의 동시성

```kotlin
// 코루틴은 기본적으로 동시성을 제공한다
// 단일 스레드(Main)에서도 여러 코루틴이 동시에 실행 가능

lifecycleScope.launch(Dispatchers.Main) {
    // 이 두 코루틴은 메인 스레드에서 동시에(concurrently) 실행됨
    val job1 = launch {
        repeat(5) {
            println("코루틴1: $it")
            delay(100) // 일시 중단 -> 다른 코루틴에게 실행 기회
        }
    }

    val job2 = launch {
        repeat(5) {
            println("코루틴2: $it")
            delay(100) // 일시 중단 -> 다른 코루틴에게 실행 기회
        }
    }

    // 출력: 코루틴1: 0, 코루틴2: 0, 코루틴1: 1, 코루틴2: 1, ...
    // 단일 스레드에서 번갈아 실행됨 (동시성)
}
```

### Dispatchers와 병렬성

```kotlin
lifecycleScope.launch {
    // Dispatchers.Default: CPU 코어 수만큼의 스레드 풀
    // 실제 병렬 실행이 가능
    val results = withContext(Dispatchers.Default) {
        val deferred1 = async { cpuIntensiveWork1() } // 코어1에서 실행
        val deferred2 = async { cpuIntensiveWork2() } // 코어2에서 실행
        val deferred3 = async { cpuIntensiveWork3() } // 코어3에서 실행

        // 세 작업이 물리적으로 동시에 실행 (병렬성)
        listOf(deferred1.await(), deferred2.await(), deferred3.await())
    }
    updateUI(results)
}
```

### 구조화된 동시성 (Structured Concurrency)

```kotlin
// 코루틴의 구조화된 동시성: 부모-자식 관계로 생명주기 관리
suspend fun loadDashboard(): DashboardData = coroutineScope {
    // 세 작업이 동시에 실행됨
    val userDeferred = async { fetchUser() }
    val statsDeferred = async { fetchStats() }
    val notificationsDeferred = async { fetchNotifications() }

    // 하나라도 실패하면 나머지도 자동 취소
    DashboardData(
        user = userDeferred.await(),
        stats = statsDeferred.await(),
        notifications = notificationsDeferred.await()
    )
}

// ViewModel에서 사용
class DashboardViewModel : ViewModel() {
    fun load() {
        viewModelScope.launch {
            try {
                val data = loadDashboard()
                _uiState.value = UiState.Success(data)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message)
            }
        }
    }
}
```

### 동시성 제어: 제한된 병렬성

```kotlin
// 동시에 실행할 코루틴 수를 제한
val limitedDispatcher = Dispatchers.IO.limitedParallelism(3)

suspend fun downloadImages(urls: List<String>) = coroutineScope {
    urls.map { url ->
        async(limitedDispatcher) {
            // 최대 3개만 동시에 다운로드
            downloadImage(url)
        }
    }.awaitAll()
}

// Semaphore를 활용한 동시성 제어
val semaphore = Semaphore(5) // 최대 5개 동시 실행

suspend fun rateLimitedApiCall(id: Int): Result {
    semaphore.withPermit {
        return api.getData(id)
    }
}
```

## 🔗 관련 개념

- [[00-CS-기초/동시성/프로세스-vs-스레드|프로세스 vs 스레드]]
- [[00-CS-기초/동시성/스레드-풀|스레드 풀]]
- [[00-CS-기초/동시성/동기-vs-비동기|동기 vs 비동기]]
- [[00-CS-기초/동시성/경쟁-조건|경쟁 조건]]
- [[00-CS-기초/동시성/뮤텍스-세마포어|뮤텍스와 세마포어]]

## 📚 더 보기

- [Structured Concurrency](https://kotlinlang.org/docs/coroutines-basics.html#structured-concurrency)
- [Coroutine Context and Dispatchers](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)

---

**핵심 요약:** 동시성은 여러 작업을 논리적으로 동시에 다루는 구조이고 병렬성은 물리적으로 동시에 실행하는 것으로, 코루틴은 동시성을 기본 제공하며 Dispatchers.Default를 통해 멀티코어 병렬 실행도 활용할 수 있다.