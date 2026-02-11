---
tags: [cs, memory, value-type, reference-type, kotlin]
---

# 값 타입 vs 참조 타입

## 💡 핵심 개념

**값 타입**은 변수에 실제 값이 저장되고, **참조 타입**은 변수에 객체의 메모리 주소(참조)가 저장된다. Kotlin/JVM에서 `Int`, `Boolean` 등 원시 타입은 값으로, 클래스 인스턴스는 참조로 다뤄진다.

## 📌 왜 필요한가?

`==`와 `===`의 차이, 함수 인자 전달 시 동작, 불변성 설계를 이해하는 기반이 된다.

## 🔍 자세히

### Kotlin의 타입 체계

```kotlin
// Kotlin에서 모든 것은 객체지만, 컴파일 시 원시 타입으로 최적화됨

val a: Int = 42          // 컴파일 → JVM int (값 타입, Stack)
val b: Int? = 42         // 컴파일 → JVM Integer (참조 타입, Heap에 박싱)
val c: String = "hello"  // 항상 참조 타입 (Heap)
```

### 동등성 비교

```kotlin
val x = 1000
val y = 1000
println(x == y)   // true  (구조적 동등성 - 값 비교)
println(x === y)  // true  (컴파일러가 int로 최적화)

val a: Int? = 1000
val b: Int? = 1000
println(a == b)   // true  (값이 같음)
println(a === b)  // false (서로 다른 Integer 객체)

// -128 ~ 127은 Integer 캐시로 같은 객체
val p: Int? = 100
val q: Int? = 100
println(p === q)  // true (캐시된 같은 객체)
```

### 함수 인자 전달

```kotlin
// JVM은 항상 "값에 의한 전달" (Call by Value)
// 원시 타입: 값 복사
// 참조 타입: 참조(주소) 복사 → 같은 객체를 가리킴

fun modifyValue(n: Int) {
    // n은 복사본 → 원본 영향 없음
}

fun modifyObject(list: MutableList<Int>) {
    list.add(99)  // 참조 복사이므로 원본 리스트가 변경됨
}

val myList = mutableListOf(1, 2, 3)
modifyObject(myList)
println(myList) // [1, 2, 3, 99]
```

### data class와 동등성

```kotlin
data class User(val name: String, val age: Int)

val u1 = User("Kim", 25)
val u2 = User("Kim", 25)
println(u1 == u2)  // true  (data class는 equals()를 자동 생성)
println(u1 === u2) // false (서로 다른 객체)
```

### Android에서의 주의점

```kotlin
// Bundle로 데이터 전달 시 직렬화/역직렬화 → 다른 객체
val user = User("Kim", 25)
bundle.putParcelable("user", user)
val restored = bundle.getParcelable<User>("user")
println(user === restored) // false (다른 객체)
println(user == restored)  // true  (값이 같음)
```

## 🔗 관련 개념

- [[00-CS-기초/메모리/스택-vs-힙|스택 vs 힙]]
- [[00-CS-기초/메모리/깊은-복사-vs-얕은-복사|깊은 복사 vs 얕은 복사]]
- [[00-CS-기초/컴파일과-런타임/타입-시스템|타입 시스템]]

## 📚 더 보기

- [Kotlin - Equality](https://kotlinlang.org/docs/equality.html)

---

**핵심 요약:** Kotlin에서 원시 타입은 값으로, 객체는 참조로 저장된다. `==`는 값 비교, `===`는 참조(동일 객체) 비교. 함수 인자는 항상 값(또는 참조의 복사)으로 전달된다.
