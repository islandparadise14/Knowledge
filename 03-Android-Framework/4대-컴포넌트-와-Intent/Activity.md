---
tags: [android, activity, lifecycle, 4대컴포넌트]
---

# Activity

## 💡 핵심 개념

사용자와 상호작용하는 단일 화면 컴포넌트. 앱의 진입점이며 생명주기를 가진다. 모든 Activity는 AndroidManifest.xml에 등록해야 한다.

## 📌 왜 필요한가?

Android 시스템이 Activity의 생명주기를 직접 관리한다. 전화가 오거나 앱이 백그라운드로 가는 등의 상황에서 시스템이 Activity를 제어하므로, 각 생명주기 콜백에서 적절한 처리를 해야 메모리 누수와 데이터 손실을 막을 수 있다.

## 🔍 자세히

### 기본 구조

```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }
}
```

### 생명주기

```
                    Activity 시작
                         ↓
                     onCreate()      ← 초기화, View 생성
                         ↓
                     onStart()       ← 화면에 보이기 시작
                         ↓
                     onResume()      ← 포커스 획득, 상호작용 가능
                         ↓
                   [사용자와 상호작용]
                         ↓
                     onPause()       ← 포커스 잃음 (다른 Activity 진입)
                         ↓
                     onStop()        ← 화면에서 완전히 안 보임
                         ↓
                    onRestart()      ← 다시 돌아올 때 (onStop → onStart)
                         또는
                    onDestroy()      ← 완전 종료
```

```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // View 초기화, ViewModel 연결, 데이터 복원
    }

    override fun onStart() {
        super.onStart()
        // 화면이 보이기 시작 — 위치 추적 등록 등
    }

    override fun onResume() {
        super.onResume()
        // 포커스 획득 — 카메라, 센서 시작
    }

    override fun onPause() {
        super.onPause()
        // 포커스 잃음 — 카메라, 센서 중지
        // 여기서 너무 무거운 작업 금지 (다음 Activity 진입 지연됨)
    }

    override fun onStop() {
        super.onStop()
        // 화면에서 안 보임 — 위치 추적 해제 등
    }

    override fun onDestroy() {
        super.onDestroy()
        // 완전 종료 — 리소스 해제
    }
}
```

### savedInstanceState — 상태 복원

시스템이 Activity를 종료했다가 재생성할 때(화면 회전 등) 상태를 보존한다.

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)

    // 이전 상태 복원
    val count = savedInstanceState?.getInt("count") ?: 0
}

override fun onSaveInstanceState(outState: Bundle) {
    super.onSaveInstanceState(outState)
    outState.putInt("count", currentCount)
}
```

ViewModel을 쓰면 화면 회전 시 자동 보존되므로 `onSaveInstanceState`를 직접 쓸 일이 줄어든다.

### Manifest 등록

```xml
<activity
    android:name=".MainActivity"
    android:exported="true"
    android:launchMode="standard">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

### Launch Mode

| Mode | 동작 | 사용 사례 |
|------|------|----------|
| `standard` | 항상 새 인스턴스 생성 | 기본값 |
| `singleTop` | 스택 최상단에 있으면 재사용 (`onNewIntent` 호출) | 알림에서 열리는 화면 |
| `singleTask` | 태스크 내 하나만 존재, 위에 쌓인 것들 제거 | 메인 화면 |
| `singleInstance` | 독립된 태스크에 하나만 존재 | 런처, 브라우저 |

```kotlin
// singleTop, singleTask에서 재사용될 때 호출
override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    setIntent(intent)
    // 새 Intent 처리
}
```

### Task와 Back Stack

```
[Task A]
  ┌─────────────┐
  │  ActivityC  │  ← 최상단 (현재 화면)
  │  ActivityB  │
  │  ActivityA  │  ← 맨 아래
  └─────────────┘

뒤로 가기 → ActivityC pop → ActivityB가 최상단
```

### Intent로 시작

```kotlin
// 명시적 Intent
val intent = Intent(this, SecondActivity::class.java).apply {
    putExtra("userId", "123")
}
startActivity(intent)

// 암시적 Intent
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://example.com"))
startActivity(intent)
```

### 결과 받기 (Activity Result API)

```kotlin
private val launcher = registerForActivityResult(
    ActivityResultContracts.StartActivityForResult()
) { result ->
    if (result.resultCode == RESULT_OK) {
        val data = result.data?.getStringExtra("result")
    }
}

// 실행
launcher.launch(Intent(this, SecondActivity::class.java))

// SecondActivity에서 결과 반환
setResult(RESULT_OK, Intent().apply { putExtra("result", "값") })
finish()
```

### 권한 요청

```kotlin
private val requestPermission = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { isGranted ->
    if (isGranted) {
        // 권한 허용
    } else {
        // 권한 거부
    }
}

requestPermission.launch(Manifest.permission.CAMERA)
```

## 💻 실전 패턴

### ViewModel + StateFlow 기본 구조

```kotlin
class MainActivity : AppCompatActivity() {
    private val viewModel: MainViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    render(state)
                }
            }
        }
    }
}
```

## 🔗 관련 개념

- [[03-Android-Framework/생명주기/Activity-Lifecycle]]
- [[03-Android-Framework/생명주기/Configuration-Changes]]
- [[03-Android-Framework/4대-컴포넌트-와-Intent/Service]]
- [[03-Android-Framework/4대-컴포넌트-와-Intent/BroadcastReceiver]]
- [[03-Android-Framework/4대-컴포넌트-와-Intent/ContentProvider]]

## 📚 더 보기

- [Android Docs - Activity](https://developer.android.com/guide/components/activities/intro-activities)

---

**핵심 요약:** 생명주기 콜백에서 리소스 관리. 화면 회전 시 ViewModel로 상태 보존. LaunchMode로 Back Stack 제어.
