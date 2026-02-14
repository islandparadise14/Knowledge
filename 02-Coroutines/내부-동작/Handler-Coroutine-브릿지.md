---
tags: [coroutines, handler, looper, bridge, suspendCancellableCoroutine, 내부동작]
---

# Handler-Coroutine 브릿지

## 💡 핵심 개념

Android의 전통적 비동기 패턴(Handler, Callback, Listener)과 코루틴을 연결하는 기법이다. 핵심은 `suspendCancellableCoroutine`으로 콜백 기반 API를 suspend 함수로 감싸는 것이며, 반대로 코루틴 결과를 Handler로 전달할 수도 있다.

## 📌 왜 필요한가?

기존 Android API 중 상당수가 콜백 기반(LocationManager, SensorManager, Camera, MediaPlayer 등)이다. 이들을 코루틴과 자연스럽게 연결하는 패턴을 알아야 콜백 지옥 없이 깔끔한 코드를 작성할 수 있다.

## 🔍 자세히

### Handler → Coroutine: suspendCancellableCoroutine

콜백 기반 API를 suspend 함수로 변환하는 핵심 도구.

```kotlin
// 콜백 기반 API를 suspend 함수로 감싸기
suspend fun View.awaitNextLayout(): Unit = suspendCancellableCoroutine { cont ->
    val listener = object : View.OnLayoutChangeListener {
        override fun onLayoutChange(
            v: View, l: Int, t: Int, r: Int, b: Int,
            ol: Int, ot: Int, or2: Int, ob: Int
        ) {
            // 레이아웃 완료 → 코루틴 재개
            removeOnLayoutChangeListener(this)
            cont.resume(Unit)
        }
    }
    addOnLayoutChangeListener(listener)

    // 코루틴 취소 시 리스너 해제
    cont.invokeOnCancellation {
        removeOnLayoutChangeListener(listener)
    }
}

// 사용
lifecycleScope.launch {
    textView.text = "New text"
    textView.awaitNextLayout()  // 레이아웃 완료까지 대기
    // 여기서는 textView의 새 크기가 확정됨
    val width = textView.width
}
```

### Handler.postDelayed → delay

```kotlin
// ❌ 전통적 방식: Handler + Runnable
val handler = Handler(Looper.getMainLooper())
handler.postDelayed({
    textView.text = "Delayed"
}, 3000)
// 취소하려면 Runnable 참조를 보관해야 함

// ✅ 코루틴 방식: delay
lifecycleScope.launch {
    delay(3000)
    textView.text = "Delayed"
}
// lifecycleScope가 자동으로 취소 관리
```

### 콜백 API를 suspend로 변환하는 패턴들

**1. 단발성 콜백 (One-shot)**

```kotlin
// Animation 완료 대기
suspend fun Animator.awaitEnd(): Unit = suspendCancellableCoroutine { cont ->
    addListener(object : AnimatorListenerAdapter() {
        override fun onAnimationEnd(animation: Animator) {
            removeListener(this)
            cont.resume(Unit)
        }
        override fun onAnimationCancel(animation: Animator) {
            removeListener(this)
            cont.resume(Unit)
        }
    })

    cont.invokeOnCancellation {
        cancel()  // 코루틴 취소 시 애니메이션도 취소
    }
}

// 사용: 애니메이션 순차 실행
lifecycleScope.launch {
    fadeInAnimator.start()
    fadeInAnimator.awaitEnd()        // 페이드인 완료 대기
    slideUpAnimator.start()
    slideUpAnimator.awaitEnd()       // 슬라이드 완료 대기
    textView.text = "Animation done"
}
```

**2. Dialog 결과 대기**

```kotlin
suspend fun Context.showConfirmDialog(message: String): Boolean =
    suspendCancellableCoroutine { cont ->
        val dialog = AlertDialog.Builder(this)
            .setMessage(message)
            .setPositiveButton("확인") { _, _ -> cont.resume(true) }
            .setNegativeButton("취소") { _, _ -> cont.resume(false) }
            .setOnCancelListener { cont.resume(false) }
            .create()

        cont.invokeOnCancellation { dialog.dismiss() }
        dialog.show()
    }

// 사용
lifecycleScope.launch {
    val confirmed = showConfirmDialog("삭제하시겠습니까?")
    if (confirmed) {
        repository.delete(item)
    }
}
```

**3. 연속 콜백 → callbackFlow**

```kotlin
// 연속적으로 값을 내보내는 콜백 → Flow로 변환
fun LocationManager.locationFlow(
    provider: String,
    minTimeMs: Long = 1000L
): Flow<Location> = callbackFlow {
    val listener = LocationListener { location ->
        trySend(location)  // Flow로 값 전달
    }

    requestLocationUpdates(provider, minTimeMs, 0f, listener)

    awaitClose {
        // Flow 수집 종료 시 리스너 해제
        removeUpdates(listener)
    }
}

// 사용
lifecycleScope.launch {
    locationManager.locationFlow(LocationManager.GPS_PROVIDER)
        .collect { location ->
            updateMap(location.latitude, location.longitude)
        }
}
```

