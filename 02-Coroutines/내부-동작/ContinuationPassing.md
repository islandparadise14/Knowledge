---
tags: [coroutines, continuation, cps, state-machine, suspend, 내부동작]
---

# Continuation Passing (CPS 변환)

## 💡 핵심 개념

Kotlin 컴파일러는 `suspend` 함수를 **Continuation Passing Style(CPS)**로 변환한다. 각 suspend 함수에 `Continuation` 파라미터가 추가되고, 함수 내부는 **상태 머신(State Machine)**으로 변환되어 중단 지점마다 상태를 저장했다가 재개한다. 이것이 "스레드를 블로킹하지 않고 중단"할 수 있는 핵심 원리다.

## 📌 왜 필요한가?

suspend가 "마법"이 아니라 컴파일러 변환임을 이해하면, 코루틴의 성능 특성, 디버깅 시 스택 트레이스 해석, 그리고 `suspendCancellableCoroutine`으로 콜백 API를 감싸는 패턴을 제대로 활용할 수 있다.

## 🔍 자세히

### suspend 함수의 시그니처 변환

```kotlin
// 개발자가 작성한 코드
suspend fun fetchUser(id: String): User

// 컴파일러가 변환한 실제 시그니처 (JVM 바이트코드)
fun fetchUser(id: String, continuation: Continuation<User>): Any?
//                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//                        컴파일러가 추가한 파라미터

// 반환값 Any?인 이유:
// - User를 반환하면 → 즉시 완료
// - COROUTINE_SUSPENDED를 반환하면 → 중단됨, 나중에 continuation.resume()으로 재개
```

### Continuation 인터페이스

```kotlin
public interface Continuation<in T> {
    // 이 코루틴이 실행될 컨텍스트
    public val context: CoroutineContext

    // 결과를 받아 코루틴을 재개
    public fun resumeWith(result: Result<T>)
}

// 확장 함수
continuation.resume(value)        // 성공으로 재개
continuation.resumeWithException(e) // 예외로 재개
```

### 상태 머신 변환

```kotlin
// 원본 코드
suspend fun loadData(): String {
    val user = fetchUser("123")    // 중단 지점 1
    val posts = fetchPosts(user.id) // 중단 지점 2
    return "${user.name}: ${posts.size} posts"
}
```

```kotlin
// 컴파일러가 변환한 코드 (단순화)
fun loadData(continuation: Continuation<String>): Any? {
    // 상태를 저장하는 Continuation 서브클래스
    val sm = continuation as? LoadDataContinuation
        ?: LoadDataContinuation(continuation)

    when (sm.label) {
        0 -> {
            // 초기 상태
            sm.label = 1  // 다음 상태로
            val result = fetchUser("123", sm)  // suspend 호출
            if (result == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
            // 즉시 반환된 경우 fall through
        }
        1 -> {
            // fetchUser 완료 후 재개
            val user = sm.result as User
            sm.user = user  // 지역 변수 저장
            sm.label = 2
            val result = fetchPosts(user.id, sm)
            if (result == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
        }
        2 -> {
            // fetchPosts 완료 후 재개
            val posts = sm.result as List<Post>
            val user = sm.user  // 저장했던 지역 변수 복원
            return "${user.name}: ${posts.size} posts"
        }
    }
}

// 상태를 저장하는 Continuation
class LoadDataContinuation(
    val completion: Continuation<String>
) : Continuation<Any?> {
    var label = 0      // 현재 상태
    var result: Any? = null  // 이전 단계 결과
    var user: User? = null   // 중단 지점 사이의 지역 변수

    override val context = completion.context

    override fun resumeWith(result: Result<Any?>) {
        this.result = result.getOrNull()
        loadData(this)  // 상태 머신 재진입
    }
}
```

### 중단과 재개의 전체 흐름

