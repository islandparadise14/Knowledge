---
tags: [cs, memory, stack, heap, android]
---

# 스택 vs 힙

## 💡 핵심 개념

**스택**(Stack)은 메서드 호출 시 지역 변수와 프레임이 쌓이는 LIFO 구조의 메모리이고, **힙**(Heap)은 객체 인스턴스가 동적으로 할당되는 공유 메모리이다. 스택은 스레드마다 독립적이고, 힙은 모든 스레드가 공유한다.

## 📌 왜 필요한가?

StackOverflowError, OutOfMemoryError의 원인을 이해하고, 메모리 릭을 방지하려면 객체가 어디에 저장되는지 알아야 한다.

## 🔍 자세히

### 메모리 구조

```
┌─────────────────────────────┐
│          Heap (공유)         │
│  ┌───────────┐ ┌──────────┐│
│  │ User 객체  │ │ List 객체 ││
│  └───────────┘ └──────────┘│
│         GC가 관리            │
├──────────┬──────────────────┤
│ Stack    │ Stack            │
│ (Main)   │ (Worker)        │
│ ┌──────┐ │ ┌──────┐       │
│ │frame3│ │ │frame1│       │
│ │frame2│ │ └──────┘       │
│ │frame1│ │                 │
│ └──────┘ │                 │
└──────────┴──────────────────┘
```

### 스택 vs 힙 비교

| 특성 | 스택 | 힙 |
|---|---|---|
| 저장 내용 | 지역 변수, 참조, 프레임 | 객체 인스턴스 |
| 스레드 공유 | X (스레드 독립) | O (공유) |
| 할당 속도 | **빠름** (포인터 이동) | 느림 (GC 관리) |
| 해제 | 자동 (프레임 pop) | GC가 수거 |
| 크기 제한 | 작음 (보통 1~8MB) | 큼 (앱당 수백 MB) |
| 에러 | StackOverflowError | OutOfMemoryError |

### Kotlin에서의 동작

```kotlin
fun processUser() {
    val id = 42                    // id(Int) → Stack
    val name = "Kim"               // name(참조) → Stack, String 객체 → Heap
    val user = User(id, name)      // user(참조) → Stack, User 객체 → Heap
    val list = listOf(user)        // list(참조) → Stack, List 객체 → Heap
}
// 메서드 종료 → Stack의 id, name, user, list 참조 모두 제거
// Heap의 User, List 객체 → 참조 없으면 GC 대상
```

### StackOverflowError

```kotlin
// ❌ 무한 재귀 → 스택 프레임이 계속 쌓여 오버플로
fun infinite(): Int = 1 + infinite()

// ✅ 꼬리 재귀 최적화 (tailrec) → 스택 프레임 재사용
tailrec fun factorial(n: Int, acc: Long = 1): Long {
    return if (n <= 1) acc else factorial(n - 1, n * acc)
}
```

### OutOfMemoryError

```kotlin
// ❌ Heap 초과
val hugeList = mutableListOf<ByteArray>()
while (true) {
    hugeList.add(ByteArray(1024 * 1024)) // 1MB씩 계속 할당
}

// Android에서 Heap 제한 확인
val maxHeap = Runtime.getRuntime().maxMemory() / 1024 / 1024
Log.d("Memory", "Max heap: ${maxHeap}MB")
```

### Android에서의 스레드 스택 크기

```kotlin
// 기본 스레드 스택: 약 1MB
// 메인 스레드 스택: 약 8MB

// 커스텀 스택 크기 스레드
val thread = Thread(null, {
    // 깊은 재귀가 필요한 작업
}, "custom-thread", 2 * 1024 * 1024) // 2MB 스택
thread.start()
```

## 🔗 관련 개념

- [[00-CS-기초/JVM/JVM-구조|JVM 구조]]
- [[00-CS-기초/JVM/GC-가비지-컬렉션|GC 가비지 컬렉션]]
- [[00-CS-기초/메모리/메모리-릭|메모리 릭]]
- [[00-CS-기초/메모리/값-타입-vs-참조-타입|값 타입 vs 참조 타입]]

## 📚 더 보기

- [Android Memory Management](https://developer.android.com/topic/performance/memory-overview)

---

**핵심 요약:** 스택은 스레드별 지역 변수/프레임 저장(자동 해제), 힙은 공유 객체 저장(GC 해제). Android 앱은 힙 크기가 제한되어 OOM에 주의해야 한다.
