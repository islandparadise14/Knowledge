---
tags: [coroutines, stateflow, hot-stream, state-management]
---

# StateFlow

## 💡 핵심 개념

**StateFlow**는 현재 상태를 가지고 있는 Hot Flow다. 항상 최신 값을 가지며, 새로운 구독자는 즉시 현재 값을 받는다. LiveData의 코루틴 버전이다.

## 📌 왜 필요한가?

UI 상태 관리에 최적화돼 있다. LiveData보다 강력하고, 코루틴과 완벽히 통합된다. 상태의 단일 소스(single source of truth)를 제공한다.

## 🔍 자세히

### 기본 사용법

```kotlin
// MutableStateFlow 생성
private val _uiState = MutableStateFlow(UiState.Loading)

// 읽기 전용으로 노출
val uiState: StateFlow<UiState> = _uiState.asStateFlow()

// 값 변경
_uiState.value = UiState.Success(data)

// 값 읽기
val current = _uiState.value
```

### StateFlow vs LiveData

```kotlin
// LiveData
private val _data = MutableLiveData<String>()
val data: LiveData<String> = _data

_data.value = "New value"  // Main 스레드만
_data.postValue("New value")  // 어느 스레드든

// StateFlow
private val _data = MutableStateFlow("")
val data: StateFlow<String> = _data.asStateFlow()

_data.value = "New value"  // 어느 스레드든 OK
```

### collect vs collectAsState

```kotlin
// ViewModel
class MyViewModel : ViewModel() {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count.asStateFlow()
}

// Compose에서
@Composable
fun CounterScreen(viewModel: MyViewModel) {
    // collectAsStateWithLifecycle - Compose 권장
    val count by viewModel.count.collectAsStateWithLifecycle()
    
    Text("Count: $count")
}

// Activity/Fragment에서
lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.count.collect { count ->
            updateUI(count)
        }
    }
}
```

### update 함수

```kotlin
// 안전한 업데이트
_uiState.update { currentState ->
    currentState.copy(isLoading = false)
}

// 직접 수정은 race condition 가능
// 나쁨
_uiState.value = _uiState.value.copy(count = _uiState.value.count + 1)

// 좋음
_uiState.update { it.copy(count = it.count + 1) }
```

### 초기값 필수

```kotlin
// 반드시 초기값 필요
val state = MutableStateFlow<User?>(null)  // OK
// val state = MutableStateFlow<User>()  // ERROR

// sealed class로 상태 관리 (권장)
sealed class UiState {
    object Initial : UiState()
    object Loading : UiState()
    data class Success(val data: String) : UiState()
    data class Error(val message: String) : UiState()
}

val state = MutableStateFlow<UiState>(UiState.Initial)
```

### 동작 원리

**Hot Stream**

```kotlin
val stateFlow = MutableStateFlow(0)

// Collector 1
launch {
    stateFlow.collect { println("1: $it") }
}

// Collector 2
launch {
    stateFlow.collect { println("2: $it") }
}

stateFlow.value = 1  // 모든 collector가 받음
// 출력: 1: 1, 2: 1
```

**Conflation (중복 제거)**

```kotlin
val state = MutableStateFlow(0)

launch {
    state.collect { value ->
        println(value)
        delay(100)
    }
}

// 빠르게 변경
repeat(10) {
    state.value = it
}
// 일부 값이 스킵될 수 있음 (최신 값만 받음)
```

**distinctUntilChanged**

```kotlin
val state = MutableStateFlow(0)

state.value = 1  // 방출
state.value = 1  // 스킵 (같은 값)
state.value = 2  // 방출
```

### 주의사항

**1. 초기값 설정 주의**

```kotlin
// 나쁨 - nullable 남발
val state = MutableStateFlow<User?>(null)

// 좋음 - sealed class
sealed class UserState {
    object NotLoaded : UserState()
    object Loading : UserState()
    data class Loaded(val user: User) : UserState()
}
val state = MutableStateFlow<UserState>(UserState.NotLoaded)
```

