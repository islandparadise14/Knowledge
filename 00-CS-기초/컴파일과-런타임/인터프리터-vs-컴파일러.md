---
tags: [CS, 컴파일러, 인터프리터, JVM, ART, Kotlin]
---

# 인터프리터 vs 컴파일러

## 💡 핵심 개념

컴파일러(Compiler)는 소스 코드 전체를 한번에 기계어 또는 중간 코드로 변환하는 프로그램이고, 인터프리터(Interpreter)는 소스 코드를 한 줄씩 읽어 즉시 실행하는 프로그램이다. 현대 언어들은 두 방식을 혼합하여 사용하며, Kotlin/Android는 컴파일러(kotlinc) + JIT/AOT 컴파일의 하이브리드 방식을 채택한다.

## 📌 왜 필요한가?

안드로이드 앱의 빌드와 실행 과정을 이해하려면 컴파일러와 인터프리터의 차이를 알아야 한다. Kotlin 코드가 `.class` → `.dex` → 네이티브 코드로 변환되는 과정, ART의 AOT/JIT 혼합 전략이 모두 이 개념에 기반한다.

## 🔍 자세히

### 비교 표

| 특성 | 컴파일러 | 인터프리터 |
|------|---------|-----------|
| 변환 단위 | 전체 소스 코드 | 한 줄 / 한 구문 |
| 실행 속도 | 빠름 (미리 변환) | 느림 (실시간 해석) |
| 에러 감지 | 컴파일 시 전체 검사 | 실행 중 해당 줄 도달 시 |
| 출력물 | 실행 파일 / 바이트코드 | 없음 (즉시 실행) |
| 디버깅 | 컴파일 후 실행 필요 | 즉시 결과 확인 |
| 대표 언어 | C, C++, Kotlin, Java | Python, Ruby, JavaScript* |

*JavaScript는 현대 엔진(V8)에서 JIT 컴파일을 사용한다.

### 하이브리드 접근: Kotlin → Android 실행 경로

```
[컴파일 타임 - 개발자 머신]
Kotlin 소스 (.kt)
    ↓ kotlinc (Kotlin 컴파일러)
Java 바이트코드 (.class)
    ↓ D8 / R8 (Android 전용 컴파일러)
DEX 바이트코드 (.dex)
    ↓ 패키징
APK / AAB

[설치 시 - 사용자 기기]
APK 설치
    ↓ ART (Android Runtime)
AOT 컴파일: .dex → .oat (네이티브 코드)  ← 일부 미리 컴파일

[런타임 - 앱 실행]
.oat 네이티브 코드 직접 실행 (AOT 컴파일된 부분)
    + JIT 컴파일 (자주 사용되는 hot path를 추가 최적화)
    + 인터프리터 (드물게 실행되는 cold path)
```

### AOT vs JIT 컴파일

```
AOT (Ahead-Of-Time):
- 설치 시 또는 유휴 시간에 미리 컴파일
- 장점: 빠른 시작, 안정적 성능
- 단점: 설치 시간 증가, 저장 공간 사용

JIT (Just-In-Time):
- 실행 중 핫스팟을 감지하여 실시간 컴파일
- 장점: 런타임 프로파일 기반 최적화, 적은 저장 공간
- 단점: 초기 실행 느림, 메모리 사용

ART의 Profile-Guided Compilation:
- JIT가 프로파일 수집 → 유휴 시 AOT가 핫스팟 컴파일
- 두 방식의 장점을 조합
```

### 빌드 시간에 영향을 주는 요소

```kotlin
// 1. KAPT (Annotation Processing) - 컴파일 타임에 코드 생성
// kapt는 Java 스텁을 생성해야 하므로 느림
@Entity(tableName = "users")
data class User(
    @PrimaryKey val id: String,
    val name: String
)
// → Room이 컴파일 타임에 UserDao_Impl.java 생성

// 2. KSP - KAPT보다 빠른 대안
// KSP는 Kotlin 컴파일러 플러그인으로 직접 동작
// kapt → ksp 마이그레이션으로 빌드 시간 단축

// build.gradle.kts
plugins {
    id("com.google.devtools.ksp") version "2.0.0-1.0.21"
}
dependencies {
    ksp("androidx.room:room-compiler:2.6.1") // kapt 대신 ksp
}
```

### 인터프리터 방식이 사용되는 안드로이드 영역

```kotlin
// 1. Gradle 빌드 스크립트 (Groovy는 인터프리터 성격)
// build.gradle - Groovy 스크립트로 빌드 설정

// 2. WebView에서 JavaScript 실행
webView.evaluateJavascript(
    "document.getElementById('title').innerText",
    { result -> Log.d("JS", "Result: $result") }
)

// 3. 정규표현식 엔진 (런타임 해석)
val regex = Regex("""\d{3}-\d{4}-\d{4}""")
val isPhone = regex.matches("010-1234-5678") // 패턴을 런타임에 해석

// 4. Kotlin Scripting (.kts)
// Gradle KTS는 Kotlin Script → 컴파일 후 실행
```

### R8 최적화 (컴파일 타임 최적화)

```kotlin
// R8 컴파일러가 수행하는 최적화
// 1. 코드 축소 (Tree Shaking) - 미사용 코드 제거
// 2. 난독화 (Obfuscation) - 클래스/메서드명 변경
// 3. 최적화 (Optimization) - 인라이닝, 상수 폴딩 등

// proguard-rules.pro
// -keep class com.myapp.data.model.** { *; } // 직렬화 모델 보존
// -keepclassmembers class * implements android.os.Parcelable {
//     static ** CREATOR;
// }
```

### Kotlin 컴파일러 플러그인

```kotlin
// Compose 컴파일러 플러그인 - 컴파일 타임에 UI 코드 변환
@Composable
fun Greeting(name: String) {
    Text("Hello, $name")
}
// → Compose 컴파일러가 $composer, $changed 파라미터를 추가하고
//   스마트 리컴포지션을 위한 코드를 삽입

// Kotlin Serialization 컴파일러 플러그인
@Serializable
data class User(val name: String, val age: Int)
// → 컴파일러 플러그인이 serializer() 메서드를 자동 생성
```

## 🔗 관련 개념

- [[00-CS-기초/컴파일과-런타임/컴파일-과정|컴파일 과정]]
- [[00-CS-기초/컴파일과-런타임/컴파일-타임-vs-런타임|컴파일 타임 vs 런타임]]
- [[00-CS-기초/컴파일과-런타임/바이트코드와-기계어|바이트코드와 기계어]]
- [[00-CS-기초/컴파일과-런타임/kapt-vs-ksp|KAPT vs KSP]]
- [[00-CS-기초/JVM/JIT-컴파일|JIT 컴파일]]
- [[00-CS-기초/JVM/Dalvik-vs-ART|Dalvik vs ART]]

## 📚 더 보기

- [Android ART 공식 문서](https://source.android.com/docs/core/runtime)
- [Kotlin 컴파일러 플러그인](https://kotlinlang.org/docs/ksp-overview.html)
- [R8 최적화](https://developer.android.com/build/shrink-code)

---

**핵심 요약:** 컴파일러는 전체 코드를 미리 변환하고 인터프리터는 즉시 실행하며, Android/Kotlin은 kotlinc 컴파일 → D8/R8 변환 → ART의 AOT+JIT 혼합 실행이라는 다단계 하이브리드 방식을 사용한다.
