# JUnit

> Java/Kotlin 단위 테스트 프레임워크

## 설정

```kotlin
// build.gradle
dependencies {
    testImplementation("junit:junit:4.13.2")
    // 또는 JUnit 5
    testImplementation("org.junit.jupiter:junit-jupiter:5.9.0")
}
```

## 기본 구조

```kotlin
class CalculatorTest {

    private lateinit var calculator: Calculator

    @Before  // 각 테스트 전 실행
    fun setup() {
        calculator = Calculator()
    }

    @Test
    fun `add returns sum of two numbers`() {
        // Given
        val a = 2
        val b = 3

        // When
        val result = calculator.add(a, b)

        // Then
        assertEquals(5, result)
    }

    @After  // 각 테스트 후 실행
    fun teardown() {
        // 정리 코드
    }
}
```

## 주요 Assertion

```kotlin
assertEquals(expected, actual)
assertNotEquals(unexpected, actual)
assertTrue(condition)
assertFalse(condition)
assertNull(value)
assertNotNull(value)
assertThrows<Exception> { /* code */ }
```

## 테스트 명명

```kotlin
// 백틱으로 읽기 쉬운 이름
@Test
fun `user with valid email should be created successfully`() { }

// Given-When-Then 패턴
@Test
fun givenValidEmail_whenCreateUser_thenSuccess() { }
```

## ViewModel 테스트

```kotlin
class MainViewModelTest {

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    private lateinit var viewModel: MainViewModel
    private lateinit var fakeRepository: FakeUserRepository

    @Before
    fun setup() {
        fakeRepository = FakeUserRepository()
        viewModel = MainViewModel(fakeRepository)
    }

    @Test
    fun `loadUser updates uiState with user data`() = runTest {
        // Given
        fakeRepository.setUser(User("1", "John"))

        // When
        viewModel.loadUser("1")

        // Then
        val state = viewModel.uiState.value
        assertTrue(state is UiState.Success)
        assertEquals("John", (state as UiState.Success).data.name)
    }
}
```

## MainDispatcherRule

```kotlin
class MainDispatcherRule(
    private val dispatcher: TestDispatcher = UnconfinedTestDispatcher()
) : TestWatcher() {

    override fun starting(description: Description) {
        Dispatchers.setMain(dispatcher)
    }

    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```

---

## 관련 문서

[← 테스트](12-테스트/README.md)