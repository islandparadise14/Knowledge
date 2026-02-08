---
tags: [android, process-death, lifecycle, state-restoration]
---

# Process Death

## 💡 핵심 개념

**Process Death**는 시스템이 메모리 부족 시 백그라운드 프로세스를 강제 종료하는 것이다. onDestroy() 없이 바로 kill된다.

## 📌 왜 필요한가?

개발자 대부분이 놓치는 케이스다. 테스트하지 않으면 사용자에게만 발생하는 버그가 생긴다. 상태를 제대로 저장/복원하지 않으면 데이터 손실, 크래시가 발생한다.

## 🔍 자세히

### Process Death 발생 시나리오

```kotlin
1. 앱 실행 (백그라운드로 전환)
2. 시스템이 메모리 확보 위해 프로세스 kill
3. 사용자가 앱으로 돌아옴
4. 시스템이 프로세스 재생성
5. Activity/Fragment 복원 시도
   → onSaveInstanceState()로 저장한 상태만 복원
```

### 정상 종료 vs Process Death

```kotlin
// 정상 종료
onPause() → onStop() → onDestroy()

// Process Death
onPause() → onStop() → [PROCESS KILLED]

// 복원
onCreate() → onStart() → onResume()
```

### 상태 저장

```kotlin
// Activity
override fun onSaveInstanceState(outState: Bundle) {
    super.onSaveInstanceState(outState)
    outState.putString("key", value)
    outState.putParcelable("user", user)
}

// Fragment
override fun onSaveInstanceState(outState: Bundle) {
    super.onSaveInstanceState(outState)
    // Fragment 상태 저장
}
```

### ViewModel과 SavedStateHandle

```kotlin
class MyViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    // 자동으로 저장/복원
    var query: String
        get() = savedStateHandle.get<String>("query") ?: ""
        set(value) {
            savedStateHandle["query"] = value
        }
    
    // StateFlow로
    val queryFlow: StateFlow<String> = savedStateHandle
        .getStateFlow("query", "")
}
```

### 테스트 방법

**개발자 옵션에서**
1. Settings → Developer options
2. Apps → Don't keep activities (활성화)
3. 앱 실행 → 백그라운드 → 포그라운드
4. Activity가 재생성됨

**adb 명령**
```bash
# 프로세스 kill
adb shell am kill [package_name]

# 백그라운드로 전환
adb shell input keyevent KEYCODE_HOME

# 다시 실행
adb shell am start -n [package_name]/[activity_name]
```

### 주의사항

**1. ViewModel은 Process Death에서 소멸**

```kotlin
// 나쁨 - Process Death 시 소멸
class MyViewModel : ViewModel() {
    var data: String = ""  // 소멸됨!
}

// 좋음 - SavedStateHandle 사용
class MyViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    var data: String
        get() = savedStateHandle["data"] ?: ""
        set(value) { savedStateHandle["data"] = value }
}
```

**2. Bundle 크기 제한**

```kotlin
// Bundle은 1MB 제한
// 큰 데이터는 저장하지 말 것
outState.putParcelable("largeData", data)  // 위험!

// ID만 저장하고 DB에서 다시 로드
outState.putString("dataId", data.id)
```

**3. Parcelable vs Serializable**

```kotlin
// Parcelable (빠름, 권장)
@Parcelize
data class User(val name: String, val age: Int) : Parcelable

// Serializable (느림)
data class User(val name: String, val age: Int) : Serializable
```

## 💻 실전 예제

### 올바른 상태 관리

```kotlin
class MyViewModel(
    private val savedStateHandle: SavedStateHandle,
    private val repository: UserRepository
) : ViewModel() {
    // Process Death에서도 유지
    val userId: String = savedStateHandle["userId"] ?: ""
    
    private val _user = MutableStateFlow<User?>(null)
    val user: StateFlow<User?> = _user.asStateFlow()
    
    init {
        if (userId.isNotEmpty()) {
            loadUser(userId)
        }
    }
    
    fun loadUser(id: String) {
        savedStateHandle["userId"] = id
        viewModelScope.launch {
            _user.value = repository.getUser(id)
        }
    }
}
```

### Activity에서

```kotlin
class MainActivity : AppCompatActivity() {
    private val viewModel: MyViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Process Death 후 복원
        if (savedInstanceState != null) {
            // 필요한 처리
        }
    }
}
```

## 🔗 관련 개념

- [[03-Android-Framework/생명주기/Activity-Lifecycle]]
- [[06-Architecture/ViewModel/SavedStateHandle]]

## 📚 더 보기

- [Android Developers - Saving UI State](https://developer.android.com/topic/libraries/architecture/saving-states)

---

**핵심 요약:** onDestroy 없이 kill. SavedStateHandle로 저장. "Don't keep activities"로 테스트.
