# Repository 패턴

> 데이터 소스를 추상화하여 비즈니스 로직과 분리

## 왜 필요한가?

- 데이터 소스 교체 용이 (API → DB)
- 테스트 용이 (Mock 주입)
- 관심사 분리

## 기본 구조

```
ViewModel → Repository → DataSource (API, DB, Cache)
```

## 구현 예시

### Interface 정의
```kotlin
interface UserRepository {
    suspend fun getUser(id: String): Result<User>
    suspend fun saveUser(user: User): Result<Unit>
    fun observeUsers(): Flow<List<User>>
}
```

### 구현체
```kotlin
class UserRepositoryImpl(
    private val api: UserApi,
    private val dao: UserDao,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) : UserRepository {

    override suspend fun getUser(id: String): Result<User> {
        return withContext(ioDispatcher) {
            try {
                // 네트워크 우선
                val user = api.getUser(id)
                dao.insert(user)  // 캐시
                Result.success(user)
            } catch (e: Exception) {
                // 실패 시 로컬 캐시
                dao.getUser(id)?.let {
                    Result.success(it)
                } ?: Result.failure(e)
            }
        }
    }

    override fun observeUsers(): Flow<List<User>> {
        return dao.observeAll()
    }
}
```

## 캐싱 전략

### Cache-First
```kotlin
override suspend fun getData(): Data {
    // 1. 캐시 확인
    cache.get()?.let { return it }

    // 2. 네트워크 호출
    val data = api.getData()
    cache.set(data)
    return data
}
```

### Network-First
```kotlin
override suspend fun getData(): Data {
    return try {
        val data = api.getData()
        cache.set(data)
        data
    } catch (e: Exception) {
        cache.get() ?: throw e
    }
}
```

## Hilt로 DI

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds
    @Singleton
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl
    ): UserRepository
}
```

## ViewModel에서 사용

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {

    fun loadUser(id: String) {
        viewModelScope.launch {
            userRepository.getUser(id)
                .onSuccess { /* ... */ }
                .onFailure { /* ... */ }
        }
    }
}
```

---

## 관련 문서
- [[06-Architecture/패턴/MVVM|MVVM]]
- [[06-Architecture/의존성-주입/Hilt|Hilt]]

[← Architecture](06-Architecture/README.md)
