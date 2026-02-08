---
tags: [architecture, mvvm, viewmodel]
---

# MVVM

## 💡 핵심 개념

**MVVM**(Model-View-ViewModel)은 UI와 비즈니스 로직을 분리하는 패턴이다. View는 ViewModel을 관찰하고, ViewModel은 Model을 관리한다.

## 🔍 자세히

### 구조

```
View ←→ ViewModel ←→ Model (Repository)
│        │              │
UI      상태/이벤트    데이터
```

### 각 레이어

**Model (Repository)**
```kotlin
class UserRepository(
    private val api: UserApi,
    private val dao: UserDao
) {
    suspend fun getUser(id: String): User {
        return api.getUser(id).also {
            dao.insert(it)
        }
    }
}
```

**ViewModel**
```kotlin
class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Initial)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    fun loadUser(id: String) {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val user = repository.getUser(id)
                _uiState.value = UiState.Success(user)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message)
            }
        }
    }
}
```

**View (Compose)**
```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    when (uiState) {
        UiState.Loading -> LoadingView()
        is UiState.Success -> UserContent(uiState.data)
        is UiState.Error -> ErrorView(uiState.message)
    }
}
```

### UI State 패턴

```kotlin
sealed class UiState<out T> {
    object Initial : UiState<Nothing>()
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}
```

### 단방향 데이터 흐름

```
User Action → ViewModel → State Change → View Update
    ↑                                         │
    └─────────────────────────────────────────┘
```

## 💻 실전 예제

### 완전한 MVVM

```kotlin
// Repository
class UserRepository @Inject constructor(
    private val api: UserApi
) {
    suspend fun getUsers(): List<User> = api.getUsers()
}

// ViewModel
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Initial)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    init {
        loadUsers()
    }
    
    fun loadUsers() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val users = repository.getUsers()
                _uiState.value = UiState.Success(users)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Error")
            }
        }
    }
    
    fun retry() {
        loadUsers()
    }
}

// View
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    when (uiState) {
        UiState.Initial -> { }
        UiState.Loading -> CircularProgressIndicator()
        is UiState.Success -> {
            UserList((uiState as UiState.Success).data)
        }
        is UiState.Error -> {
            ErrorView(
                message = (uiState as UiState.Error).message,
                onRetry = { viewModel.retry() }
            )
        }
    }
}
```

## 🔗 관련 개념

- [[06-Architecture/ViewModel/ViewModel-기초]]
- [[06-Architecture/패턴/MVI]]
- [[06-Architecture/상태-관리/UiState]]

---

**핵심 요약:** View ←→ ViewModel ←→ Repository. StateFlow로 상태 관리. 단방향 데이터 흐름.
