# MVI (Model-View-Intent)

> 단방향 데이터 흐름의 아키텍처 패턴

## 핵심 개념

```
User → Intent → Model → View → User
        ↑________________________|
```

- **Model**: 불변 상태 (Immutable State)
- **View**: 상태를 렌더링
- **Intent**: 사용자 액션, 이벤트

## MVVM과 차이

| MVVM | MVI |
|------|-----|
| 여러 LiveData/Flow | 단일 State |
| 양방향 가능 | 단방향 |
| 상태 분산 | 상태 집중 |

## 구현 예시

### State 정의
```kotlin
data class MainState(
    val isLoading: Boolean = false,
    val items: List<Item> = emptyList(),
    val error: String? = null
)
```

### Intent (Event) 정의
```kotlin
sealed class MainIntent {
    object LoadItems : MainIntent()
    data class DeleteItem(val id: String) : MainIntent()
    object Refresh : MainIntent()
}
```

### ViewModel
```kotlin
class MainViewModel : ViewModel() {
    private val _state = MutableStateFlow(MainState())
    val state: StateFlow<MainState> = _state.asStateFlow()

    fun processIntent(intent: MainIntent) {
        when (intent) {
            is MainIntent.LoadItems -> loadItems()
            is MainIntent.DeleteItem -> deleteItem(intent.id)
            is MainIntent.Refresh -> refresh()
        }
    }

    private fun loadItems() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }

            repository.getItems()
                .onSuccess { items ->
                    _state.update { it.copy(isLoading = false, items = items) }
                }
                .onFailure { error ->
                    _state.update { it.copy(isLoading = false, error = error.message) }
                }
        }
    }
}
```

### View (Compose)
```kotlin
@Composable
fun MainScreen(viewModel: MainViewModel = viewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) {
        viewModel.processIntent(MainIntent.LoadItems)
    }

    when {
        state.isLoading -> LoadingView()
        state.error != null -> ErrorView(state.error)
        else -> ItemList(
            items = state.items,
            onDelete = { viewModel.processIntent(MainIntent.DeleteItem(it)) }
        )
    }
}
```

## 장점

- 상태 예측 가능 (단일 소스)
- 디버깅 용이 (상태 변화 추적)
- 테스트 용이

## 단점

- 보일러플레이트 증가
- 간단한 화면에는 과도함

---

## 관련 문서
- [[06-Architecture/패턴/MVVM|MVVM]]
- [[06-Architecture/상태-관리/UiState|UiState 패턴]]

[← Architecture](06-Architecture/README.md)