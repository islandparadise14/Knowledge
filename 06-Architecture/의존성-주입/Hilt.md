# Hilt

> Android를 위한 의존성 주입 라이브러리

## 설정

```kotlin
// build.gradle (project)
plugins {
    id("com.google.dagger.hilt.android") version "2.48" apply false
}

// build.gradle (app)
plugins {
    id("com.google.dagger.hilt.android")
    id("kotlin-kapt")
}

dependencies {
    implementation("com.google.dagger:hilt-android:2.48")
    kapt("com.google.dagger:hilt-compiler:2.48")
}
```

## 기본 사용

### Application
```kotlin
@HiltAndroidApp
class MyApplication : Application()
```

### Activity/Fragment
```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {

    @Inject
    lateinit var repository: UserRepository
}
```

### ViewModel
```kotlin
@HiltViewModel
class MainViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel()
```

## Module 정의

### @Binds
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

### @Provides
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    @Provides
    @Singleton
    fun provideUserApi(retrofit: Retrofit): UserApi {
        return retrofit.create(UserApi::class.java)
    }
}
```

## Component 범위

| Component | 범위 | Scope |
|-----------|------|-------|
| SingletonComponent | Application | @Singleton |
| ActivityComponent | Activity | @ActivityScoped |
| ViewModelComponent | ViewModel | @ViewModelScoped |
| FragmentComponent | Fragment | @FragmentScoped |

## Qualifier

같은 타입의 다른 인스턴스 구분:

```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class IoDispatcher

@Module
@InstallIn(SingletonComponent::class)
object DispatcherModule {

    @Provides
    @IoDispatcher
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO
}

// 사용
class MyRepository @Inject constructor(
    @IoDispatcher private val dispatcher: CoroutineDispatcher
)
```

## Compose에서 사용

```kotlin
@Composable
fun MainScreen(
    viewModel: MainViewModel = hiltViewModel()
) {
    // ...
}
```

---

## 관련 문서
- [[06-Architecture/패턴/Repository-패턴|Repository 패턴]]
- [[06-Architecture/ViewModel/ViewModel-기초|ViewModel 기초]]

[← Architecture](06-Architecture/README.md)