---
tags: [kotlin, sealed-class, pattern-matching]
---

# Sealed Class

## 💡 핵심 개념

**Sealed class**는 제한된 클래스 계층을 나타낸다. 모든 서브클래스를 컴파일 타임에 알 수 있어서, `when` 표현식에서 `else` 없이 완전한 분기 처리가 가능하다.

## 📌 왜 필요한가?

상태 관리나 API 응답 처리 시 명확한 타입 안전성을 제공한다. enum은 단일 인스턴스만 가능하지만, sealed class는 각 케이스마다 다른 데이터를 가질 수 있다.

## 🔍 자세히

### 기본 사용법

```kotlin
// Sealed class 정의
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Exception) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

// when과 함께 사용 (else 불필요)
fun handleResult(result: Result<String>) {
    when (result) {
        is Result.Success -> println("Data: ${result.data}")
        is Result.Error -> println("Error: ${result.exception}")
        Result.Loading -> println("Loading...")
    }
    // else가 필요 없음! 컴파일러가 모든 케이스를 알고 있음
}
```

### Sealed interface (Kotlin 1.5+)

```kotlin
sealed interface UiState {
    object Loading : UiState
    data class Success(val data: String) : UiState
    data class Error(val message: String) : UiState
}

// 장점: 다중 상속 가능
sealed interface NetworkState
sealed interface CacheState

class DataState : NetworkState, CacheState
```

### Enum vs Sealed Class

```kotlin
// Enum - 단일 인스턴스, 상태만
enum class Status {
    LOADING,
    SUCCESS,
    ERROR
    // 데이터를 담을 수 없음!
}

// Sealed class - 각 케이스마다 다른 데이터
sealed class UiState {
    object Loading : UiState()
    data class Success(val data: List<String>) : UiState()
    data class Error(val message: String, val code: Int) : UiState()
}
```

### 계층 구조

```kotlin
// 계층적 sealed class
sealed class Animal {
    abstract val name: String
    
    sealed class Mammal : Animal() {
        data class Dog(override val name: String) : Mammal()
        data class Cat(override val name: String) : Mammal()
    }
    
    sealed class Bird : Animal() {
        data class Eagle(override val name: String) : Bird()
        data class Penguin(override val name: String) : Bird()
    }
}

fun describe(animal: Animal) {
    when (animal) {
        is Animal.Mammal.Dog -> println("Dog: ${animal.name}")
        is Animal.Mammal.Cat -> println("Cat: ${animal.name}")
        is Animal.Bird.Eagle -> println("Eagle: ${animal.name}")
        is Animal.Bird.Penguin -> println("Penguin: ${animal.name}")
    }
}
```

### 동작 원리

**Sealed class의 제약**

```kotlin
// 1. 서브클래스는 같은 파일 또는 같은 패키지 안에만
// 2. abstract class처럼 직접 인스턴스화 불가

sealed class Result
// class MyResult : Result()  // 다른 파일이면 ERROR

// 3. 생성자는 private 또는 protected만
sealed class Result private constructor()
```

**컴파일러 최적화**

```kotlin
sealed class Color {
    object Red : Color()
    object Green : Color()
    object Blue : Color()
}

// when이 완전하면 else 불필요
fun describe(color: Color) = when (color) {
    Color.Red -> "빨강"
    Color.Green -> "초록"
    Color.Blue -> "파랑"
    // else 불필요
}

// 새로운 케이스 추가하면?
sealed class Color {
    object Red : Color()
    object Green : Color()
    object Blue : Color()
    object Yellow : Color()  // 추가!
}

// 컴파일 에러! when이 Yellow 케이스를 처리 안 함
// -> 빠뜨린 케이스를 컴파일 타임에 잡아냄
```

### 주의사항

**1. object vs class vs data class**

```kotlin
sealed class State {
    // object - 싱글톤, 데이터 없음
    object Loading : State()
    
    // data class - 데이터 포함
    data class Success(val data: String) : State()
    
    // class - 커스텀 로직 필요 시
    class Error(val exception: Exception) : State() {
        val message: String get() = exception.localizedMessage ?: "Unknown"
    }
}
```

