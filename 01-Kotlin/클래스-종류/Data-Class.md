---
tags: [kotlin, data-class]
---

# Data Class

## 💡 핵심 개념

**Data class**는 데이터를 담는 용도의 클래스. 컴파일러가 자동으로 `equals()`, `hashCode()`, `toString()`, `copy()`, `componentN()` 함수를 생성한다.

## 📌 왜 필요한가?

Java에서 POJO를 만들려면 보일러플레이트 코드가 너무 많다. Kotlin은 `data` 키워드 하나로 이를 해결한다. 특히 불변 데이터 모델을 만들 때 필수다.

## 🔍 자세히

### 기본 사용법

```kotlin
// 기본 data class
data class User(
    val id: String,
    val name: String,
    val age: Int
)

// 자동 생성되는 것들
val user1 = User("1", "민트", 27)
val user2 = User("1", "민트", 27)

// equals() & hashCode()
println(user1 == user2)  // true
println(user1 === user2)  // false (다른 인스턴스)

// toString()
println(user1)  // User(id=1, name=민트, age=27)

// copy()
val user3 = user1.copy(age = 28)

// componentN() - 구조 분해
val (id, name, age) = user1
```

### Data Class 요구사항

```kotlin
// 주 생성자에 최소 1개 이상의 파라미터 필요
// data class Empty()  // ERROR

// 주 생성자의 모든 파라미터는 val 또는 var
data class User(val name: String, var age: Int)

// abstract, open, sealed, inner 불가
// abstract data class User(val name: String)  // ERROR
```

### copy() 함수

```kotlin
data class User(
    val name: String,
    val age: Int,
    val email: String
)

val user = User("민트", 27, "mint@example.com")

// 일부만 변경
val updated = user.copy(age = 28)

// 여러 개 변경
val updated2 = user.copy(
    age = 28,
    email = "newemail@example.com"
)
```

### 구조 분해 (Destructuring)

```kotlin
data class User(val name: String, val age: Int, val city: String)

val user = User("민트", 27, "Seoul")

// 전체 분해
val (name, age, city) = user

// 일부만
val (name, age) = user  // city는 무시

// 특정 요소만
val (_, age) = user  // name 무시
```

### 동작 원리

**컴파일러가 생성하는 코드**

```kotlin
data class User(val name: String, val age: Int)

// 컴파일 후 (대략적인 Java 코드)
public final class User {
    private final String name;
    private final int age;
    
    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() { return name; }
    public int getAge() { return age; }
    
    // equals()
    public boolean equals(Object other) {
        if (this == other) return true;
        if (!(other instanceof User)) return false;
        User user = (User) other;
        return Objects.equals(name, user.name) && age == user.age;
    }
    
    // hashCode()
    public int hashCode() {
        return Objects.hash(name, age);
    }
    
    // toString()
    public String toString() {
        return "User(name=" + name + ", age=" + age + ")";
    }
    
    // copy()
    public User copy(String name, int age) {
        return new User(name, age);
    }
    
    // componentN()
    public String component1() { return name; }
    public int component2() { return age; }
}
```

### 주의사항

**1. 주 생성자에만 포함되는 프로퍼티가 고려됨**

```kotlin
data class User(val name: String) {
    var age: Int = 0  // equals, hashCode, toString에서 제외!
}

val user1 = User("민트").apply { age = 27 }
val user2 = User("민트").apply { age = 30 }

println(user1 == user2)  // true! (age는 비교 안 됨)
```

**2. 가변 프로퍼티 주의**

```kotlin
data class User(var name: String)

val user = User("민트")
val set = mutableSetOf(user)

user.name = "민트2"  // hashCode 변경!
println(set.contains(user))  // false! (찾을 수 없음)
```

**해결책:** data class는 불변(val)으로 만들기

**3. Collection에서 사용 시**

```kotlin
// OK
data class User(val id: String, val name: String)

val users = listOf(
    User("1", "민트"),
    User("2", "민트2")
).associateBy { it.id }

// 주의: mutable 프로퍼티
data class BadUser(var id: String, val name: String)
// Map의 키로 사용하면 위험
```

**4. 상속**

```kotlin
// data class는 다른 클래스 상속 가능
open class Person(val name: String)
data class User(val id: String, name: String) : Person(name)

// 하지만 data class를 상속은 불가
// class Developer : User("", "")  // ERROR
```

## 💻 실전 예제

### API Response 모델

```kotlin
data class ApiResponse<T>(
    val success: Boolean,
    val data: T?,
    val message: String?
)

data class User(
    val id: String,
    val name: String,
    val email: String,
    val createdAt: String
)

// 사용
val response: ApiResponse<User> = api.getUser("123")
```

### Domain 모델

```kotlin
// Entity (DB)
@Entity
data class UserEntity(
    @PrimaryKey val id: String,
    val name: String,
    val age: Int
)

// Domain Model
data class User(
    val id: String,
    val name: String,
    val age: Int
)

// DTO (Network)
data class UserDto(
    val id: String,
    val name: String,
    val age: Int
)

// Mapper
fun UserDto.toDomain() = User(id, name, age)
fun User.toEntity() = UserEntity(id, name, age)
```

### UI State

```kotlin
data class UserUiState(
    val user: User? = null,
    val isLoading: Boolean = false,
    val error: String? = null
)

class UserViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()
    
    fun loadUser(id: String) {
        _uiState.update { it.copy(isLoading = true) }
        viewModelScope.launch {
            try {
                val user = repository.getUser(id)
                _uiState.update { 
                    it.copy(user = user, isLoading = false)
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

### 불변 리스트 업데이트

```kotlin
data class TodoList(val items: List<Todo> = emptyList())

data class Todo(
    val id: String,
    val title: String,
    val completed: Boolean = false
)

fun TodoList.addTodo(todo: Todo) = copy(
    items = items + todo
)

fun TodoList.toggleTodo(id: String) = copy(
    items = items.map { 
        if (it.id == id) it.copy(completed = !it.completed) 
        else it 
    }
)

fun TodoList.removeTodo(id: String) = copy(
    items = items.filter { it.id != id }
)
```

## 🔗 관련 개념

- [[01-Kotlin/기본문법/클래스와-상속]]
- [[01-Kotlin/클래스-종류/Sealed-Class]]
- [[06-Architecture/상태-관리/UiState]]

## 📚 더 보기

- [Kotlin Docs - Data Classes](https://kotlinlang.org/docs/data-classes.html)

---

**핵심 요약:** data class는 불변으로 만들 것. copy()로 수정. 주 생성자 파라미터만 equals/hashCode에 포함.
