---
tags: [coroutines, dispatcher, thread]
---

# Dispatcher

## 💡 핵심 개념

**Dispatcher**는 코루틴이 어느 스레드에서 실행될지 결정한다. Main, IO, Default, Unconfined 4가지 기본 Dispatcher가 있다.

## 📌 왜 필요한가?

작업 종류에 따라 적절한 스레드를 선택해야 한다. UI 업데이트는 Main, 네트워크/DB는 IO, CPU 집약 작업은 Default를 사용한다.

## 🔍 자세히

### Main Dispatcher

```kotlin
// UI 스레드에서 실행
launch(Dispatchers.Main) {
    textView.text = "Updated"  // UI 업데이트
}

// Compose
LaunchedEffect(Unit) {
    // 기본적으로 Main dispatcher
}
```

**사용 시기:**
- UI 업데이트
- LiveData/StateFlow 업데이트
- Fragment/Activity 작업

### IO Dispatcher

```kotlin
// 네트워크, DB, 파일 I/O
withContext(Dispatchers.IO) {
    val data = api.fetchData()  // 네트워크
    dao.insert(data)  // DB
    File("path").readText()  // 파일
}
```

**특징:**
- 스레드 풀 크기: 64 (또는 CPU 코어 수 중 큰 값)
- 블로킹 I/O 작업에 최적화
- 스레드가 많아서 블로킹돼도 다른 작업 가능

**사용 시기:**
- 네트워크 요청 (Retrofit, OkHttp)
- 데이터베이스 작업 (Room)
- 파일 I/O
- SharedPreferences, DataStore

### Default Dispatcher

```kotlin
// CPU 집약적 작업
withContext(Dispatchers.Default) {
    val result = complexCalculation()
    val sorted = largeList.sortedBy { it.value }
    val encrypted = encrypt(data)
}
```

**특징:**
- 스레드 풀 크기: CPU 코어 수
- CPU 집약적 작업에 최적화
- 블로킹하면 다른 작업이 대기해야 함

**사용 시기:**
- 정렬, 필터링 등 컬렉션 연산
- JSON 파싱
- 암호화/복호화
- 이미지 처리

### Unconfined Dispatcher

```kotlin
// 첫 중단 지점까지는 현재 스레드, 이후 불확실
launch(Dispatchers.Unconfined) {
    println("1: ${Thread.currentThread().name}")  // 현재 스레드
    delay(100)
    println("2: ${Thread.currentThread().name}")  // 다를 수 있음!
}
```

**주의:** 거의 사용하지 않음. 테스트나 특수한 경우에만.

### 커스텀 Dispatcher

```kotlin
// 단일 스레드 Dispatcher
val singleThread = newSingleThreadContext("MyThread")

// 고정 스레드 풀
val fixedPool = newFixedThreadPoolContext(4, "MyPool")

// Executor를 Dispatcher로 변환
val executor = Executors.newCachedThreadPool()
val dispatcher = executor.asCoroutineDispatcher()

// 사용 후 정리 필수
dispatcher.close()
```

### Dispatcher 전환

```kotlin
suspend fun loadUser() {
    // Main에서 시작
    updateUI("Loading...")
    
    // IO로 전환
    val user = withContext(Dispatchers.IO) {
        api.getUser()
    }
    
    // 자동으로 Main으로 복귀
    updateUI("Loaded: ${user.name}")
}
```

### 동작 원리

**스레드 풀 관리**

```kotlin
// Dispatchers.IO 내부 (단순화)
object IO : CoroutineDispatcher() {
    private val threadPool = ThreadPoolExecutor(
        corePoolSize = 0,
        maximumPoolSize = 64,
        ...
    )
    
    override fun dispatch(context: CoroutineContext, block: Runnable) {
        threadPool.execute(block)
    }
}
```

**limitedParallelism**

```kotlin
// IO에서 동시 실행 제한
val limitedIO = Dispatchers.IO.limitedParallelism(10)

repeat(100) {
    launch(limitedIO) {
        // 최대 10개만 동시 실행
    }
}
```

### 주의사항

**1. Main에서 블로킹 작업 금지**

