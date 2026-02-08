# SharedFlow

> 여러 구독자에게 이벤트를 브로드캐스트하는 Hot Flow

## StateFlow와 차이점

| 특성 | StateFlow | SharedFlow |
|------|-----------|------------|
| 초기값 | 필수 | 불필요 |
| 중복 값 | 무시 | 전달 |
| 용도 | 상태 | 이벤트 |

## 기본 사용

```kotlin
class EventViewModel : ViewModel() {
    private val _events = MutableSharedFlow<UiEvent>()
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()

    fun sendEvent(event: UiEvent) {
        viewModelScope.launch {
            _events.emit(event)
        }
    }
}
```

## 설정 옵션

```kotlin
MutableSharedFlow<Int>(
    replay = 0,           // 새 구독자에게 재전송할 값 개수
    extraBufferCapacity = 0,  // 추가 버퍼
    onBufferOverflow = BufferOverflow.SUSPEND  // 버퍼 초과 시 동작
)
```

### onBufferOverflow 옵션
- `SUSPEND`: emit 일시 중단 (기본값)
- `DROP_OLDEST`: 가장 오래된 값 삭제
- `DROP_LATEST`: 최신 값 삭제

## 이벤트 처리 패턴

```kotlin
// Activity/Fragment
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.events.collect { event ->
            when (event) {
                is UiEvent.ShowToast -> showToast(event.message)
                is UiEvent.Navigate -> navigate(event.route)
            }
        }
    }
}
```

## 주의사항

- 구독자 없으면 이벤트 손실 (replay = 0일 때)
- 일회성 이벤트에 적합 (토스트, 네비게이션 등)

---

## 관련 문서
- [[02-Coroutines/Flow/StateFlow|StateFlow]]
- [[02-Coroutines/Flow/Flow-기초|Flow 기초]]

[← Coroutines](02-Coroutines/README.md)
