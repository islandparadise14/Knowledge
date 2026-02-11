---
tags: [CS, 동시성, 블로킹, 논블로킹, Coroutine, Android]
---

# 블로킹 vs 논블로킹

## 💡 핵심 개념

**블로킹(Blocking)**은 호출된 함수가 작업을 완료할 때까지 호출한 스레드의 제어권을 돌려주지 않는 방식이다. **논블로킹(Non-blocking)**은 호출된 함수가 즉시 제어권을 반환하여 호출한 스레드가 다른 작업을 계속할 수 있는 방식이다. 코루틴의 `suspend`는 스레드를 블로킹하지 않으면서 실행을 일시 중단하는 논블로킹 방식의 핵심이다.

## 📌 왜 필요한가?

Android 메인 스레드가 블로킹되면 16ms 내에 프레임을 그리지 못해 UI가 버벅이거나 ANR이 발생한다. 논블로킹 방식을 이해하면 메인 스레드를 효율적으로 활용하면서 백그라운드 작업을 수행할 수 있다.

## 🔍 자세히

### 블로킹과 논블로킹 비교

| 구분 | 블로킹 | 논블로킹 |
|------|--------|----------|
| 제어권 | 작업 완료까지 반환하지 않음 | 즉시 반환 |
| 스레드 상태 | 대기 상태 (idle but occupied) | 다른 작업 수행 가능 |
| 리소스 효율 | 낮음 (스레드가 점유됨) | 높음 (스레드 재활용) |
| Android 예시 | `Thread.sleep()`, JDBC 호출 | `delay()`, `suspend` 함수 |

### 블로킹의 문제: Thread.sleep vs delay

```kotlin
// ❌ 블로킹: Thread.sleep() - 스레드 자체가 멈춤
fun blockingExample() {
    println("시작: ${Thread.currentThread().name}")
    Thread.sleep(1000) // 이 스레드는 1초간 아무것도 못 함
    println("끝: ${Thread.currentThread().name}")
}

// ✅ 논블로킹: delay() - 코루틴만 일시 중단, 스레드는 자유
suspend fun nonBlockingExample() {
    println("시작: ${Thread.currentThread().name}")
    delay(1000) // 코루틴 일시 중단, 스레드는 다른 코루틴 실행 가능
    println("끝: ${Thread.currentThread().name}")
}
```

### 블로킹이 Android에서 위험한 이유

```kotlin
// ❌ 메인 스레드 블로킹 - UI 프레임 드롭
class BadActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 메인 스레드에서 블로킹 I/O 호출
        val data = URL("https://api.example.com/data").readText() // 블로킹!
        // 이 시간 동안 UI 이벤트 처리 불가 -> ANR
    }
}

// ✅ 논블로킹으로 수정
class GoodActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleScope.launch {
            val data = withContext(Dispatchers.IO) {
                URL("https://api.example.com/data").readText()
            }
            // IO 스레드에서 블로킹되지만, 메인 스레드는 자유
            textView.text = data
        }
    }
}
```

### runBlocking vs coroutineScope

```kotlin
// runBlocking: 현재 스레드를 블로킹하고 코루틴 실행
// 테스트나 main 함수에서만 사용해야 함
fun main() = runBlocking {
    // 메인 스레드가 블로킹됨
    val result = async { fetchData() }
    println(result.await())
}

// coroutineScope: 논블로킹으로 코루틴 스코프 생성
// suspend 함수 내에서 사용
suspend fun loadData() = coroutineScope {
    val user = async { fetchUser() }
    val posts = async { fetchPosts() }
    // 현재 코루틴만 일시 중단, 스레드는 자유
    Pair(user.await(), posts.await())
}
```

### Dispatchers와 블로킹 관계

```kotlin
lifecycleScope.launch {
    // Dispatchers.Main: 메인 스레드 - 절대 블로킹하면 안 됨
    // UI 업데이트만 수행

    withContext(Dispatchers.IO) {
        // 블로킹 I/O 작업을 여기서 수행
        // 최대 64개 스레드가 블로킹 I/O를 처리
        database.query("SELECT * FROM users")
    }

    withContext(Dispatchers.Default) {
        // CPU 집약적 작업 (블로킹이 아닌 연산)
        // CPU 코어 수만큼의 스레드
        heavyJsonParsing(largeData)
    }
}
```

### 논블로킹 I/O와 채널

```kotlin
// Channel을 활용한 논블로킹 생산자-소비자 패턴
val channel = Channel<Int>(capacity = Channel.BUFFERED)

// 생산자: 논블로킹으로 데이터 전송
launch(Dispatchers.Default) {
    repeat(100) {
        channel.send(it) // 버퍼가 꽉 차면 일시 중단 (블로킹 아님)
    }
    channel.close()
}

// 소비자: 논블로킹으로 데이터 수신
launch(Dispatchers.Main) {
    for (item in channel) {
        updateProgress(item) // 메인 스레드에서 UI 업데이트
    }
}
```

### 동기/비동기와의 관계

```
동기 + 블로킹:     일반적인 함수 호출 (Thread.sleep)
동기 + 논블로킹:   바쁜 대기 (polling, spin-lock)
비동기 + 블로킹:   Node.js의 일부 패턴 (실무에서 드묾)
비동기 + 논블로킹:  코루틴, Flow, Channel (가장 효율적)
```

## 🔗 관련 개념

- [[00-CS-기초/동시성/동기-vs-비동기|동기 vs 비동기]]
- [[00-CS-기초/동시성/프로세스-vs-스레드|프로세스 vs 스레드]]
- [[00-CS-기초/동시성/스레드-풀|스레드 풀]]
- [[00-CS-기초/동시성/동시성-vs-병렬성|동시성 vs 병렬성]]

## 📚 더 보기

- [Coroutines and Channels](https://kotlinlang.org/docs/coroutines-and-channels.html)
- [Android Performance: Keeping the UI Thread Responsive](https://developer.android.com/topic/performance/threads)

---

**핵심 요약:** 블로킹은 스레드 제어권을 작업 완료까지 붙잡는 방식이고, 논블로킹은 즉시 제어권을 반환하는 방식으로, 코루틴의 suspend는 스레드를 블로킹하지 않고 코루틴만 일시 중단하여 Android 메인 스레드의 응답성을 유지한다.