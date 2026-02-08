# Touch Event 기초

> 터치 이벤트의 전달과 처리 방식

## 터치 이벤트 종류

| Action | 설명 |
|--------|------|
| `ACTION_DOWN` | 손가락이 화면에 닿음 |
| `ACTION_MOVE` | 손가락이 이동 |
| `ACTION_UP` | 손가락이 떨어짐 |
| `ACTION_CANCEL` | 이벤트가 취소됨 |

## 이벤트 전달 순서

```
Activity.dispatchTouchEvent()
    ↓
ViewGroup.dispatchTouchEvent()
    ↓
ViewGroup.onInterceptTouchEvent()  // 가로챌지 결정
    ↓
Child View.dispatchTouchEvent()
    ↓
Child View.onTouchEvent()
```

## 핵심 메서드

### dispatchTouchEvent()
```kotlin
override fun dispatchTouchEvent(ev: MotionEvent): Boolean {
    // 이벤트 분배
    return super.dispatchTouchEvent(ev)
}
```

### onInterceptTouchEvent() (ViewGroup만)
```kotlin
override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
    // true 반환 시 자식에게 전달 안 함
    return false
}
```

### onTouchEvent()
```kotlin
override fun onTouchEvent(event: MotionEvent): Boolean {
    when (event.action) {
        MotionEvent.ACTION_DOWN -> {
            // 터치 시작
            return true  // 이벤트 소비
        }
        MotionEvent.ACTION_MOVE -> {
            // 드래그
        }
        MotionEvent.ACTION_UP -> {
            // 터치 끝
        }
    }
    return super.onTouchEvent(event)
}
```

## 반환값의 의미

- `true`: 이벤트를 소비함 (더 이상 전달 안 됨)
- `false`: 이벤트를 소비 안 함 (상위로 전달)

## 주의사항

### ACTION_DOWN에서 true 반환 필수
```kotlin
override fun onTouchEvent(event: MotionEvent): Boolean {
    when (event.action) {
        MotionEvent.ACTION_DOWN -> return true  // 이후 이벤트 받으려면 필수!
    }
    return super.onTouchEvent(event)
}
```

ACTION_DOWN에서 false 반환하면 MOVE, UP 이벤트를 받지 못함.

## 스크롤 충돌 해결

```kotlin
// 부모의 intercept 막기
parent.requestDisallowInterceptTouchEvent(true)
```

---

## 관련 문서
- [[04-View-System/View/Custom-View|Custom View]]
- [[04-View-System/렌더링/View-렌더링-파이프라인|View 렌더링 파이프라인]]

[← View System](04-View-System/README.md)
