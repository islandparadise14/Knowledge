---
tags: [CS, 소프트웨어-공학, SOLID, 설계원칙, Android, Kotlin]
---

# SOLID 원칙

## 💡 핵심 개념

SOLID는 객체지향 설계의 다섯 가지 원칙(SRP, OCP, LSP, ISP, DIP)을 말한다. 코드의 유연성, 확장성, 유지보수성을 높이기 위한 가이드라인이며, Android 아키텍처 설계의 근간이 된다.

## 📌 왜 필요한가?

Android 앱은 요구사항 변경이 빈번하다. SOLID 원칙을 따르면 새 기능 추가 시 기존 코드 수정을 최소화하고, 테스트 가능한 구조를 만들 수 있다.

## 🔍 자세히

### S - 단일 책임 원칙 (Single Responsibility Principle)

클래스는 하나의 책임만 가져야 하며, 변경 이유가 하나뿐이어야 한다.

```kotlin
// ❌ 나쁜 예: 여러 책임이 섞임
class UserViewModel : ViewModel() {
    fun loadUser() { /* 네트워크 호출 */ }
    fun saveToDb() { /* DB 저장 */ }
    fun formatDate() { /* 날짜 포맷 */ }
    fun validateEmail() { /* 이메일 검증 */ }
}

// ✅ 좋은 예: 책임 분리
class UserRepository(private val api: UserApi, private val dao: UserDao) {
    suspend fun getUser(id: String): User = /* ... */
}

class DateFormatter {
    fun format(date: LocalDateTime): String = /* ... */
}

class EmailValidator {
    fun isValid(email: String): Boolean = /* ... */
}

class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {
    fun loadUser(id: String) { /* repository만 사용 */ }
}
```

### O - 개방-폐쇄 원칙 (Open-Closed Principle)

확장에는 열려 있고, 수정에는 닫혀 있어야 한다.

```kotlin
// ✅ sealed class로 OCP 적용
sealed class AnalyticsEvent(val name: String) {
    data class ScreenView(val screenName: String) : AnalyticsEvent("screen_view")
    data class ButtonClick(val buttonId: String) : AnalyticsEvent("button_click")
    // 새 이벤트 추가 시 기존 코드 수정 불필요
    data class Purchase(val amount: Double) : AnalyticsEvent("purchase")
}

class AnalyticsTracker(private val providers: List<AnalyticsProvider>) {
    fun track(event: AnalyticsEvent) {
        providers.forEach { it.log(event) }  // provider 추가해도 이 코드는 변경 없음
    }
}
```

### L - 리스코프 치환 원칙 (Liskov Substitution Principle)

하위 타입은 상위 타입으로 교체해도 프로그램의 정확성이 유지되어야 한다.

```kotlin
// ✅ 좋은 예: 어떤 구현이든 Repository 인터페이스를 대체 가능
interface UserRepository {
    suspend fun getUser(id: String): User
}

class RemoteUserRepository(private val api: UserApi) : UserRepository {
    override suspend fun getUser(id: String): User = api.fetchUser(id)
}

class FakeUserRepository : UserRepository {
    override suspend fun getUser(id: String): User = User(id, "Test User")
}

// 테스트에서 FakeUserRepository로 교체해도 ViewModel은 정상 동작
```

### I - 인터페이스 분리 원칙 (Interface Segregation Principle)

클라이언트가 사용하지 않는 메서드에 의존하면 안 된다.

```kotlin
// ❌ 나쁜 예: 너무 큰 인터페이스
interface UserManager {
    fun login(email: String, password: String)
    fun logout()
    fun updateProfile(name: String)
    fun deleteAccount()
    fun exportData()
}

// ✅ 좋은 예: 역할별로 분리
interface Authenticator {
    fun login(email: String, password: String)
    fun logout()
}

interface ProfileEditor {
    fun updateProfile(name: String)
}

interface AccountManager {
    fun deleteAccount()
    fun exportData()
}
```

### D - 의존 역전 원칙 (Dependency Inversion Principle)

상위 모듈이 하위 모듈에 직접 의존하지 않고, 둘 다 추상화에 의존해야 한다.

```kotlin
// ❌ 나쁜 예: 구체 클래스에 직접 의존
class UserViewModel {
    private val repository = RetrofitUserRepository()  // 구체 클래스 직접 생성
}

// ✅ 좋은 예: 추상화에 의존 + Hilt로 주입
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository  // 인터페이스에 의존
) : ViewModel()

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    abstract fun bindUserRepository(impl: RemoteUserRepository): UserRepository
}
```

### SOLID와 Android 아키텍처

| 원칙 | Android 적용 사례 |
|------|-------------------|
| SRP | ViewModel은 UI 로직, Repository는 데이터 로직 |
| OCP | RecyclerView.Adapter의 ViewType 확장 |
| LSP | Room DAO 인터페이스를 Fake로 교체하여 테스트 |
| ISP | Callback 인터페이스를 기능별로 분리 |
| DIP | Hilt/Dagger로 인터페이스 기반 의존성 주입 |

## 🔗 관련 개념

- [[00-CS-기초/소프트웨어-공학/OOP-객체지향|OOP 객체지향]]
- [[00-CS-기초/소프트웨어-공학/의존성-주입-개념|의존성 주입]]
- [[00-CS-기초/소프트웨어-공학/결합도와-응집도|결합도와 응집도]]
- [[00-CS-기초/소프트웨어-공학/클린-코드|클린 코드]]

## 📚 더 보기

- [Wikipedia - SOLID](https://en.wikipedia.org/wiki/SOLID)
- [Android Developers - Guide to App Architecture](https://developer.android.com/topic/architecture)
- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

---

**핵심 요약:** SOLID 원칙은 Android 앱을 테스트 가능하고 확장 가능하게 설계하기 위한 다섯 가지 핵심 가이드라인이며, 현대 Android 아키텍처(MVVM, Clean Architecture)의 기반이다.
