---
tags: [CS, 동시성, 프로세스, 스레드, Android]
---

# 프로세스 vs 스레드

## 💡 핵심 개념

**프로세스(Process)**는 운영체제로부터 독립된 메모리 공간을 할당받아 실행되는 프로그램 단위이다. **스레드(Thread)**는 프로세스 내에서 실행되는 흐름의 단위로, 같은 프로세스의 메모리를 공유한다. Android 앱은 기본적으로 하나의 프로세스에서 메인 스레드(UI 스레드)와 여러 백그라운드 스레드로 동작한다.

## 📌 왜 필요한가?

Android에서 네트워크 요청이나 DB 접근 같은 무거운 작업을 메인 스레드에서 수행하면 ANR(Application Not Responding)이 발생한다. 프로세스와 스레드의 차이를 이해해야 올바른 동시성 전략을 설계할 수 있다.

## 🔍 자세히

### 프로세스와 스레드 비교

| 구분 | 프로세스 | 스레드 |
|------|----------|--------|
| 메모리 | 독립된 주소 공간 | 프로세스 내 힙/코드 영역 공유 |
| 생성 비용 | 높음 (fork) | 낮음 |
| 통신 방식 | IPC (Binder, Intent 등) | 공유 메모리 직접 접근 |
| 안정성 | 하나 죽어도 다른 프로세스 무관 | 하나가 크래시하면 프로세스 전체 종료 |
| 컨텍스트 스위칭 | 비용 높음 | 비용 낮음 |

### Android의 프로세스 모델

Android에서 각 앱은 기본적으로 별도의 Linux 프로세스에서 실행된다. `AndroidManifest.xml`에서 `android:process` 속성을 통해 멀티 프로세스를 구성할 수 있다.

```xml
<!-- 별도 프로세스에서 실행되는 Service -->
<service
    android:name=".PushService"
    android:process=":push" />
```

멀티 프로세스 사용 시 주의사항:
- `SharedPreferences`, 싱글톤 객체 등은 프로세스마다 별도 인스턴스가 생성된다
- 프로세스 간 통신은 `Binder`, `ContentProvider`, `BroadcastReceiver` 등을 사용해야 한다
- 메모리 사용량이 프로세스 수만큼 증가한다

### Android의 스레드 모델

```kotlin
// 메인 스레드 (UI 스레드) - 앱 시작 시 자동 생성
// UI 업데이트는 반드시 메인 스레드에서 수행해야 함

class MyActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // ❌ 잘못된 예: 메인 스레드에서 네트워크 호출
        // val result = fetchDataFromNetwork() // ANR 발생!

        // ✅ 코루틴으로 백그라운드 작업
        lifecycleScope.launch {
            val data = withContext(Dispatchers.IO) {
                fetchDataFromNetwork() // IO 스레드에서 실행
            }
            updateUI(data) // 메인 스레드에서 UI 업데이트
        }
    }
}
```

### Handler/Looper 메커니즘

Android 메인 스레드는 내부적으로 `Looper`와 `MessageQueue`를 갖고 있다.

```kotlin
// Handler를 통한 스레드 간 통신 (전통적 방식)
val handler = Handler(Looper.getMainLooper())

Thread {
    val result = heavyComputation()
    handler.post {
        // 메인 스레드에서 실행
        textView.text = result
    }
}.start()

// 현대적 방식: 코루틴 사용
lifecycleScope.launch(Dispatchers.Main) {
    val result = withContext(Dispatchers.Default) {
        heavyComputation()
    }
    textView.text = result
}
```

### 스레드 안전성 (Thread Safety)

공유 메모리를 사용하는 스레드 환경에서는 동기화가 필수적이다.

```kotlin
// ❌ 스레드 안전하지 않은 코드
var count = 0
repeat(1000) {
    thread { count++ }
}
// count가 1000이 아닐 수 있음

// ✅ 코루틴에서의 안전한 방법
val mutex = Mutex()
var safeCount = 0

coroutineScope {
    repeat(1000) {
        launch(Dispatchers.Default) {
            mutex.withLock { safeCount++ }
        }
    }
}
// safeCount는 정확히 1000
```

## 🔗 관련 개념

- [[00-CS-기초/동시성/동시성-vs-병렬성|동시성 vs 병렬성]]
- [[00-CS-기초/동시성/스레드-풀|스레드 풀]]
- [[00-CS-기초/동시성/교착상태-데드락|교착상태 (데드락)]]
- [[00-CS-기초/동시성/경쟁-조건|경쟁 조건]]
- [[00-CS-기초/운영체제/프로세스-관리|프로세스 관리]]
- [[00-CS-기초/운영체제/IPC-프로세스간-통신|IPC 프로세스간 통신]]

## 📚 더 보기

- [Android Processes and Threads](https://developer.android.com/guide/components/processes-and-threads)
- [Kotlin Coroutines Guide](https://kotlinlang.org/docs/coroutines-guide.html)

---

**핵심 요약:** 프로세스는 독립된 메모리를 가진 실행 단위이고, 스레드는 프로세스 내에서 메모리를 공유하는 실행 흐름으로, Android에서는 메인 스레드 블로킹을 피하기 위해 코루틴 기반의 백그라운드 스레드 활용이 필수적이다.