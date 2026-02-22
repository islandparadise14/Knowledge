# ViewModel 기초

> UI 관련 데이터를 생명주기에 맞게 관리

## ViewModel이란?

- UI 데이터를 저장하고 관리
- Configuration Change에도 데이터 유지
- Activity/Fragment보다 긴 생명주기

## 기본 사용

```kotlin
class MainViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(UiState())
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun loadData() {
        viewModelScope.launch {
            _uiState.value = UiState(loading = true)
            val data = repository.getData()
            _uiState.value = UiState(data = data)
        }
    }
}
```

## Activity/Fragment에서 사용

```kotlin
class MainActivity : AppCompatActivity() {
    private val viewModel: MainViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    // UI 업데이트
                }
            }
        }
    }
}
```

## Compose에서 사용

```kotlin
@Composable
fun MainScreen(
    viewModel: MainViewModel = viewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    // UI
}
```

## viewModelScope

```kotlin
class MyViewModel : ViewModel() {
    init {
        viewModelScope.launch {
            // ViewModel이 clear될 때 자동 취소
        }
    }
}
```

## SavedStateHandle

프로세스 종료 후에도 데이터 복원:

```kotlin
class MyViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    val query = savedStateHandle.getStateFlow("query", "")

    fun setQuery(value: String) {
        savedStateHandle["query"] = value
    }
}
```

## 생명주기

```
Activity/Fragment created
    ↓
ViewModel created (처음 한 번)
    ↓
Configuration Change → Activity recreated → ViewModel 유지
    ↓
Activity finished
    ↓
ViewModel.onCleared()
```

## 주의사항

- **Context 참조 금지**: Memory Leak 발생
- **View 참조 금지**: 생명주기 불일치
- Application Context 필요하면 `AndroidViewModel` 사용

---

## 관련 문서
- [[06-Architecture/패턴/MVVM|MVVM]]

[← Architecture](06-Architecture/README.md)