### Coroutine → Handler: 결과 전달

```kotlin
// 코루틴 결과를 Handler 기반 API에 전달해야 할 때
class MyHandlerThread : HandlerThread("worker") {
    lateinit var workerHandler: Handler

    override fun onLooperPrepared() {
        workerHandler = Handler(looper)
    }
}

// 코루틴에서 HandlerThread로 작업 전달
suspend fun postToHandlerThread(handler: Handler, block: () -> Unit) {
    suspendCancellableCoroutine<Unit> { cont ->
        handler.post {
            block()
            cont.resume(Unit)
        }
        cont.invokeOnCancellation {
            handler.removeCallbacksAndMessages(null)
        }
    }
}
```

### Handler + Coroutine 혼용 시 주의사항

```kotlin
// ❌ 위험: Handler 콜백 안에서 코루틴 스코프 문제
handler.post {
    // 여기는 코루틴 스코프가 아님
    // launch { } 사용 불가 (스코프 없음)

    // GlobalScope.launch는 생명주기 관리 안 됨
    GlobalScope.launch { /* 위험 */ }
}

// ✅ 안전: CoroutineScope을 명시적으로 사용
class MyService : Service() {
    private val scope = CoroutineScope(Dispatchers.Main + SupervisorJob())
    private val handler = Handler(Looper.getMainLooper())

    fun doWork() {
        handler.postDelayed({
            scope.launch {
                val result = withContext(Dispatchers.IO) { fetchData() }
                updateNotification(result)
            }
        }, 5000)
    }

    override fun onDestroy() {
        scope.cancel()  // 코루틴 정리
        handler.removeCallbacksAndMessages(null)  // Handler 정리
        super.onDestroy()
    }
}

// ✅ 더 나은 방법: Handler 없이 코루틴만 사용
class MyService : Service() {
    private val scope = CoroutineScope(Dispatchers.Main + SupervisorJob())

    fun doWork() {
        scope.launch {
            delay(5000)  // Handler.postDelayed 대체
            val result = withContext(Dispatchers.IO) { fetchData() }
            updateNotification(result)
        }
    }
}
```

### HandlerThread를 Dispatcher로 변환

```kotlin
// HandlerThread를 코루틴 Dispatcher로 사용
val handlerThread = HandlerThread("background").apply { start() }
val handler = Handler(handlerThread.looper)
val dispatcher = handler.asCoroutineDispatcher()

// 이제 해당 HandlerThread에서 코루틴 실행 가능
scope.launch(dispatcher) {
    // handlerThread에서 실행
    // 순차 실행 보장 (단일 스레드)
}

// 정리
dispatcher.close()
handlerThread.quitSafely()
```

### 마이그레이션 가이드

```
전통적 패턴                     코루틴 대체
─────────────────────────────────────────────
Handler.post { }              → launch(Dispatchers.Main) { }
Handler.postDelayed(r, ms)    → launch { delay(ms); ... }
AsyncTask.execute()           → launch(Dispatchers.IO) { }
Thread { }.start()            → launch(Dispatchers.Default) { }
Callback<T>                   → suspendCancellableCoroutine
Listener (연속)                → callbackFlow { }
Timer/TimerTask               → launch { while(isActive) { delay() } }
HandlerThread                 → handler.asCoroutineDispatcher()
CountDownLatch                → Job.join() 또는 CompletableDeferred
```

## 🔗 관련 개념

- [[02-Coroutines/내부-동작/Dispatchers-내부-구현|Dispatchers 내부 구현]]
- [[02-Coroutines/내부-동작/ContinuationPassing|Continuation Passing]]
- [[02-Coroutines/기초/Dispatcher|Dispatcher]]
- [[02-Coroutines/Flow/Flow-기초|Flow 기초]]
- [[00-CS-기초/동시성/동기-vs-비동기|동기 vs 비동기]]

## 📚 더 보기

- [suspendCancellableCoroutine 문서](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/suspend-cancellable-coroutine.html)
- [callbackFlow 문서](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/callback-flow.html)
- [Android KTX - Animation 확장](https://developer.android.com/kotlin/ktx#animator)

---

**핵심 요약:** `suspendCancellableCoroutine`으로 단발성 콜백을, `callbackFlow`로 연속 콜백을 코루틴/Flow로 변환한다. Handler 기반 코드는 대부분 코루틴으로 대체 가능하며, `handler.asCoroutineDispatcher()`로 HandlerThread도 Dispatcher로 쓸 수 있다.
