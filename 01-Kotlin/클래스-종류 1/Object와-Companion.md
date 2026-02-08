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

## 주의사항

- `companion object`는 클래스당 하나만 가능
- 실제 객체이므로 메모리 사용
- 늦은 초기화 원하면 `lazy` 사용

---

## 관련 문서
- [[01-Kotlin/클래스-종류/Data-Class|Data Class]]
- [[01-Kotlin/클래스-종류/Sealed-Class|Sealed Class]]

[← Kotlin](../README.md)
