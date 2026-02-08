---
tags: [android, view-system, custom-view]
---

# Custom View

## 💡 핵심 개념

**Custom View**는 기본 제공되는 View로 구현할 수 없는 UI를 직접 만드는 것이다. Canvas API를 사용해 직접 그린다.

## 📌 왜 필요한가?

차트, 게이지, 특수한 애니메이션, 게임 UI 등 표준 View로 불가능한 것을 구현할 때 필요하다.

## 🔍 자세히

### 기본 구조

```kotlin
class MyCustomView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
    
    private val paint = Paint(Paint.ANTI_ALIAS_FLAG)
    
    init {
        // 속성 읽기
        context.obtainStyledAttributes(attrs, R.styleable.MyCustomView).apply {
            val color = getColor(R.styleable.MyCustomView_customColor, Color.BLACK)
            paint.color = color
            recycle()
        }
    }
    
    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        // 크기 결정
        val desiredWidth = 200.dpToPx()
        val desiredHeight = 200.dpToPx()
        
        val width = resolveSize(desiredWidth, widthMeasureSpec)
        val height = resolveSize(desiredHeight, heightMeasureSpec)
        
        setMeasuredDimension(width, height)
    }
    
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        // 그리기
        canvas.drawCircle(
            width / 2f,
            height / 2f,
            min(width, height) / 2f,
            paint
        )
    }
}
```

### Canvas 그리기

```kotlin
override fun onDraw(canvas: Canvas) {
    // 도형
    canvas.drawRect(left, top, right, bottom, paint)
    canvas.drawCircle(cx, cy, radius, paint)
    canvas.drawLine(startX, startY, endX, endY, paint)
    canvas.drawRoundRect(rect, rx, ry, paint)
    
    // 텍스트
    canvas.drawText("Hello", x, y, paint)
    
    // Path
    val path = Path().apply {
        moveTo(x1, y1)
        lineTo(x2, y2)
        cubicTo(x3, y3, x4, y4, x5, y5)
    }
    canvas.drawPath(path, paint)
    
    // 비트맵
    canvas.drawBitmap(bitmap, x, y, paint)
}
```

### Paint 설정

```kotlin
private val paint = Paint().apply {
    // 안티앨리어싱
    isAntiAlias = true
    
    // 스타일
    style = Paint.Style.FILL  // FILL, STROKE, FILL_AND_STROKE
    
    // 색상
    color = Color.RED
    
    // 두께
    strokeWidth = 5f
    
    // 텍스트
    textSize = 20.sp
    textAlign = Paint.Align.CENTER
}
```

### 터치 이벤트

```kotlin
override fun onTouchEvent(event: MotionEvent): Boolean {
    when (event.action) {
        MotionEvent.ACTION_DOWN -> {
            // 터치 시작
            touchX = event.x
            touchY = event.y
            return true
        }
        MotionEvent.ACTION_MOVE -> {
            // 드래그
            val dx = event.x - touchX
            val dy = event.y - touchY
            invalidate()
            return true
        }
        MotionEvent.ACTION_UP -> {
            // 터치 종료
            performClick()
            return true
        }
    }
    return super.onTouchEvent(event)
}

override fun performClick(): Boolean {
    super.performClick()
    // 클릭 처리
    return true
}
```

### 속성 정의

```xml
<!-- attrs.xml -->
<declare-styleable name="MyCustomView">
    <attr name="customColor" format="color"/>
    <attr name="customSize" format="dimension"/>
    <attr name="customText" format="string"/>
</declare-styleable>
```

```kotlin
// 속성 읽기
init {
    context.obtainStyledAttributes(attrs, R.styleable.MyCustomView).apply {
        customColor = getColor(R.styleable.MyCustomView_customColor, Color.BLACK)
        customSize = getDimension(R.styleable.MyCustomView_customSize, 100f)
        customText = getString(R.styleable.MyCustomView_customText) ?: ""
        recycle()
    }
}
```

### 상태 저장

```kotlin
override fun onSaveInstanceState(): Parcelable {
    val superState = super.onSaveInstanceState()
    val savedState = SavedState(superState)
    savedState.currentValue = currentValue
    return savedState
}

override fun onRestoreInstanceState(state: Parcelable?) {
    val savedState = state as SavedState
    super.onRestoreInstanceState(savedState.superState)
    currentValue = savedState.currentValue
}

private class SavedState : BaseSavedState {
    var currentValue: Int = 0
    
    constructor(superState: Parcelable?) : super(superState)
    
    constructor(source: Parcel) : super(source) {
        currentValue = source.readInt()
    }
    
    override fun writeToParcel(out: Parcel, flags: Int) {
        super.writeToParcel(out, flags)
        out.writeInt(currentValue)
    }
    
    companion object {
        @JvmField
        val CREATOR = object : Parcelable.Creator<SavedState> {
            override fun createFromParcel(source: Parcel) = SavedState(source)
            override fun newArray(size: Int) = arrayOfNulls<SavedState>(size)
        }
    }
}
```

## 💻 실전 예제

### 프로그레스 바

```kotlin
class CircularProgressView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : View(context, attrs) {
    
    var progress: Float = 0f
        set(value) {
            field = value.coerceIn(0f, 100f)
            invalidate()
        }
    
    private val progressPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeWidth = 10.dpToPx().toFloat()
        color = Color.BLUE
        strokeCap = Paint.Cap.ROUND
    }
    
    private val backgroundPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeWidth = 10.dpToPx().toFloat()
        color = Color.LIGHT_GRAY
    }
    
    private val rect = RectF()
    
    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        val padding = 20f
        rect.set(padding, padding, w - padding, h - padding)
    }
    
    override fun onDraw(canvas: Canvas) {
        // 배경 원
        canvas.drawArc(rect, 0f, 360f, false, backgroundPaint)
        
        // 진행률
        val sweepAngle = 360f * progress / 100f
        canvas.drawArc(rect, -90f, sweepAngle, false, progressPaint)
        
        // 텍스트
        val text = "${progress.toInt()}%"
        val textPaint = Paint().apply {
            textSize = 40.sp
            textAlign = Paint.Align.CENTER
        }
        canvas.drawText(
            text,
            width / 2f,
            height / 2f + textPaint.textSize / 3,
            textPaint
        )
    }
}
```

### 차트 View

```kotlin
class LineChartView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : View(context, attrs) {
    
    var dataPoints: List<Float> = emptyList()
        set(value) {
            field = value
            invalidate()
        }
    
    private val linePaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeWidth = 4f
        color = Color.BLUE
    }
    
    private val path = Path()
    
    override fun onDraw(canvas: Canvas) {
        if (dataPoints.isEmpty()) return
        
        val maxValue = dataPoints.maxOrNull() ?: 1f
        val xStep = width.toFloat() / (dataPoints.size - 1)
        
        path.reset()
        dataPoints.forEachIndexed { index, value ->
            val x = index * xStep
            val y = height - (value / maxValue * height)
            
            if (index == 0) {
                path.moveTo(x, y)
            } else {
                path.lineTo(x, y)
            }
        }
        
        canvas.drawPath(path, linePaint)
    }
}
```

## 🔗 관련 개념

- [[04-View-System/렌더링/View-렌더링-파이프라인]]
- [[04-View-System/애니메이션/Property-Animation]]

## 📚 더 보기

- [Android Docs - Custom Views](https://developer.android.com/develop/ui/views/layout/custom-views/custom-components)

---

**핵심 요약:** onMeasure로 크기, onDraw로 그리기. Paint + Canvas 사용. 상태 변경 시 invalidate() 호출.
