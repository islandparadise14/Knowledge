---
tags: [coroutines, dispatcher, handler, looper, thread-pool, 내부동작]
---

# Dispatchers 내부 구현

## 💡 핵심 개념

코루틴의 Dispatcher는 내부적으로 **기존 Android 스레딩 메커니즘** 위에 구축되어 있다. `Dispatchers.Main`은 Android의 `Handler(Looper.getMainLooper())`를 사용하고, `Dispatchers.Default`와 `Dispatchers.IO`는 **같은 스레드 풀을 공유**하되 동시 실행 수를 다르게 제한한다.

## 📌 왜 필요한가?

`withContext(Dispatchers.Main)`이 어떻게 UI 스레드에서 실행되는지, IO와 Default가 왜 같은 풀인데 분리되는지, Handler와 Coroutine이 어떻게 연결되는지를 알아야 성능 이슈와 데드락을 진단할 수 있다.

## 🔍 자세히

### Dispatchers.Main의 내부

`Dispatchers.Main`은 `HandlerDispatcher`로 구현되어 있다. 핵심은 **Handler.post()** 를 통해 코루틴 블록을 메인 스레드의 MessageQueue에 넣는 것이다.

```kotlin
// kotlinx-coroutines-android 내부 (단순화)
internal class HandlerDispatcher(
    private val handler: Handler
) : CoroutineDispatcher() {

    override fun dispatch(context: CoroutineContext, block: Runnable) {
        // 코루틴 블록을 Handler.post()로 메인 스레드에 전달
        handler.post(block)
    }

    override fun isDispatchNeeded(context: CoroutineContext): Boolean {
        // 이미 메인 스레드면 dispatch 불필요
        return Looper.myLooper() != handler.looper
    }
}

// Dispatchers.Main = HandlerDispatcher(Handler(Looper.getMainLooper()))
```

**동작 흐름:**

```
코루틴에서 withContext(Dispatchers.Main) 호출
     │
     ▼
HandlerDispatcher.dispatch() 호출
     │
     ▼
handler.post(block)
     │
     ▼
MessageQueue에 Message 삽입
     │
     ▼
Looper.loop()이 Message를 꺼내 실행
     │
     ▼
메인 스레드에서 코루틴 블록 실행
```

### Dispatchers.Main.immediate

```kotlin
// Main: 항상 Handler.post()로 큐에 넣음
// Main.immediate: 이미 메인 스레드면 즉시 실행 (post 안 함)

// 차이가 보이는 예시
fun onButtonClick() {  // 이미 메인 스레드
    lifecycleScope.launch(Dispatchers.Main) {
        // Handler.post() → MessageQueue → 다음 루프에서 실행
        println("1: Main")
    }
    lifecycleScope.launch(Dispatchers.Main.immediate) {
        // 이미 메인 스레드 → 즉시 실행
        println("2: Main.immediate")
    }
    println("3: 직접 실행")
}
// 출력 순서: 3 → 2 → 1
// immediate는 큐를 거치지 않으므로 먼저 실행

// lifecycleScope.launch는 기본이 Dispatchers.Main.immediate
```

### Dispatchers.Default와 IO의 공유 풀

```
┌──────────────────────────────────────────────┐
│           공유 스레드 풀 (CoroutineScheduler)  │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │  Worker Thread 1                        │  │
│  │  Worker Thread 2                        │  │
│  │  Worker Thread 3    ← Default 전용      │  │
│  │  Worker Thread 4       (코어 수만큼)     │  │
│  │  ....                                   │  │
│  │  Worker Thread 5    ← IO 추가 할당      │  │
│  │  Worker Thread 6       (최대 64개)       │  │
│  │  ....                                   │  │
│  │  Worker Thread 64                       │  │
│  └─────────────────────────────────────────┘  │
│                                               │
│  Default: 코어 수만큼만 동시 실행 허용         │
│  IO: 최대 64개까지 동시 실행 허용              │
│  같은 풀이지만 동시 실행 "한도"가 다름          │
└──────────────────────────────────────────────┘
```

```kotlin
// 내부 구현 (단순화)
object DefaultScheduler : CoroutineDispatcher() {
    // CPU 코어 수만큼만 동시 실행
    val corePoolSize = Runtime.getRuntime().availableProcessors()
    // 최대 스레드 수 (IO와 공유)
    val maxPoolSize = max(64, corePoolSize)

    // Default: corePoolSize개까지만 동시 실행
    // IO: maxPoolSize개까지 동시 실행
}

// 왜 공유인가?
// Default에서 IO로 withContext 전환 시 실제 스레드 전환 없이
// "동시 실행 한도"만 변경 → 컨텍스트 스위칭 비용 최소화
```

### withContext의 스레드 전환 비용

```kotlin
// Default → IO 전환: 실제 스레드 전환 안 일어날 수 있음
withContext(Dispatchers.Default) {
    val sorted = list.sorted()  // CPU 작업 (Default 한도 사용)

    withContext(Dispatchers.IO) {
        dao.insert(sorted)  // IO 작업 (IO 한도로 전환)
        // 같은 스레드에서 실행될 수 있음! (한도만 변경)
    }
}

// Main ↔ IO 전환: 실제 스레드 전환 발생
withContext(Dispatchers.Main) {
    // 메인 스레드
    val data = withContext(Dispatchers.IO) {
        // 워커 스레드로 전환 (실제 스레드 변경)
        api.fetchData()
    }
    // 다시 메인 스레드로 전환
    textView.text = data
}
```

### Dispatchers.Main이 사용하는 Handler/Looper 구조

