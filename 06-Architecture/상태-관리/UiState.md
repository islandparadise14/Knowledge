# UiState 패턴

> UI 상태를 단일 객체로 관리하는 패턴

## 기본 구조

```kotlin
data class MainUiState(
    val isLoading: Boolean = false,
    val items: List<Item> = emptyList(),
    val error: String? = null
)
```

## Sealed Class로 상태 분리

```kotlin
sealed class UiState<out T> {
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}
```

## ViewModel에서 사용

```kotlin
class MainViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<UiState<List<Item>>>(UiState.Loading)
    val uiState: StateFlow<UiState<List<Item>>> = _uiState.asStateFlow()

    fun loadItems() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading

            repository.getItems()
                .onSuccess { items ->
                    _uiState.value = UiState.Success(items)
                }
                .onFailure { error ->
                    _uiState.value = UiState.Error(error.message ?: "Unknown error")
                }
        }
    }
}
```

## Compose에서 처리

```kotlin
@Composable
fun MainScreen(viewModel: MainViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        is UiState.Loading -> {
            CircularProgressIndicator()
        }
        is UiState.Success -> {
            ItemList(items = state.data)
        }
        is UiState.Error -> {
            ErrorMessage(message = state.message)
        }
    }
}
```

## 복합 상태

```kotlin
data class HomeUiState(
    val userState: UiState<User> = UiState.Loading,
    val postsState: UiState<List<Post>> = UiState.Loading,
    val isRefreshing: Boolean = false
)
```

## 일회성 이벤트 분리

```kotlin
class MainViewModel : ViewModel() {
    // 상태 (화면에 표시)
    private val _uiState = MutableStateFlow(MainUiState())
    val uiState: StateFlow<MainUiState> = _uiState.asStateFlow()

    // 이벤트 (토스트, 네비게이션 등)
    private val _events = MutableSharedFlow<UiEvent>()
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()

    fun onDeleteClick(id: String) {
        viewModelScope.launch {
            repository.delete(id)
            _events.emit(UiEvent.ShowToast("삭제됨"))
        }
    }
}

sealed class UiEvent {
    data class ShowToast(val message: String) : UiEvent()
    data class Navigate(val route: String) : UiEvent()
}
```

---

## 관련 문서
- [[06-Architecture/패턴/MVI|MVI]]

[← Architecture](06-Architecture/README.md)