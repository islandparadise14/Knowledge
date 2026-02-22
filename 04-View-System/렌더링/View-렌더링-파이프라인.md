---
tags: [android, view, rendering]
---

# View 렌더링 파이프라인

## 💡 핵심 개념

View 렌더링은 **Measure → Layout → Draw** 3단계로 이루어진다. 각 단계는 View 트리를 순회하며 작업한다.

## 🔍 자세히

### 렌더링 단계

**1. Measure Pass**
```kotlin
// View 크기 측정
override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    val width = MeasureSpec.getSize(widthMeasureSpec)
    val mode = MeasureSpec.getMode(widthMeasureSpec)
    // EXACTLY, AT_MOST, UNSPECIFIED
    
    setMeasuredDimension(measuredWidth, measuredHeight)
}
```

**2. Layout Pass**
```kotlin
// View 위치 결정
override fun onLayout(changed: Boolean, left: Int, top: Int, right: Int, bottom: Int) {
    // 자식 View 배치
    child.layout(childLeft, childTop, childRight, childBottom)
}
```

**3. Draw Pass**
```kotlin
override fun onDraw(canvas: Canvas) {
    // 그리기
    canvas.drawRect(...)
    canvas.drawText(...)
}
```

### Invalidate vs RequestLayout

```kotlin
// invalidate - Draw만 다시
view.invalidate()  // onDraw() 호출

// requestLayout - Measure, Layout, Draw 모두
view.requestLayout()  // onMeasure() → onLayout() → onDraw()
```

### 최적화

**1. 불필요한 requestLayout 피하기**
```kotlin
// 나쁨
repeat(100) {
    view.requestLayout()  // 100번 호출!
}

// 좋음
// 한 번만 호출
```

**2. ConstraintLayout 사용**
- Flat hierarchy (중첩 최소화)
- 성능 우수

**3. ViewStub 사용**
```xml
<ViewStub
    android:id="@+id/stub"
    android:layout="@layout/complex_view"/>
```

## 🔗 관련 개념

- [[04-View-System/View/Custom-View]]

---

**핵심 요약:** Measure(크기) → Layout(위치) → Draw(그리기). invalidate는 Draw만, requestLayout은 전체.