```
┌──────────────────────────────────────┐
│          메인 스레드                   │
│                                       │
│  ┌─────────────────────────────────┐  │
│  │  Looper.loop() (무한 루프)       │  │
│  │       │                          │  │
│  │       ▼                          │  │
│  │  MessageQueue에서 Message 꺼냄   │  │
│  │       │                          │  │
│  │       ├─ Input Event (터치)      │  │
│  │       ├─ View 그리기             │  │
│  │       ├─ Handler.post() 콜백     │  │
│  │       └─ 코루틴 블록 (Dispatchers.Main) ← 여기!  │
│  │                                  │  │
│  │  msg.target.handleMessage(msg)   │  │
│  └─────────────────────────────────┘  │
└──────────────────────────────────────┘
```

코루틴 블록도 결국 MessageQueue의 Message 중 하나로 처리된다. 터치 이벤트, View 렌더링과 같은 큐에서 순서를 기다린다.

### limitedParallelism의 내부

```kotlin
// limitedParallelism은 부모 Dispatcher의 스레드를 공유하되
// 동시 실행 수만 제한하는 래퍼
val dbDispatcher = Dispatchers.IO.limitedParallelism(4)
// → IO 풀의 스레드를 사용하지만, 동시에 4개까지만 실행

val singleThread = Dispatchers.Default.limitedParallelism(1)
// → Default 풀의 스레드 1개만 사용 (순차 실행 보장)
// → newSingleThreadContext()보다 효율적 (전용 스레드 안 만듦)
```

### runBlocking과 Handler 충돌

```kotlin
// ❌ 메인 스레드에서 runBlocking + Dispatchers.Main = 데드락!
class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 메인 스레드를 블로킹하면서
        runBlocking {
            // 메인 스레드에 dispatch하려고 시도 → 데드락
            withContext(Dispatchers.Main) {
                textView.text = "Never reached"
            }
        }
    }
}
// 원인: runBlocking이 메인 스레드를 점유
// → Handler.post()가 MessageQueue에 넣지만
// → Looper가 현재 메시지(runBlocking) 처리 중이라 꺼내지 못함
// → 서로 영원히 대기 = 데드락

// ✅ 해결: runBlocking 대신 lifecycleScope.launch 사용
lifecycleScope.launch {
    withContext(Dispatchers.Main) {
        textView.text = "OK"
    }
}
```

### Choreographer와의 관계

```kotlin
// Android의 프레임 렌더링도 Handler 기반
// Choreographer.postFrameCallback() → 다음 VSYNC에 실행

// 코루틴이 메인 스레드를 오래 점유하면
// Choreographer의 프레임 콜백이 밀려 → 프레임 드롭(jank)

lifecycleScope.launch(Dispatchers.Main) {
    // ❌ 메인 스레드에서 무거운 연산
    val result = heavyComputation()  // 50ms 소요
    // → 3프레임 드롭 (16ms × 3)

    // ✅ CPU 작업은 Default로
    val result = withContext(Dispatchers.Default) {
        heavyComputation()
    }
    textView.text = result  // Main에서 UI만 업데이트
}
```

## 💻 실전 예제

### Dispatcher 확인 디버깅

```kotlin
fun logDispatcher(tag: String) {
    val thread = Thread.currentThread().name
    val isMain = Looper.myLooper() == Looper.getMainLooper()
    Log.d("Dispatcher", "$tag: thread=$thread, isMain=$isMain")
}

lifecycleScope.launch {
    logDispatcher("launch")      // main, isMain=true
    withContext(Dispatchers.IO) {
        logDispatcher("IO")      // DefaultDispatcher-worker-1, isMain=false
    }
    logDispatcher("after IO")   // main, isMain=true
    withContext(Dispatchers.Default) {
        logDispatcher("Default") // DefaultDispatcher-worker-2, isMain=false
        withContext(Dispatchers.IO) {
            logDispatcher("IO in Default") // 같은 worker일 수 있음!
        }
    }
}
```

### 테스트에서 Dispatcher 교체

```kotlin
// 테스트에서는 Main Dispatcher가 없으므로 교체 필요
class MyViewModelTest {
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    @Test
    fun testLoadData() = runTest {
        // Dispatchers.Main이 TestDispatcher로 교체됨
        // → Handler/Looper 없이 동작
        val viewModel = MyViewModel()
        viewModel.loadData()
        assertEquals("expected", viewModel.uiState.value)
    }
}

class MainDispatcherRule(
    val testDispatcher: TestDispatcher = UnconfinedTestDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(testDispatcher)
    }
    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```

## 🔗 관련 개념

- [[02-Coroutines/기초/Dispatcher|Dispatcher]]
- [[02-Coroutines/기초/CoroutineContext|CoroutineContext]]
- [[02-Coroutines/내부-동작/Handler-Coroutine-브릿지|Handler-Coroutine 브릿지]]
- [[02-Coroutines/내부-동작/ContinuationPassing|Continuation Passing]]
- [[00-CS-기초/동시성/스레드-풀|스레드 풀]]

## 📚 더 보기

- [kotlinx-coroutines-android 소스](https://github.com/Kotlin/kotlinx.coroutines/tree/master/ui/kotlinx-coroutines-android)
- [CoroutineScheduler 소스](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/jvm/src/scheduling/CoroutineScheduler.kt)
- [Android Looper 문서](https://developer.android.com/reference/android/os/Looper)

---

**핵심 요약:** Dispatchers.Main은 Handler.post()로 메인 스레드 MessageQueue에 코루틴 블록을 넣는 구조. Default와 IO는 같은 스레드 풀을 공유하되 동시 실행 한도만 다르다. runBlocking + Dispatchers.Main = 데드락.