**2. value 접근은 동기적**

```kotlin
// value는 즉시 반환
val current = state.value  // 블로킹 없음

// collect는 suspend
state.collect { }  // suspend 함수
```

**3. Lifecycle 고려**

```kotlin
// 나쁨 - Activity onDestroy 후에도 수집
lifecycleScope.launch {
    viewModel.state.collect { }
}

// 좋음 - STARTED 상태에서만
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.state.collect { }
    }
}
```

**4. StateFlow는 항상 값을 가짐**

```kotlin
// StateFlow는 nullable이어도 초기값 필요
val state: StateFlow<User?> = MutableStateFlow(null)

// 초기 값이 없는 상태는 SharedFlow 사용
```

## 💻 실전 예제

### ViewModel 상태 관리

```kotlin
data class UiState(
    val users: List<User> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

class UserViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(UiState())
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    fun loadUsers() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            
            try {
                val users = repository.getUsers()
                _uiState.update { 
                    it.copy(users = users, isLoading = false)
                }
            } catch (e: Exception) {
                _uiState.update { 
                    it.copy(error = e.message, isLoading = false)
                }
            }
        }
    }
}
```

### Compose 통합

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    when {
        uiState.isLoading -> LoadingView()
        uiState.error != null -> ErrorView(uiState.error!!)
        else -> UserList(uiState.users)
    }
}
```

### 여러 StateFlow 결합

```kotlin
class CombinedViewModel : ViewModel() {
    private val _users = MutableStateFlow<List<User>>(emptyList())
    private val _filter = MutableStateFlow("")
    
    val filteredUsers: StateFlow<List<User>> = combine(
        _users,
        _filter
    ) { users, filter ->
        if (filter.isEmpty()) users
        else users.filter { it.name.contains(filter, ignoreCase = true) }
    }.stateIn(
        viewModelScope,
        SharingStarted.WhileSubscribed(5000),
        emptyList()
    )
}
```

### Repository에서 StateFlow

```kotlin
class UserRepository {
    private val _users = MutableStateFlow<List<User>>(emptyList())
    val users: StateFlow<List<User>> = _users.asStateFlow()
    
    suspend fun refresh() {
        _users.value = api.getUsers()
    }
    
    fun addUser(user: User) {
        _users.update { it + user }
    }
}
```

### 검색 기능

```kotlin
class SearchViewModel : ViewModel() {
    val searchQuery = MutableStateFlow("")
    
    val searchResults: StateFlow<List<Item>> = searchQuery
        .debounce(300)
        .filter { it.length >= 2 }
        .flatMapLatest { query ->
            repository.search(query)
        }
        .stateIn(
            viewModelScope,
            SharingStarted.WhileSubscribed(5000),
            emptyList()
        )
}
```

### 로딩 상태 관리

```kotlin
class DataViewModel : ViewModel() {
    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()
    
    suspend fun <T> withLoading(block: suspend () -> T): T {
        _isLoading.value = true
        try {
            return block()
        } finally {
            _isLoading.value = false
        }
    }
    
    fun loadData() {
        viewModelScope.launch {
            withLoading {
                repository.getData()
            }
        }
    }
}
```

### stateIn으로 Flow를 StateFlow로

```kotlin
// Cold Flow
fun getDataFlow(): Flow<Data> = flow {
    emit(fetchData())
}

// Hot StateFlow로 변환
val dataState: StateFlow<Data?> = getDataFlow()
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = null
    )
```

## 🔗 관련 개념

- [[02-Coroutines/Flow/SharedFlow]]
- [[02-Coroutines/Flow/Flow-기초]]
- [[06-Architecture/상태-관리/UiState]]

## 📚 더 보기

- [Kotlin Docs - StateFlow](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/)

---

**핵심 요약:** Hot Flow. 항상 값 보유. UI 상태 관리용. update()로 안전하게 변경. LiveData 대체.
