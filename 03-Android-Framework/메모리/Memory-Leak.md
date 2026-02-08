---
tags: [android, memory-leak]
---

# Memory Leak

## 💡 핵심 개념

**Memory Leak**은 더 이상 필요 없는 객체가 GC되지 않고 메모리에 남아있는 것이다. Context, Listener, Handler, static 참조가 주요 원인이다.

## 🔍 자세히

### 주요 원인

**1. Context 누수**
```kotlin
// 나쁨
companion object {
    var context: Context? = null  // Activity 참조 유지
}

// 좋음
companion object {
    var appContext: Context? = null  // Application Context 사용
}
```

**2. Listener 미해제**
```kotlin
// 나쁨
override fun onStart() {
    eventBus.register(this)
    // onStop에서 unregister 안 함
}

// 좋음
override fun onStop() {
    eventBus.unregister(this)
    super.onStop()
}
```

**3. Handler 참조**
```kotlin
// 나쁨
class MyActivity : AppCompatActivity() {
    val handler = Handler(Looper.getMainLooper()) {
        // Activity 참조
    }
}

// 좋음
class MyHandler(activity: MyActivity) : Handler(Looper.getMainLooper()) {
    private val activityRef = WeakReference(activity)
}
```

**4. Inner Class**
```kotlin
// 나쁨 - 외부 클래스 참조
inner class MyTask {
    fun doWork() {
        // Activity 참조 유지
    }
}

// 좋음 - static (companion)
companion object {
    class MyTask(activityRef: WeakReference<MyActivity>) {
        fun doWork() {
            activityRef.get()?.let { }
        }
    }
}
```

### 감지 도구

**LeakCanary**
```kotlin
// build.gradle
debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.12'

// 자동으로 누수 감지 및 알림
```

**Android Profiler**
- Memory Profiler → Heap Dump
- Leak 의심 객체 확인

## 💻 실전 예제

### ViewModel 사용
```kotlin
// Context 직접 참조 금지
class MyViewModel(private val context: Context) : ViewModel() {  // 나쁨
}

// Application Context 사용
class MyViewModel(
    private val app: Application
) : ViewModel() {  // 좋음
}
```

### Coroutine 정리
```kotlin
class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // lifecycleScope - 자동 취소
        lifecycleScope.launch {
            // Activity 종료 시 자동 취소
        }
    }
}
```

## 🔗 관련 개념

- [[03-Android-Framework/Context/Context란]]
- [[03-Android-Framework/메모리/WeakReference]]

---

**핵심 요약:** Context, Listener, Handler 참조 조심. LeakCanary로 감지. ViewModel, lifecycleScope 활용.