```kotlin
// 나쁨
launch(Dispatchers.Main) {
    Thread.sleep(1000)  // UI 멈춤!
}

// 좋음
launch(Dispatchers.Main) {
    withContext(Dispatchers.IO) {
        Thread.sleep(1000)  // 백그라운드에서
    }
}
```

**2. Dispatcher 남용 금지**

```kotlin
// 나쁨 - 불필요한 전환
withContext(Dispatchers.IO) {
    withContext(Dispatchers.Default) {
        // 전환 비용만 낭비
    }
}

// 좋음
withContext(Dispatchers.IO) {
    // 하나의 Dispatcher로
}
```

**3. 커스텀 Dispatcher는 close() 필수**

```kotlin
val dispatcher = newSingleThreadContext("MyThread")
try {
    withContext(dispatcher) {
        // ...
    }
} finally {
    dispatcher.close()  // 필수!
}
```

**4. suspend 함수는 Dispatcher 명시 X**

```kotlin
// 나쁨 - Repository에서 Dispatcher 명시
suspend fun getUser() = withContext(Dispatchers.IO) {
    api.getUser()
}

// 좋음 - 호출하는 쪽에서 결정
suspend fun getUser() = api.getUser()

// ViewModel에서
viewModelScope.launch {
    val user = withContext(Dispatchers.IO) {
        repository.getUser()
    }
}
```

## 💻 실전 예제

### Repository 패턴

```kotlin
class UserRepository(
    private val api: UserApi,
    private val dao: UserDao
) {
    // suspend 함수는 Dispatcher 명시 X
    suspend fun getUser(id: String): User {
        return api.getUser(id).also {
            dao.insert(it)
        }
    }
}

// ViewModel에서 Dispatcher 결정
class UserViewModel : ViewModel() {
    fun loadUser(id: String) {
        viewModelScope.launch {
            val user = withContext(Dispatchers.IO) {
                repository.getUser(id)
            }
            _uiState.value = UiState.Success(user)
        }
    }
}
```

### 병렬 다운로드

```kotlin
suspend fun downloadFiles(urls: List<String>) = coroutineScope {
    urls.map { url ->
        async(Dispatchers.IO) {
            downloadFile(url)
        }
    }.awaitAll()
}
```

### 이미지 처리

```kotlin
suspend fun processImage(bitmap: Bitmap): Bitmap {
    return withContext(Dispatchers.Default) {
        // CPU 집약적 작업
        val processed = applyFilter(bitmap)
        resize(processed)
    }
}
```

### DB + 네트워크

```kotlin
suspend fun syncData() {
    withContext(Dispatchers.IO) {
        // 네트워크
        val data = api.fetchData()
        
        // DB (같은 IO Dispatcher)
        dao.insertAll(data)
    }
}
```

### limitedParallelism 활용

```kotlin
class FileUploader {
    // 동시 업로드 제한
    private val uploadDispatcher = Dispatchers.IO.limitedParallelism(3)
    
    suspend fun uploadFiles(files: List<File>) = coroutineScope {
        files.map { file ->
            async(uploadDispatcher) {
                uploadFile(file)
            }
        }.awaitAll()
    }
}
```

### Compose에서

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel) {
    LaunchedEffect(Unit) {
        // 기본 Dispatchers.Main
        viewModel.loadUser()
    }
    
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    // UI 렌더링
}
```

### 테스트에서 Dispatcher 교체

```kotlin
class TestDispatcherRule : TestWatcher() {
    val testDispatcher = UnconfinedTestDispatcher()
    
    override fun starting(description: Description) {
        Dispatchers.setMain(testDispatcher)
    }
    
    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}

@get:Rule
val testDispatcherRule = TestDispatcherRule()

@Test
fun test() = runTest {
    // Main Dispatcher가 testDispatcher로 교체됨
}
```

## 🔗 관련 개념

- [[02-Coroutines/기초/CoroutineContext]]
- [[02-Coroutines/빌더/WithContext]]
- [[02-Coroutines/실전-패턴/병렬-처리]]

## 📚 더 보기

- [Kotlin Docs - Dispatchers](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)

---

**핵심 요약:** Main(UI), IO(네트워크/DB), Default(CPU). withContext로 전환. Main에서 블로킹 금지.
