# Object와 Companion Object

> Kotlin의 싱글톤과 정적 멤버 구현

## Object 선언

싱글톤을 언어 차원에서 지원:

```kotlin
object DatabaseManager {
    private val connection = createConnection()

    fun query(sql: String): Result {
        return connection.execute(sql)
    }
}

// 사용
DatabaseManager.query("SELECT * FROM users")
```

## Object Expression (익명 객체)

```kotlin
val listener = object : OnClickListener {
    override fun onClick(v: View) {
        // 처리
    }
}
```

## Companion Object

클래스 내부의 싱글톤 (Java의 static과 유사):

```kotlin
class User private constructor(val name: String) {

    companion object {
        const val MAX_NAME_LENGTH = 50

        fun create(name: String): User {
            require(name.length <= MAX_NAME_LENGTH)
            return User(name)
        }
    }
}

// 사용
val user = User.create("John")
println(User.MAX_NAME_LENGTH)
```

## 이름 있는 Companion Object

```kotlin
class MyClass {
    companion object Factory {
        fun create(): MyClass = MyClass()
    }
}

// 사용
MyClass.create()
MyClass.Factory.create()  // 이름으로도 접근 가능
```

## 인터페이스 구현

```kotlin
interface Factory<T> {
    fun create(): T
}

class MyClass {
    companion object : Factory<MyClass> {
        override fun create(): MyClass = MyClass()
    }
}
```

## Java와 호환

```kotlin
class User {
    companion object {
        @JvmStatic  // Java에서 User.create()로 호출 가능
        fun create(): User = User()

        @JvmField  // Java에서 User.MAX_LENGTH로 접근 가능
        val MAX_LENGTH = 50
    }
}
```

## 싱글톤 구현 방식 비교

### object vs companion object + getInstance()

```kotlin
// 방식 1: companion object + getInstance() (Java 스타일)
class DatabaseManager private constructor() {
    companion object {
        private var instance: DatabaseManager? = null

        fun getInstance(): DatabaseManager {
            if (instance == null) {          // 스레드 A, B 동시 통과 가능
                instance = DatabaseManager() // 두 개 생성될 수 있음
            }
            return instance!!
        }
    }
}

// 방식 2: object (Kotlin 스타일)
object DatabaseManager { }  // JVM이 스레드 안전 보장
```

기본 `getInstance()` 구현은 스레드 안전하지 않다. 안전하게 하려면 직접 처리해야 한다.

```kotlin
fun getInstance() = instance ?: synchronized(this) {
    instance ?: DatabaseManager().also { instance = it }
}
```

### companion object + getInstance()를 쓰는 경우

`object`로 해결 안 되는 상황일 때만 쓴다.

```kotlin
// 초기화에 파라미터가 필요할 때
class ApiClient private constructor(val baseUrl: String) {
    companion object {
        private var instance: ApiClient? = null

        fun init(baseUrl: String) {
            instance = ApiClient(baseUrl)
        }

        fun getInstance(): ApiClient {
            return instance ?: error("init() 먼저 호출해야 함")
        }
    }
}

// 테스트에서 인스턴스를 교체해야 할 때
fun setInstance(mock: DatabaseManager) { instance = mock }
```

| | `object` | `companion object + getInstance()` |
|--|--|--|
| 스레드 안전 | JVM이 보장 | 직접 처리해야 함 |
| 코드량 | 최소 | 보일러플레이트 많음 |
| 초기화 파라미터 | 불가 | 가능 |
| 인스턴스 교체 | 불가 | 가능 (테스트 등) |

### LazyHolder는 Kotlin에서 필요한가?

Java에서 LazyHolder 패턴은 언어 차원의 싱글톤 지원이 없어서 만들어진 우회책이다.

```java
// Java LazyHolder
public class DatabaseManager {
    private static class Holder {
        static final DatabaseManager INSTANCE = new DatabaseManager();
    }
    public static DatabaseManager getInstance() {
        return Holder.INSTANCE;
    }
}
```

원리: JVM 클래스 로딩은 스레드 안전하다. `Holder`는 `getInstance()` 첫 호출 시에만 로드되므로 lazy + thread-safe를 달성한다.

Kotlin `object`는 컴파일하면 동일한 메커니즘을 사용한다.

```kotlin
object DatabaseManager { }

// 컴파일 후 Java 코드
public final class DatabaseManager {
    public static final DatabaseManager INSTANCE;
    static {
        INSTANCE = new DatabaseManager();  // 클래스 로딩 시 초기화, JVM이 스레드 안전 보장
    }
}
```

**결론**: LazyHolder와 `object`는 동일한 JVM 클래스 로딩 메커니즘을 사용한다. Kotlin에서 LazyHolder는 불필요한 보일러플레이트다. 단순 싱글톤이면 `object`를 쓰면 된다.

## 주의사항

- `companion object`는 클래스당 하나만 가능
- 실제 객체이므로 메모리 사용
- 늦은 초기화 원하면 `lazy` 사용

---

## 관련 문서
- [[01-Kotlin/클래스-종류/Data-Class|Data Class]]
- [[01-Kotlin/클래스-종류/Sealed-Class|Sealed Class]]

[← Kotlin](../README.md)
