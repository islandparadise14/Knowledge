# 테스트

> Android 앱 테스트의 모든 것

## 📚 목차

### [[12-테스트/Unit-Test/JUnit|Unit Test]]
- [[12-테스트/Unit-Test/JUnit|JUnit]]
- Mockk
- Coroutine 테스트

### Integration Test
- Repository 테스트
- Room 테스트

### UI Test
- Espresso
- Compose Test

### 테스트 더블
- Mock vs Fake vs Stub

### 베스트 프랙티스
- 테스트 작성 원칙

---

## 테스트 피라미드

```
      /\
     /UI\        적음, 느림
    /----\
   /Integ-\
  / ration \
 /----------\
/  Unit Test \   많음, 빠름
--------------
```

## 테스트 위치

```
app/
├── src/
│   ├── main/           # 프로덕션 코드
│   ├── test/           # Unit Test (JVM)
│   └── androidTest/    # Instrumented Test (기기/에뮬)
```

---

## 관련 문서

[← 메인으로](README.md)