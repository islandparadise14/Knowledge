# 변성 (Variance) - in, out

> 제네릭 타입의 하위 타입 관계를 정의하는 방법

## 핵심 개념

### 공변성 (Covariance) - out
```kotlin
interface Producer<out T> {
    fun produce(): T  // T를 반환만 가능
}
```
- `out` = 생산자 (Producer)
- 하위 타입 관계 유지: `Producer<Dog>`는 `Producer<Animal>`의 하위 타입

### 반공변성 (Contravariance) - in
```kotlin
interface Consumer<in T> {
    fun consume(item: T)  // T를 소비만 가능
}
```
- `in` = 소비자 (Consumer)
- 하위 타입 관계 역전: `Consumer<Animal>`은 `Consumer<Dog>`의 하위 타입

## 기억법

| 키워드 | 역할 | 위치 | 예시 |
|--------|------|------|------|
| `out` | 생산 | 반환 타입 | `List<out T>` |
| `in` | 소비 | 파라미터 타입 | `Comparable<in T>` |

## 실전 예시

```kotlin
// List는 공변 (out) - 읽기 전용
val dogs: List<Dog> = listOf(Dog())
val animals: List<Animal> = dogs  // OK

// MutableList는 무공변 - 읽기/쓰기 모두
val mutableDogs: MutableList<Dog> = mutableListOf()
// val mutableAnimals: MutableList<Animal> = mutableDogs  // 컴파일 에러
```

## 왜 중요한가?

타입 안전성을 유지하면서 유연한 API 설계 가능.

---

## 관련 문서
- [[01-Kotlin/제네릭/제네릭-기초|제네릭 기초]]

[← Kotlin](01-Kotlin/README.md)
