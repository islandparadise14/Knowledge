# Android 메모리 모델

> Android 앱의 메모리 구조와 관리 방식

## 메모리 영역

### Heap
- 객체가 저장되는 영역
- GC(Garbage Collection) 대상
- 앱마다 제한된 힙 크기 (기기별 상이)

### Stack
- 메서드 호출, 지역 변수
- 스레드마다 별도 스택

### Native Memory
- NDK 코드가 사용
- Bitmap (Android 8.0+)
- GC 대상 아님

## 앱 메모리 제한

```kotlin
val runtime = Runtime.getRuntime()
val maxMemory = runtime.maxMemory() / 1024 / 1024  // MB
val usedMemory = (runtime.totalMemory() - runtime.freeMemory()) / 1024 / 1024
```

## Garbage Collection

### GC 트리거 조건
- 힙 메모리 부족
- 명시적 호출 (권장하지 않음)
- 시스템 메모리 부족

### GC 영향
- GC 중 앱 일시 정지 (Stop-the-world)
- 잦은 GC → 프레임 드랍, 버벅임

## Low Memory Killer

시스템 메모리 부족 시 프로세스 종료 순서:
1. Empty processes
2. Background processes
3. Service processes
4. Visible processes
5. Foreground processes

## onTrimMemory()

```kotlin
override fun onTrimMemory(level: Int) {
    when (level) {
        TRIM_MEMORY_UI_HIDDEN -> {
            // UI가 보이지 않음 - 캐시 해제
        }
        TRIM_MEMORY_RUNNING_LOW -> {
            // 메모리 부족 - 불필요한 리소스 해제
        }
        TRIM_MEMORY_COMPLETE -> {
            // 곧 죽을 수 있음 - 모든 캐시 해제
        }
    }
}
```

## 메모리 최적화 원칙

1. **큰 객체 주의**: Bitmap, 대용량 컬렉션
2. **캐시 크기 제한**: LruCache 사용
3. **Leak 방지**: Context, Listener 참조 관리
4. **적절한 데이터 구조**: SparseArray vs HashMap

---

## 관련 문서
- [[03-Android-Framework/메모리/Memory-Leak|Memory Leak]]
- [[03-Android-Framework/Context/Context란|Context란]]

[← Android Framework](03-Android-Framework/README.md)