```
launch(Dispatchers.Main) {
    val user = fetchUser("123")  // ← 여기서 중단
    textView.text = user.name
}

1. launch가 코루틴 시작
2. fetchUser() 호출 → 네트워크 요청 시작
3. COROUTINE_SUSPENDED 반환 → 메인 스레드 해방
4. 메인 스레드는 다른 Message 처리 (터치, 렌더링 등)
   ...
5. 네트워크 응답 도착 (IO 스레드)
6. continuation.resume(user) 호출
7. Dispatchers.Main이 Handler.post()로 메인 스레드에 전달
8. 상태 머신의 label=1 지점부터 재개
9. textView.text = user.name 실행
```

```
시간축:

메인 스레드: [launch 시작][fetchUser 호출][SUSPENDED → 해방!][다른 작업...][재개 → textView 업데이트]
IO 스레드:                                [네트워크 요청 ─────────────── 응답]
                                                                        │
                                                           continuation.resume(user)
```

### 스택 트레이스 이해

```
// 코루틴 예외 발생 시 스택 트레이스
Exception in thread "main"
    at UserRepository.fetchUser(UserRepository.kt:15)
    at UserViewModel$loadUser$1.invokeSuspend(UserViewModel.kt:23)
    //                        ^^^^^^^^^^^^^ 상태 머신의 invoke
    at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(...)
    //                                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //                                   Continuation.resumeWith()으로 재개된 것
    at kotlinx.coroutines.DispatchedTask.run(...)
    at android.os.Handler.handleCallback(...)
    //  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //  Dispatchers.Main → Handler → MessageQueue → 여기서 실행
    at android.os.Looper.loop(...)
    at android.app.ActivityThread.main(...)
```

### 코루틴이 "가벼운" 이유

```
스레드:
- OS 레벨 리소스 (1~2MB 스택 메모리)
- 컨텍스트 스위칭 = 커널 개입
- 10,000개 → 메모리 부족

코루틴:
- Continuation 객체 하나 (수십~수백 바이트)
- 중단 = label 변경 + 지역변수 저장
- 재개 = Runnable 하나 dispatch
- 10,000개 → 문제 없음

// 실제로 확인
repeat(100_000) {
    launch {
        delay(5000)
        println("Done $it")
    }
}
// 100,000개 코루틴이 동시에 동작 → 정상 작동
```

## 💻 실전 예제

### 바이트코드 확인 방법

```
Android Studio에서:
Tools → Kotlin → Show Kotlin Bytecode → Decompile

suspend fun example() {
    delay(1000)
    println("done")
}

// Decompile 결과에서 확인할 수 있는 것:
// 1. Continuation 파라미터 추가
// 2. label 기반 switch 문 (상태 머신)
// 3. COROUTINE_SUSPENDED 반환
// 4. invokeSuspend() 메서드
```

### 상태 머신의 성능 특성

```kotlin
// 각 suspend 지점마다:
// - 1개의 label 증가 (상수 시간)
// - 지역 변수를 Continuation 필드에 저장 (참조 복사)
// - 함수 호출 오버헤드 없음 (같은 함수 내 goto와 유사)

// 따라서 suspend 호출은:
// ✅ 인라인 함수처럼 가벼움
// ✅ 스레드 전환보다 훨씬 저렴
// ⚠️ 다만 Continuation 객체 할당은 있음 (GC 대상)
```

## 🔗 관련 개념

- [[02-Coroutines/기초/Coroutine이란|Coroutine이란]]
- [[02-Coroutines/내부-동작/Dispatchers-내부-구현|Dispatchers 내부 구현]]
- [[02-Coroutines/내부-동작/Handler-Coroutine-브릿지|Handler-Coroutine 브릿지]]
- [[00-CS-기초/JVM/JVM-구조|JVM 구조]]

## 📚 더 보기

- [KotlinConf - Deep Dive into Coroutines](https://www.youtube.com/watch?v=YrrUCSi72E8)
- [Kotlin Coroutines Design Document](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md)
- [BaseContinuationImpl 소스](https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/jvm/src/kotlin/coroutines/jvm/internal/ContinuationImpl.kt)

---

**핵심 요약:** suspend 함수는 컴파일러가 Continuation 파라미터를 추가하고 상태 머신으로 변환한다. 중단 시 COROUTINE_SUSPENDED를 반환해 스레드를 해방하고, 재개 시 continuation.resume()으로 상태 머신에 재진입한다.
