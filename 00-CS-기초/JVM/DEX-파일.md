---
tags: [cs, jvm, dex, android, dalvik]
---

# DEX 파일

## 💡 핵심 개념

**DEX**(Dalvik Executable)는 Android 런타임(ART/Dalvik)이 실행하는 바이트코드 형식이다. Java의 `.class`와 다른 포맷으로, 모바일 환경에 최적화되어 있다. `.kt/.java` → `.class` → **`.dex`** → `.apk` 순으로 변환된다.

## 📌 왜 필요한가?

64K 메서드 제한, APK 크기 최적화, R8/ProGuard 설정, MultiDex 이슈 등 실무에서 자주 만나는 문제가 모두 DEX와 관련된다.

## 🔍 자세히

### .class vs .dex

```
.class (JVM)                    .dex (Android)
────────────────────────────────────────────
클래스 하나당 파일 하나            여러 클래스가 하나의 DEX에
스택 기반 바이트코드              레지스터 기반 바이트코드
상수 풀이 클래스마다 중복          공유 상수 풀 (중복 제거)
→ 파일 크기 큼                   → 파일 크기 작음
```

### DEX 변환 과정

```
.kt / .java
     │  kotlinc / javac
     ▼
.class (JVM 바이트코드)
     │  D8 (디버그) / R8 (릴리즈)
     ▼
.dex (Dalvik 바이트코드)
     │
     ▼
classes.dex, classes2.dex, ...  (MultiDex)
```

### 64K 메서드 제한

DEX 파일 하나의 메서드 참조 인덱스는 16비트(2^16 = 65,536)로 제한된다.

```
메서드 참조 ID: unsigned short (16bit)
→ 최대 65,536개

초과 시: "Cannot fit requested classes in a single dex file"
```

**해결 방법:**

```groovy
// build.gradle.kts
android {
    defaultConfig {
        // minSdk 21+ → ART가 네이티브로 MultiDex 지원
        minSdk = 21

        // minSdk < 21일 때만 필요
        multiDexEnabled = true
    }
}
```

**메서드 수 확인:**

```bash
# APK의 DEX 메서드 수 확인
# Android Studio → Build → Analyze APK → classes.dex 선택
# 또는
apkanalyzer dex references app-release.apk
```

### D8 vs R8

```
D8 (디버그 빌드):
├── .class → .dex 변환만 수행
├── 디슈가링 (Java 8+ API를 하위 버전 호환)
└── 빠른 빌드

R8 (릴리즈 빌드):
├── .class → .dex 변환
├── 코드 축소 (Tree Shaking) - 미사용 코드 제거
├── 난독화 (Obfuscation) - 이름을 a, b, c로
├── 최적화 (Optimization) - 인라이닝 등
└── ProGuard 규칙 호환
```

```groovy
// build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled = true   // R8 활성화
            isShrinkResources = true // 미사용 리소스 제거
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

### DEX 파일 구조

```
┌──────────────────┐
│    Header        │  매직넘버, 체크섬, 파일 크기
├──────────────────┤
│   String IDs     │  문자열 테이블
├──────────────────┤
│    Type IDs      │  타입(클래스) 테이블
├──────────────────┤
│   Proto IDs      │  메서드 프로토타입
├──────────────────┤
│   Field IDs      │  필드 참조
├──────────────────┤
│  Method IDs      │  메서드 참조 (← 64K 제한)
├──────────────────┤
│  Class Defs      │  클래스 정의
├──────────────────┤
│     Data         │  실제 바이트코드, 어노테이션 등
└──────────────────┘
```

### 디슈가링 (Desugaring)

D8/R8이 Java 8+ 문법을 하위 Android 버전에서도 동작하도록 변환한다.

```kotlin
// 개발자가 작성한 코드 (Java 8 Stream API)
val names = users.stream()
    .filter { it.age > 20 }
    .map { it.name }
    .collect(Collectors.toList())

// D8이 디슈가링: minSdk < 24여도 동작하도록 변환
// → 내부적으로 호환 코드로 대체
```

```groovy
// build.gradle.kts - coreLibraryDesugaring 활성화
android {
    compileOptions {
        isCoreLibraryDesugaringEnabled = true
    }
}

dependencies {
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.0.4")
}
```

## 🔗 관련 개념

- [[00-CS-기초/JVM/Dalvik-vs-ART|Dalvik vs ART]]
- [[00-CS-기초/JVM/JVM-구조|JVM 구조]]
- [[00-CS-기초/컴파일과-런타임/컴파일-과정|컴파일 과정]]
- [[00-CS-기초/컴파일과-런타임/바이트코드와-기계어|바이트코드와 기계어]]

## 📚 더 보기

- [D8/R8 공식 문서](https://developer.android.com/build/shrink-code)
- [MultiDex 가이드](https://developer.android.com/build/multidex)
- [APK Analyzer](https://developer.android.com/studio/debug/apk-analyzer)

---

**핵심 요약:** DEX는 Android 전용 바이트코드로, .class보다 모바일에 최적화되어 있다. 64K 메서드 제한이 있고, R8이 릴리즈 빌드에서 코드 축소/난독화/최적화를 수행한다.
