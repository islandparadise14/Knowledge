---
tags: [cs, compile, kapt, ksp, annotation-processing]
---

# kapt vs ksp

## 💡 핵심 개념

**kapt**(Kotlin Annotation Processing Tool)와 **ksp**(Kotlin Symbol Processing)는 둘 다 어노테이션을 읽고 코드를 자동 생성하는 도구다. kapt는 Java의 APT를 Kotlin에서 쓸 수 있게 감싼 것이고, ksp는 Kotlin 전용으로 새로 만든 것이다. **ksp가 kapt보다 최대 2배 이상 빠르다.**

## 📌 왜 필요한가?

Dagger/Hilt, Room, Moshi, Glide 등 주요 Android 라이브러리는 어노테이션 기반으로 코드를 생성한다. kapt에서 ksp로의 마이그레이션은 빌드 속도를 크게 개선하고, Google도 ksp 사용을 공식 권장한다.

## 🔍 자세히

### 어노테이션 프로세싱이란?

컴파일 타임에 어노테이션(`@Entity`, `@Inject` 등)을 읽고 보일러플레이트 코드를 자동 생성하는 기술이다.

```kotlin
// 개발자가 작성하는 코드
@Entity(tableName = "users")
data class User(
    @PrimaryKey val id: Int,
    @ColumnInfo(name = "user_name") val name: String
)

@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    fun getAll(): Flow<List<User>>
}

// 어노테이션 프로세서(Room)가 컴파일 시 자동 생성하는 코드:
// - UserDao_Impl.java (SQL 실행 구현체)
// - AppDatabase_Impl.java (DB 초기화 코드)
// → 개발자는 인터페이스만 정의하면 됨
```

### kapt 동작 방식

```
.kt 소스
    │
    ▼
[Kotlin 컴파일러] → Java Stub 생성 (.java)  ← 여기가 병목!
    │
    ▼
[Java APT] → 스텁을 읽고 코드 생성
    │
    ▼
[Kotlin 컴파일러] → 최종 컴파일
```

**핵심 문제:** Kotlin 코드를 Java 스텁으로 변환하는 과정이 추가로 필요하다. 이 과정에서 전체 소스를 한 번 더 처리해야 하므로 느리다.

```groovy
// build.gradle.kts - kapt 설정
plugins {
    id("kotlin-kapt")
}

dependencies {
    implementation("androidx.room:room-runtime:2.6.1")
    kapt("androidx.room:room-compiler:2.6.1")

    implementation("com.google.dagger:hilt-android:2.51")
    kapt("com.google.dagger:hilt-android-compiler:2.51")
}
```

### ksp 동작 방식

```
.kt 소스
    │
    ▼
[Kotlin 컴파일러] → Kotlin Symbol 직접 분석  ← Java 스텁 불필요!
    │
    ▼
[KSP 프로세서] → 심볼을 읽고 코드 생성
    │
    ▼
[Kotlin 컴파일러] → 최종 컴파일
```

**핵심 개선:** Java 스텁 생성 단계가 없다. Kotlin 컴파일러의 심볼 정보를 직접 읽으므로 훨씬 빠르다.

```groovy
// build.gradle.kts - ksp 설정
plugins {
    id("com.google.devtools.ksp") version "2.0.0-1.0.24"
}

dependencies {
    implementation("androidx.room:room-runtime:2.6.1")
    ksp("androidx.room:room-compiler:2.6.1")

    implementation("com.google.dagger:hilt-android:2.51")
    ksp("com.google.dagger:hilt-android-compiler:2.51")
}
```

### 비교

| | kapt | ksp |
|---|---|---|
| **기반** | Java APT 래퍼 | Kotlin 네이티브 |
| **속도** | 느림 (Java 스텁 생성) | **최대 2배 이상 빠름** |
| **증분 컴파일** | 제한적 | 잘 지원 |
| **멀티플랫폼** | JVM만 | KMP 지원 |
| **Kotlin 정보** | Java 스텁으로 변환 시 손실 | Kotlin 심볼 직접 접근 |
| **상태** | 유지보수 모드 | **Google 공식 권장** |

### kapt에서 ksp로 마이그레이션

**1. 플러그인 변경**

```groovy
// before
plugins {
    id("kotlin-kapt")
}

// after
plugins {
    id("com.google.devtools.ksp") version "2.0.0-1.0.24"
}
```

**2. 의존성 변경**

