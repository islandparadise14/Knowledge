# Configuration Changes

> 화면 회전, 언어 변경 등 구성 변경 시 앱 동작

## Configuration Change란?

기기 구성이 변경되어 Activity가 재생성되는 상황:
- 화면 회전
- 언어 변경
- 키보드 연결/해제
- 다크 모드 전환
- 화면 크기 변경 (폴더블)

## 기본 동작

```
onPause() → onStop() → onDestroy() → onCreate() → onStart() → onResume()
```

Activity가 완전히 재생성됨.

## 상태 보존 방법

### 1. onSaveInstanceState
```kotlin
override fun onSaveInstanceState(outState: Bundle) {
    super.onSaveInstanceState(outState)
    outState.putString("key", value)
}

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    savedInstanceState?.getString("key")?.let { value = it }
}
```

### 2. ViewModel (권장)
```kotlin
class MyViewModel : ViewModel() {
    var data: String = ""  // Configuration Change에도 유지
}
```

### 3. SavedStateHandle
```kotlin
class MyViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    val data = savedStateHandle.getLiveData<String>("key")
}
```

## Configuration Change 직접 처리

AndroidManifest.xml:
```xml
<activity
    android:name=".MyActivity"
    android:configChanges="orientation|screenSize|keyboardHidden">
```

Activity:
```kotlin
override fun onConfigurationChanged(newConfig: Configuration) {
    super.onConfigurationChanged(newConfig)
    // Activity 재생성 없이 직접 처리
    if (newConfig.orientation == Configuration.ORIENTATION_LANDSCAPE) {
        // 가로 모드 처리
    }
}
```

### 주의사항
- 리소스 자동 갱신 안 됨 (다국어 등)
- 직접 모든 변경 처리해야 함
- 일반적으로 권장하지 않음

## Compose에서

```kotlin
@Composable
fun MyScreen(viewModel: MyViewModel = viewModel()) {
    // ViewModel은 Configuration Change에도 유지
    val data by viewModel.data.collectAsState()
}
```

---

## 관련 문서
- [[03-Android-Framework/생명주기/Activity-Lifecycle|Activity Lifecycle]]
- [[06-Architecture/ViewModel/ViewModel-기초|ViewModel 기초]]

[← Android Framework](03-Android-Framework/README.md)
