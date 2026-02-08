# withContext

> 코루틴 컨텍스트를 전환하고 결과를 반환하는 suspend 함수

## 기본 사용

```kotlin
suspend fun fetchData(): String {
    return withContext(Dispatchers.IO) {
        // IO 스레드에서 실행
        api.getData()
    }
}
```

## async와 차이점

| 특성 | withContext | async |
|------|-------------|-------|
| 반환 | 직접 결과 | Deferred |
| 대기 | 자동 | await() 필요 |
| 용도 | 컨텍스트 전환 | 병렬 실행 |

```kotlin
// withContext - 순차 실행
suspend fun sequential() {
    val a = withContext(Dispatchers.IO) { fetchA() }
    val b = withContext(Dispatchers.IO) { fetchB() }
    // a 완료 후 b 실행
}

// async - 병렬 실행
suspend fun parallel() = coroutineScope {
    val a = async(Dispatchers.IO) { fetchA() }
    val b = async(Dispatchers.IO) { fetchB() }
    a.await() + b.await()  // 동시 실행
}
```

## 주요 패턴

### Repository 패턴
```kotlin
class UserRepository(
    private val api: UserApi,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) {
    suspend fun getUser(id: String): User {
        return withContext(ioDispatcher) {
            api.getUser(id)
        }
    }
}
```

### Main-safe 함수 만들기
```kotlin
// 어떤 Dispatcher에서 호출해도 안전
suspend fun saveToDatabase(data: Data) {
    withContext(Dispatchers.IO) {
        database.save(data)
    }
}
```

## 주의사항

- `withContext`는 순차 실행 → 병렬 필요하면 `async` 사용
- 같은 Dispatcher로 전환해도 오버헤드 최소화됨

---

## 관련 문서
- [[02-Coroutines/빌더/Launch|launch]]
- [[02-Coroutines/빌더/Async-Await|async/await]]
- [[02-Coroutines/기초/Dispatcher|Dispatcher]]

[← Coroutines](02-Coroutines/README.md)
