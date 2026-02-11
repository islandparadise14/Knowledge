---
tags: [CS, 소프트웨어-공학, OOP, 객체지향, Kotlin, Android]
---

# OOP (객체지향 프로그래밍)

## 💡 핵심 개념

객체지향 프로그래밍(OOP)은 데이터와 그 데이터를 조작하는 함수를 하나의 **객체**로 묶어 프로그래밍하는 패러다임이다. **캡슐화, 상속, 다형성, 추상화** 네 가지 핵심 원칙을 기반으로 하며, Kotlin은 이를 간결한 문법으로 지원한다.

## 📌 왜 필요한가?

Android 앱은 Activity, ViewModel, Repository 등 수많은 객체들의 협력으로 동작한다. OOP를 이해해야 Android 아키텍처 컴포넌트를 올바르게 설계하고 유지보수 가능한 코드를 작성할 수 있다.

## 🔍 자세히

### 1. 캡슐화 (Encapsulation)

내부 상태를 외부에서 직접 접근하지 못하도록 보호하고, 공개된 메서드를 통해서만 접근하게 한다.

```kotlin
class UserRepository(
    private val api: UserApi,       // 외부에서 접근 불가
    private val dao: UserDao
) {
    // 공개 인터페이스만 노출
    suspend fun getUser(id: String): User {
        return dao.getUser(id) ?: api.fetchUser(id).also {
            dao.insert(it)
        }
    }
}
```

### 2. 상속 (Inheritance)

Kotlin에서는 기본적으로 클래스가 `final`이므로, `open` 키워드를 명시해야 상속이 가능하다.

```kotlin
// 기본 클래스
open class BaseViewModel : ViewModel() {
    protected val _loading = MutableStateFlow(false)
    val loading: StateFlow<Boolean> = _loading.asStateFlow()
}

// 상속
class UserViewModel(
    private val repository: UserRepository
) : BaseViewModel() {
    fun loadUser(id: String) {
        viewModelScope.launch {
            _loading.value = true
            // ...
            _loading.value = false
        }
    }
}
```

### 3. 다형성 (Polymorphism)

하나의 인터페이스로 여러 구현을 다룰 수 있다. Android에서 테스트 용이성을 높이는 핵심 기법이다.

```kotlin
interface AuthProvider {
    suspend fun signIn(token: String): Result<User>
}

class GoogleAuthProvider : AuthProvider {
    override suspend fun signIn(token: String): Result<User> {
        // Google 로그인 구현
    }
}

class KakaoAuthProvider : AuthProvider {
    override suspend fun signIn(token: String): Result<User> {
        // 카카오 로그인 구현
    }
}

// 다형성 활용 - 어떤 구현인지 몰라도 동작
class LoginViewModel(private val authProvider: AuthProvider) : ViewModel()
```

### 4. 추상화 (Abstraction)

복잡한 내부 구현을 숨기고, 필요한 인터페이스만 노출한다.

```kotlin
// 추상 클래스
abstract class BasePagingSource<T : Any> : PagingSource<Int, T>() {
    abstract suspend fun loadPage(page: Int): List<T>

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, T> {
        val page = params.key ?: 1
        return try {
            val data = loadPage(page)
            LoadResult.Page(data, prevKey = if (page == 1) null else page - 1, nextKey = page + 1)
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
}
```

### 5. Kotlin 고유의 OOP 기능

#### data class - 값 객체에 최적

```kotlin
data class User(
    val id: String,
    val name: String,
    val email: String
)
// equals(), hashCode(), toString(), copy() 자동 생성
```

#### sealed class - 제한된 계층 구조

```kotlin
sealed class UiState<out T> {
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}

// when 분기에서 else 불필요
fun render(state: UiState<User>) = when (state) {
    is UiState.Loading -> showLoading()
    is UiState.Success -> showUser(state.data)
    is UiState.Error -> showError(state.message)
}
```

#### object - 싱글턴

```kotlin
object Analytics {
    fun trackEvent(name: String, params: Map<String, Any> = emptyMap()) {
        // Firebase Analytics 등에 이벤트 전송
    }
}
```

#### interface - 다중 구현 가능

```kotlin
interface Clickable {
    fun onClick()
}

interface LongClickable {
    fun onLongClick(): Boolean
}

class CustomButton : Clickable, LongClickable {
    override fun onClick() { /* ... */ }
    override fun onLongClick(): Boolean { /* ... */ return true }
}
```

## 🔗 관련 개념

- [[00-CS-기초/소프트웨어-공학/SOLID-원칙|SOLID 원칙]]
- [[00-CS-기초/소프트웨어-공학/추상화와-캡슐화|추상화와 캡슐화]]
- [[00-CS-기초/소프트웨어-공학/결합도와-응집도|결합도와 응집도]]
- [[00-CS-기초/소프트웨어-공학/의존성-주입-개념|의존성 주입]]
- [[00-CS-기초/소프트웨어-공학/함수형-프로그래밍-개념|함수형 프로그래밍]]

## 📚 더 보기

- [Kotlin 공식 - Classes and Inheritance](https://kotlinlang.org/docs/classes.html)
- [Kotlin 공식 - Sealed Classes](https://kotlinlang.org/docs/sealed-classes.html)
- [Android Developers - App Architecture](https://developer.android.com/topic/architecture)

---

**핵심 요약:** OOP는 캡슐화·상속·다형성·추상화를 통해 Android 앱을 모듈화하고, Kotlin의 data class·sealed class·object 등이 이를 간결하게 지원한다.