```groovy
// before
kapt("androidx.room:room-compiler:2.6.1")
kapt("com.google.dagger:hilt-android-compiler:2.51")

// after
ksp("androidx.room:room-compiler:2.6.1")
ksp("com.google.dagger:hilt-android-compiler:2.51")
```

**3. kapt 옵션 → ksp 옵션**

```groovy
// before
kapt {
    arguments {
        arg("room.schemaLocation", "$projectDir/schemas")
    }
}

// after
ksp {
    arg("room.schemaLocation", "$projectDir/schemas")
}
```

### 주요 라이브러리 ksp 지원 현황

```
✅ Room           - ksp 지원
✅ Hilt/Dagger    - ksp 지원
✅ Moshi          - ksp 지원
✅ Glide          - ksp 지원
✅ Koin           - ksp 지원 (koin-annotations)
✅ Kotlin Serialization - 컴파일러 플러그인 (kapt/ksp 불필요)
```

### 주의사항

**1. ksp와 kapt 혼용**

```groovy
// 일부 라이브러리만 ksp를 지원하는 경우 혼용 가능
// 하지만 kapt가 하나라도 있으면 스텁 생성이 발생
plugins {
    id("kotlin-kapt")
    id("com.google.devtools.ksp")
}

dependencies {
    ksp("androidx.room:room-compiler:2.6.1")      // ksp 지원
    kapt("some.legacy:processor:1.0")              // ksp 미지원
}
// → kapt를 완전히 제거해야 빌드 속도 이점을 최대로 누릴 수 있음
```

**2. ksp 버전은 Kotlin 버전에 종속**

```groovy
// Kotlin 버전과 ksp 버전이 맞아야 함
// Kotlin 2.0.0 → ksp 2.0.0-1.0.24
// 버전 불일치 시 빌드 에러 발생
plugins {
    kotlin("android") version "2.0.0"
    id("com.google.devtools.ksp") version "2.0.0-1.0.24"
    //                                     ^^^^^ Kotlin 버전과 일치
}
```

**3. 생성된 코드 위치**

```
kapt 생성 코드: build/generated/source/kapt/
ksp 생성 코드:  build/generated/ksp/
```

## 💻 실전 예제

### 커스텀 ksp 프로세서 (간단 예시)

```kotlin
// 프로세서 정의
class MyProcessor(
    private val codeGenerator: CodeGenerator,
    private val logger: KSPLogger
) : SymbolProcessor {

    override fun process(resolver: Resolver): List<KSAnnotated> {
        val symbols = resolver.getSymbolsWithAnnotation("com.example.MyAnnotation")

        symbols.filterIsInstance<KSClassDeclaration>().forEach { classDecl ->
            val className = classDecl.simpleName.asString()
            logger.info("Processing $className")

            // 코드 생성
            val file = codeGenerator.createNewFile(
                Dependencies(false, classDecl.containingFile!!),
                classDecl.packageName.asString(),
                "${className}Generated"
            )
            file.write("// Generated code for $className".toByteArray())
            file.close()
        }

        return emptyList()
    }
}

class MyProcessorProvider : SymbolProcessorProvider {
    override fun create(environment: SymbolProcessorEnvironment): SymbolProcessor {
        return MyProcessor(environment.codeGenerator, environment.logger)
    }
}
```

### 빌드 속도 측정

```bash
# kapt 빌드 시간 확인
./gradlew :app:kaptDebugKotlin --profile

# ksp 빌드 시간 확인
./gradlew :app:kspDebugKotlin --profile

# 전체 빌드 비교
./gradlew assembleDebug --profile --rerun-tasks
```

## 🔗 관련 개념

- [[00-CS-기초/컴파일과-런타임/컴파일-과정|컴파일 과정]]
- [[00-CS-기초/컴파일과-런타임/컴파일-타임-vs-런타임|컴파일 타임 vs 런타임]]
- [[15-빌드-배포/Gradle/Gradle-기초|Gradle]]
- [[20-라이브러리/DI/Hilt|Hilt]]

## 📚 더 보기

- [KSP 공식 문서](https://kotlinlang.org/docs/ksp-overview.html)
- [KSP GitHub](https://github.com/google/ksp)
- [kapt에서 ksp로 마이그레이션](https://developer.android.com/build/migrate-to-ksp)

---

**핵심 요약:** kapt는 Java 스텁 생성 때문에 느리다. ksp는 Kotlin 심볼을 직접 분석해서 최대 2배 이상 빠르다. 주요 라이브러리 대부분 ksp를 지원하므로, kapt를 완전히 제거하는 것이 목표.
