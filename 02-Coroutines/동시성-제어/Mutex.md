# Mutex

> 코루틴에서 공유 자원의 동시 접근을 제어하는 동기화 도구

## 왜 필요한가?

```kotlin
var counter = 0

// 문제: Race Condition
coroutineScope {
    repeat(1000) {
        launch {
            counter++  // 동시 접근 → 잘못된 결과
        }
    }
}
println(counter)  // 1000이 아닐 수 있음
```

## Mutex 사용

```kotlin
val mutex = Mutex()
var counter = 0

coroutineScope {
    repeat(1000) {
        launch {
            mutex.withLock {
                counter++  // 한 번에 하나의 코루틴만 접근
            }
        }
    }
}
println(counter)  // 항상 1000
```

## synchronized와 차이

| 특성 | synchronized | Mutex |
|------|--------------|-------|
| 블로킹 | 스레드 블로킹 | suspend (비블로킹) |
| 사용처 | 일반 코드 | 코루틴 |
| 성능 | 스레드 낭비 | 효율적 |

```kotlin
// 나쁜 예: 코루틴에서 synchronized
synchronized(lock) {
    delay(100)  // 컴파일 에러 - suspend 불가
}

// 좋은 예: Mutex
mutex.withLock {
    delay(100)  // OK
}
```

## 주의사항

### 데드락 방지
```kotlin
// 위험: 같은 Mutex를 중첩 lock
mutex.withLock {
    mutex.withLock {  // 데드락!
        // ...
    }
}
```

### 대안: Atomic 클래스
```kotlin
// 단순 카운터는 AtomicInteger가 더 효율적
val counter = AtomicInteger(0)

launch {
    counter.incrementAndGet()
}
```

## 다른 동시성 도구

- `Semaphore`: 동시 접근 수 제한
- `Channel`: 코루틴 간 통신
- `StateFlow`: 상태 공유

---

## 관련 문서
- [[02-Coroutines/기초/CoroutineContext|CoroutineContext]]
- [[02-Coroutines/구조화된-동시성/Structured-Concurrency|Structured Concurrency]]

[← Coroutines](02-Coroutines/README.md)
