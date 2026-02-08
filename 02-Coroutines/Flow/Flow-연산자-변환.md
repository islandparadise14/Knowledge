# Flow 연산자

> Flow 데이터를 변환하고 조합하는 연산자들

## 변환 연산자

### map
```kotlin
flow.map { it * 2 }
```

### filter
```kotlin
flow.filter { it > 0 }
```

### transform
```kotlin
flow.transform { value ->
    emit(value)
    emit(value * 2)
}
```

## 크기 제한

### take
```kotlin
flow.take(3)  // 처음 3개만
```

### drop
```kotlin
flow.drop(2)  // 처음 2개 건너뜀
```

## 조합 연산자

### combine
```kotlin
combine(flow1, flow2) { a, b -> a + b }
// 둘 중 하나라도 새 값 나오면 결합
```

### zip
```kotlin
flow1.zip(flow2) { a, b -> a + b }
// 둘 다 새 값이 있을 때만 결합
```

### merge
```kotlin
merge(flow1, flow2)
// 두 flow를 하나로 합침
```

## 평탄화

### flatMapConcat
```kotlin
flow.flatMapConcat { getDetails(it) }
// 순차 처리
```

### flatMapMerge
```kotlin
flow.flatMapMerge { getDetails(it) }
// 동시 처리
```

### flatMapLatest
```kotlin
flow.flatMapLatest { getDetails(it) }
// 새 값 오면 이전 취소
```

## 버퍼링

### buffer
```kotlin
flow.buffer()  // 생산-소비 분리
```

### conflate
```kotlin
flow.conflate()  // 중간 값 건너뜀
```

### debounce
```kotlin
flow.debounce(300)  // 300ms 대기
```

## 에러 처리

### catch
```kotlin
flow
    .catch { e -> emit(defaultValue) }
    .collect { ... }
```

### retry
```kotlin
flow.retry(3) { e -> e is IOException }
```

---

## 관련 문서
- [[02-Coroutines/Flow/Flow-기초|Flow 기초]]
- [[02-Coroutines/Flow/StateFlow|StateFlow]]

[← Coroutines](02-Coroutines/README.md)
