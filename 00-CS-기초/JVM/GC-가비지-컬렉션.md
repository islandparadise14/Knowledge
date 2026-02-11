---
tags: [cs, jvm, gc, memory, android]
---

# GC (가비지 컬렉션)

## 💡 핵심 개념

**가비지 컬렉션**(GC)은 더 이상 참조되지 않는 객체를 자동으로 메모리에서 해제하는 메커니즘이다. 개발자가 직접 `free()`를 호출할 필요 없지만, GC가 동작하면 앱이 잠시 멈출 수 있다(**Stop-the-World**). Android ART는 이 멈춤을 최소화하기 위해 **Concurrent GC**를 사용한다.

## 📌 왜 필요한가?

GC가 자주 발생하면 UI가 끊기고(jank), 메모리를 잘못 관리하면 OOM이 발생한다. Android Profiler에서 GC 이벤트를 이해하고, 메모리 릭을 방지하려면 GC 동작 원리를 알아야 한다.

## 🔍 자세히

### GC의 기본 원리

```
GC Root에서 참조 체인을 따라가며 "도달 가능한" 객체를 마킹한다.
도달 불가능한 객체 = 가비지 → 메모리 해제

GC Root 종류:
├── Stack의 지역 변수 참조
├── Static 변수 참조
├── JNI 참조
└── 실행 중인 스레드
```

```kotlin
fun example() {
    var user: User? = User("Kim")  // user → GC Root에서 도달 가능
    user = null                     // 참조 끊김 → User("Kim")은 가비지
    // 다음 GC 때 해제됨
}
```

### Heap의 세대별 구분 (Generational GC)

```
┌────────────────────────────────────────┐
│              Heap                       │
│  ┌──────────────┐  ┌────────────────┐  │
│  │  Young Gen   │  │   Old Gen      │  │
│  │  (새 객체)    │  │ (오래 살아남은) │  │
│  │              │  │                │  │
│  │ Eden → S0/S1 │  │                │  │
│  └──────────────┘  └────────────────┘  │
│  Minor GC (자주)    Major GC (가끔)     │
└────────────────────────────────────────┘
```

- **Young Generation:** 새로 생성된 객체. Minor GC가 자주 발생. 대부분 여기서 해제됨.
- **Old Generation:** Young에서 여러 번 살아남은 객체. Major GC는 드물지만 오래 걸림.

### Android ART의 GC

ART는 여러 GC 전략을 상황에 따라 사용한다.

```
1. Concurrent Mark-Sweep (CMS)
   - 앱 실행 중에 백그라운드로 GC
   - Stop-the-World 시간 최소화 (보통 < 5ms)

2. Concurrent Copying (CC) - Android 10+
   - 힙 압축을 동시에 수행
   - 메모리 단편화 해결

3. Explicit GC
   - System.gc() 호출 시 (권장하지 않음)
```

### GC를 유발하는 안티패턴

**1. onDraw()에서 객체 생성**

```kotlin
// ❌ BAD - 매 프레임마다 객체 생성 → GC 폭탄
override fun onDraw(canvas: Canvas) {
    val paint = Paint()  // 60fps면 초당 60개 객체 생성
    paint.color = Color.RED
    canvas.drawCircle(100f, 100f, 50f, paint)
}

// ✅ GOOD - 멤버 변수로 재사용
private val paint = Paint().apply { color = Color.RED }

override fun onDraw(canvas: Canvas) {
    canvas.drawCircle(100f, 100f, 50f, paint)
}
```

**2. 대량 임시 객체 생성**

```kotlin
// ❌ BAD - 문자열 연결 시 매번 새 String 객체
var result = ""
for (i in 0 until 10000) {
    result += "item $i, "  // 매 반복마다 새 String 생성
}

// ✅ GOOD - StringBuilder 사용
val result = buildString {
    for (i in 0 until 10000) {
        append("item $i, ")
    }
}
```

**3. Bitmap 관리 실패**

```kotlin
// 큰 Bitmap은 즉시 recycle
bitmap.recycle()

// 또는 BitmapPool(Glide) 활용으로 재사용
```

### GC 모니터링

```kotlin
// Logcat에서 GC 로그 확인
// D/dalvikvm: GC_CONCURRENT freed 2048K, 65% free 3456K/9856K

// Android Profiler → Memory 탭에서 실시간 확인
// - Allocations: 할당된 객체 수
// - GC Events: GC 발생 시점
// - Heap Dump: 현재 메모리 스냅샷
```

### System.gc()를 쓰면 안 되는 이유

```kotlin
// ❌ 권장하지 않음
System.gc()  // GC를 "요청"할 뿐 즉시 실행 보장 없음
             // 오히려 불필요한 Full GC를 유발할 수 있음
             // ART가 알아서 최적 시점에 GC를 수행함
```

## 🔗 관련 개념

- [[00-CS-기초/JVM/JVM-구조|JVM 구조]]
- [[00-CS-기초/메모리/메모리-릭|메모리 릭]]
- [[00-CS-기초/메모리/스택-vs-힙|스택 vs 힙]]
- [[00-CS-기초/JVM/Dalvik-vs-ART|Dalvik vs ART]]

## 📚 더 보기

- [Android Memory Profiler](https://developer.android.com/studio/profile/memory-profiler)
- [ART GC 개선 사항](https://source.android.com/docs/core/runtime/gc)

---

**핵심 요약:** GC는 참조되지 않는 객체를 자동 해제한다. Android ART는 Concurrent GC로 앱 멈춤을 최소화하지만, onDraw() 안에서 객체 생성이나 대량 임시 객체 등은 GC 부하를 높여 UI jank를 유발한다.
