---
tags: [android, lifecycle, activity]
---

# Activity Lifecycle

## 💡 핵심 개념

Activity는 **6가지 생명주기 콜백**을 거친다: onCreate → onStart → onResume → onPause → onStop → onDestroy. 화면 회전, 백그라운드 전환 등에 따라 호출된다.

## 📌 왜 필요한가?

리소스 관리, 상태 저장, 메모리 누수 방지에 필수다. 생명주기를 이해하지 못하면 크래시, 데이터 손실, 배터리 낭비가 발생한다.

## 🔍 자세히

### 생명주기 흐름

```
        ┌──> onCreate() ──> onStart() ──> onResume() [RESUMED]
        │                                      │
    [CREATED]                                  │
        │                                  onPause()
        │                                      │
        │    onRestart() <── onStop() <────────┘
        │        │             │
        │        └─────────────┘
        │
    onDestroy()
        │
    [DESTROYED]
```

### 주요 콜백

**onCreate**

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    
    // 초기화: View, ViewModel, Listener 등
    viewModel = ViewModelProvider(this).get(MyViewModel::class.java)
    setupViews()
}
```

**용도:** 
- UI 초기화
- ViewModel 생성
- Intent 데이터 읽기
- 한 번만 수행할 작업

**onStart**

```kotlin
override fun onStart() {
    super.onStart()
    // 화면에 보이기 시작
    // 애니메이션 시작, BroadcastReceiver 등록
}
```

**onResume**

```kotlin
override fun onResume() {
    super.onResume()
    // 포그라운드, 사용자 상호작용 가능
    // 카메라, 센서 시작
    cameraManager.openCamera()
}
```

**onPause**

```kotlin
override fun onPause() {
    // 리소스 해제
    cameraManager.closeCamera()
    super.onPause()
}
```

**주의:** 빠르게 실행돼야 함. 다음 Activity 전환이 블록됨.

**onStop**

```kotlin
override fun onStop() {
    // 화면에서 완전히 사라짐
    // DB 저장, 네트워크 연결 해제
    super.onStop()
}
```

**onDestroy**

```kotlin
override fun onDestroy() {
    // 최종 정리
    disposables.clear()
    super.onDestroy()
}
```

**주의:** 항상 호출되는 것은 아님 (Process kill)

### 화면 회전

```kotlin
// 화면 회전 시
onPause() → onStop() → onDestroy() → onCreate() → onStart() → onResume()

// 상태 저장
override fun onSaveInstanceState(outState: Bundle) {
    super.onSaveInstanceState(outState)
    outState.putString("key", "value")
}

// 상태 복원
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    val value = savedInstanceState?.getString("key")
}
```

### 백그라운드 전환

```kotlin
// Home 버튼 누름
onPause() → onStop()

// 다시 돌아옴
onRestart() → onStart() → onResume()

// 다른 Activity 시작
onPause() → [새 Activity onCreate/onStart/onResume] → onStop()
```

### 동작 원리

**ActivityThread가 관리**

```kotlin
// 대략적인 내부 동작
class ActivityThread {
    fun handleLaunchActivity() {
        val activity = performLaunchActivity()
        handleResumeActivity(activity)
    }
    
    fun performLaunchActivity(): Activity {
        val activity = instantiate(...)
        activity.attach(...)
        activity.onCreate(...)
        return activity
    }
}
```

**Task와 Back Stack**

```
Task Stack:
┌─────────────┐
│ Activity C  │ ← onResume (Top)
├─────────────┤
│ Activity B  │ ← onStop
├─────────────┤
│ Activity A  │ ← onStop
└─────────────┘

Back 버튼:
C.onPause → C.onStop → C.onDestroy
B.onRestart → B.onStart → B.onResume
```

### 주의사항

**1. super 호출 순서**

```kotlin
// onCreate, onStart, onResume - super 먼저
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)  // 먼저!
    // 초기화 코드
}

// onPause, onStop, onDestroy - super 나중에
override fun onPause() {
    // 정리 코드
    super.onPause()  // 나중에!
}
```

**2. onDestroy는 보장 안 됨**

```kotlin
// 나쁨 - onDestroy에서만 정리
override fun onDestroy() {
    database.close()  // Process kill되면 안 호출됨!
    super.onDestroy()
}

// 좋음 - onStop에서도 정리
override fun onStop() {
    database.close()
    super.onStop()
}
```

**3. 생명주기 인식 컴포넌트 사용**

```kotlin
// 나쁨 - 수동 관리
override fun onStart() {
    super.onStart()
    locationManager.startUpdates()
}

override fun onStop() {
    locationManager.stopUpdates()
    super.onStop()
}

// 좋음 - Lifecycle 자동 관리
class LocationObserver : DefaultLifecycleObserver {
    override fun onStart(owner: LifecycleOwner) {
        locationManager.startUpdates()
    }
    
    override fun onStop(owner: LifecycleOwner) {
        locationManager.stopUpdates()
    }
}

lifecycle.addObserver(LocationObserver())
```

## 💻 실전 예제

### ViewModel 사용

```kotlin
class MainActivity : AppCompatActivity() {
    private val viewModel: MyViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // ViewModel은 화면 회전에도 유지됨
        viewModel.data.observe(this) { data ->
            updateUI(data)
        }
    }
}
```

### Lifecycle-aware 컴포넌트

```kotlin
class MyObserver : DefaultLifecycleObserver {
    override fun onCreate(owner: LifecycleOwner) {
        Log.d("MyObserver", "onCreate")
    }
    
    override fun onStart(owner: LifecycleOwner) {
        // 리소스 시작
    }
    
    override fun onStop(owner: LifecycleOwner) {
        // 리소스 정지
    }
}

// Activity에서
lifecycle.addObserver(MyObserver())
```

### 코루틴과 생명주기

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // lifecycleScope - Activity 종료 시 자동 취소
        lifecycleScope.launch {
            viewModel.data.collect { data ->
                updateUI(data)
            }
        }
        
        // repeatOnLifecycle - STARTED 상태에서만 수집
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    // onStart~onStop 사이에만 수집
                }
            }
        }
    }
}
```

### 상태 저장 및 복원

```kotlin
class MainActivity : AppCompatActivity() {
    private var counter = 0
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // 복원
        counter = savedInstanceState?.getInt("counter") ?: 0
    }
    
    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        // 저장
        outState.putInt("counter", counter)
    }
}

// ViewModel 사용하면 더 간단
class MyViewModel : ViewModel() {
    var counter = 0  // 자동으로 유지됨
}
```

## 🔗 관련 개념

- [[03-Android-Framework/생명주기/Fragment-Lifecycle]]
- [[03-Android-Framework/생명주기/Process-Death]]
- [[03-Android-Framework/생명주기/ViewModel과-생명주기]]

## 📚 더 보기

- [Android Developers - Activity Lifecycle](https://developer.android.com/guide/components/activities/activity-lifecycle)

---

**핵심 요약:** onCreate(초기화) → onStart(보임) → onResume(상호작용) → onPause(일시정지) → onStop(안 보임) → onDestroy(종료). super 호출 순서 중요.
