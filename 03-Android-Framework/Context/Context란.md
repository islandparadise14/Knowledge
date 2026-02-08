---
tags: [android, context]
---

# Context란

## 💡 핵심 개념

**Context**는 Android 시스템 서비스와 리소스에 접근하는 인터페이스다. Application Context와 Activity Context가 있다.

## 🔍 자세히

### Context 종류

**Application Context**
```kotlin
// 앱 생명주기와 동일
val appContext = applicationContext
val appContext2 = getApplication()

// 언제 사용?
// - 싱글톤 객체
// - 시스템 서비스 접근
```

**Activity Context**
```kotlin
// Activity 생명주기와 동일
val activityContext = this  // Activity에서

// 언제 사용?
// - UI 작업 (Dialog, Toast)
// - 테마가 필요한 경우
```

### 사용 가이드

```kotlin
// UI 작업 - Activity Context
AlertDialog.Builder(this)  // Activity context
Toast.makeText(this, "...", Toast.LENGTH_SHORT)

// 싱글톤 - Application Context
class MyManager(private val context: Context) {
    init {
        // Application context 전달해야 함
    }
}

// ViewModel - Application
class MyViewModel(
    private val app: Application
) : ViewModel()
```

### 주의사항

**1. Activity Context 저장 금지**
```kotlin
// 나쁨 - 메모리 누수
companion object {
    var context: Context? = null
}

fun init(context: Context) {
    MyClass.context = context  // Activity면 누수!
}

// 좋음
fun init(context: Context) {
    MyClass.context = context.applicationContext
}
```

**2. ContextWrapper**
```kotlin
// Activity, Service는 ContextWrapper
Activity extends ContextThemeWrapper extends ContextWrapper extends Context
```

## 🔗 관련 개념

- [[03-Android-Framework/메모리/Memory-Leak]]
- [[06-Architecture/ViewModel/ViewModel-기초]]

---

**핵심 요약:** Application Context(앱 생명주기), Activity Context(UI 작업). Activity Context 저장 금지.
