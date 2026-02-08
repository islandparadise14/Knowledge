# Memory Leak 패턴들

> 자주 발생하는 메모리 누수 패턴과 해결 방법

## 1. Static 참조

### 문제
```kotlin
companion object {
    var context: Context? = null  // Activity Context 누수!
}
```

### 해결
```kotlin
companion object {
    var context: Context? = null
        set(value) {
            field = value?.applicationContext  // Application Context 사용
        }
}
```

## 2. Inner Class / Anonymous Class

### 문제
```kotlin
class MyActivity : AppCompatActivity() {
    private val handler = object : Handler(Looper.getMainLooper()) {
        override fun handleMessage(msg: Message) {
            // 암묵적으로 MyActivity 참조
        }
    }
}
```

### 해결
```kotlin
class MyActivity : AppCompatActivity() {
    private val handler = MyHandler(this)

    private class MyHandler(activity: MyActivity) : Handler(Looper.getMainLooper()) {
        private val activityRef = WeakReference(activity)

        override fun handleMessage(msg: Message) {
            activityRef.get()?.let { activity ->
                // 안전하게 사용
            }
        }
    }
}
```

## 3. Listener / Callback 미해제

### 문제
```kotlin
override fun onCreate() {
    LocationManager.registerListener(this)  // Activity가 리스너
}
// onDestroy에서 해제 안 함
```

### 해결
```kotlin
override fun onDestroy() {
    super.onDestroy()
    LocationManager.unregisterListener(this)
}
```

## 4. Coroutine Scope 미취소

### 문제
```kotlin
class MyActivity : AppCompatActivity() {
    private val scope = CoroutineScope(Dispatchers.Main)

    fun loadData() {
        scope.launch {
            // Activity 종료 후에도 계속 실행
        }
    }
}
```

### 해결
```kotlin
class MyActivity : AppCompatActivity() {
    fun loadData() {
        lifecycleScope.launch {
            // Activity 종료 시 자동 취소
        }
    }
}
```

## 5. ViewModel에서 View/Context 참조

### 문제
```kotlin
class MyViewModel(
    private val context: Context  // Activity Context
) : ViewModel()
```

### 해결
```kotlin
class MyViewModel(
    private val application: Application  // Application Context
) : AndroidViewModel(application)
```

## 누수 탐지 도구

- **LeakCanary**: 자동 누수 감지
- **Android Profiler**: 메모리 분석
- **MAT**: 힙 덤프 분석

---

## 관련 문서
- [[03-Android-Framework/메모리/Memory-Leak|Memory Leak]]
- [[03-Android-Framework/Context/Context란|Context란]]

[← Android Framework](03-Android-Framework/README.md)