**2. Nothing 타입 활용**

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Exception) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

// T가 필요 없는 케이스는 Nothing 사용
```

**3. Copy와 불변성**

```kotlin
sealed class UiState {
    data class Loaded(
        val users: List<User>,
        val isRefreshing: Boolean = false
    ) : UiState()
}

val state = UiState.Loaded(users = listOf(...))
val updated = (state as? UiState.Loaded)?.copy(isRefreshing = true)
```

## 💻 실전 예제

### API 응답 처리

```kotlin
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(
        val exception: Exception,
        val code: Int? = null
    ) : ApiResult<Nothing>()
    object Loading : ApiResult<Nothing>()
}

class UserRepository {
    suspend fun getUser(id: String): ApiResult<User> {
        return try {
            ApiResult.Success(api.getUser(id))
        } catch (e: HttpException) {
            ApiResult.Error(e, e.code())
        } catch (e: Exception) {
            ApiResult.Error(e)
        }
    }
}
```

### ViewModel UI State

```kotlin
sealed class UserUiState {
    object Initial : UserUiState()
    object Loading : UserUiState()
    data class Success(val user: User) : UserUiState()
    data class Error(val message: String) : UserUiState()
}

class UserViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<UserUiState>(UserUiState.Initial)
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()
    
    fun loadUser(id: String) {
        viewModelScope.launch {
            _uiState.value = UserUiState.Loading
            
            when (val result = repository.getUser(id)) {
                is ApiResult.Success -> {
                    _uiState.value = UserUiState.Success(result.data)
                }
                is ApiResult.Error -> {
                    _uiState.value = UserUiState.Error(result.exception.message ?: "Unknown")
                }
                ApiResult.Loading -> { }
            }
        }
    }
}
```

### Compose UI 렌더링

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    when (uiState) {
        UserUiState.Initial -> { }
        UserUiState.Loading -> {
            CircularProgressIndicator()
        }
        is UserUiState.Success -> {
            val user = (uiState as UserUiState.Success).user
            UserContent(user)
        }
        is UserUiState.Error -> {
            val message = (uiState as UserUiState.Error).message
            ErrorView(message)
        }
    }
}
```

### Navigation Events

```kotlin
sealed class NavigationEvent {
    object NavigateBack : NavigationEvent()
    data class NavigateTo(val route: String) : NavigationEvent()
    data class NavigateToWithArgs(
        val route: String,
        val args: Map<String, Any>
    ) : NavigationEvent()
}

class MyViewModel : ViewModel() {
    private val _navigationEvent = Channel<NavigationEvent>()
    val navigationEvent = _navigationEvent.receiveAsFlow()
    
    fun onBackClick() {
        viewModelScope.launch {
            _navigationEvent.send(NavigationEvent.NavigateBack)
        }
    }
}
```

### Resource 패턴

```kotlin
sealed class Resource<out T> {
    data class Success<T>(val data: T) : Resource<T>()
    data class Error(val exception: Throwable) : Resource<Nothing>()
    object Loading : Resource<Nothing>()
    
    // 유틸리티 함수들
    fun <R> map(transform: (T) -> R): Resource<R> {
        return when (this) {
            is Success -> Success(transform(data))
            is Error -> this
            Loading -> Loading
        }
    }
    
    fun onSuccess(action: (T) -> Unit): Resource<T> {
        if (this is Success) action(data)
        return this
    }
    
    fun onError(action: (Throwable) -> Unit): Resource<T> {
        if (this is Error) action(exception)
        return this
    }
}

// 사용
repository.getUser(id)
    .onSuccess { user -> updateUI(user) }
    .onError { error -> showError(error) }
```

## 🔗 관련 개념

- [[01-Kotlin/클래스-종류/Data-Class]]
- [[01-Kotlin/클래스-종류/Enum-Class]]
- [[06-Architecture/상태-관리/UiState]]

## 📚 더 보기

- [Kotlin Docs - Sealed Classes](https://kotlinlang.org/docs/sealed-classes.html)

---

**핵심 요약:** Sealed class는 제한된 타입 계층. when에서 완전한 분기. API 응답과 UI 상태 관리에 필수.
