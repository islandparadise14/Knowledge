---
tags: [cs, memory, deep-copy, shallow-copy, kotlin]
---

# 깊은 복사 vs 얕은 복사

## 💡 핵심 개념

**얕은 복사**(Shallow Copy)는 객체의 최상위 필드만 복사하고 내부 참조 객체는 공유한다. **깊은 복사**(Deep Copy)는 내부 참조 객체까지 재귀적으로 모두 새로 생성한다.

## 📌 왜 필요한가?

data class의 `copy()`가 얕은 복사라는 것을 모르면, 복사본 수정 시 원본이 함께 변경되는 버그가 발생한다.

## 🔍 자세히

### 얕은 복사 vs 깊은 복사

```
얕은 복사 (Shallow Copy):
original ──→ [name="Kim", address ──→ Address("Seoul")]
copy     ──→ [name="Kim", address ──┘  (같은 객체 공유!)

깊은 복사 (Deep Copy):
original ──→ [name="Kim", address ──→ Address("Seoul")]
copy     ──→ [name="Kim", address ──→ Address("Seoul")]  (별도 객체)
```

### data class copy()는 얕은 복사

```kotlin
data class Address(var city: String)
data class User(val name: String, val address: Address)

val original = User("Kim", Address("Seoul"))
val copy = original.copy()

copy.address.city = "Busan"
println(original.address.city) // "Busan" ← 원본도 변경됨!
```

### 깊은 복사 구현

```kotlin
// 방법 1: copy()를 중첩 호출
val deepCopy = original.copy(
    address = original.address.copy()
)

// 방법 2: 직렬화/역직렬화 (Kotlin Serialization)
@Serializable
data class User(val name: String, val address: Address)

fun User.deepCopy(): User {
    val json = Json.encodeToString(this)
    return Json.decodeFromString(json)
}
```

### 컬렉션 복사

```kotlin
val original = mutableListOf(mutableListOf(1, 2), mutableListOf(3, 4))

// 얕은 복사 - 내부 리스트는 공유
val shallow = original.toMutableList()
shallow[0].add(99)
println(original[0]) // [1, 2, 99] ← 원본 변경!

// 깊은 복사 - 내부 리스트도 새로 생성
val deep = original.map { it.toMutableList() }.toMutableList()
deep[0].add(100)
println(original[0]) // [1, 2, 99] ← 원본 유지
```

### 불변 객체로 문제 회피

```kotlin
// 불변 객체를 사용하면 얕은 복사도 안전
data class Address(val city: String)  // var → val
data class User(val name: String, val address: Address)

val copy = original.copy()
// copy.address.city = "Busan"  // 컴파일 에러! val이므로 변경 불가
// 새 Address를 만들어야 함 → 원본 영향 없음
val modified = copy.copy(address = Address("Busan"))
```

## 🔗 관련 개념

- [[00-CS-기초/메모리/값-타입-vs-참조-타입|값 타입 vs 참조 타입]]
- [[00-CS-기초/메모리/스택-vs-힙|스택 vs 힙]]

## 📚 더 보기

- [Kotlin data class copy()](https://kotlinlang.org/docs/data-classes.html#copying)

---

**핵심 요약:** data class의 copy()는 얕은 복사(내부 참조 공유). 깊은 복사가 필요하면 중첩 copy() 호출하거나, 애초에 불변 객체(val)를 사용해 문제를 회피한다.
