---
tags: [android, intent, 4대컴포넌트]
---

# Intent

## 💡 핵심 개념

컴포넌트 간 통신을 위한 메시지 객체. Activity, Service, BroadcastReceiver를 시작하거나 데이터를 전달하는 데 사용한다. Android 시스템이 Intent를 받아 적절한 컴포넌트를 찾아 전달한다.

## 📌 왜 필요한가?

Android 컴포넌트는 직접 메서드를 호출하는 방식으로 통신하지 않는다. Intent라는 추상화된 메시지를 시스템에 전달하면 시스템이 적절한 컴포넌트를 찾아 실행한다. 이 덕분에 앱 간 통신과 느슨한 결합이 가능하다.

## 🔍 자세히

### 명시적 Intent vs 암시적 Intent

```
명시적 Intent — 대상 컴포넌트를 직접 지정
  Intent(context, TargetActivity::class.java)
  → 같은 앱 내부 화면 이동에 주로 사용

암시적 Intent — 작업(Action)만 지정, 시스템이 적합한 앱을 찾음
  Intent(Intent.ACTION_VIEW, Uri.parse("https://..."))
  → 브라우저, 카메라 등 다른 앱 기능 활용
```

### 명시적 Intent

```kotlin
// Activity 시작
val intent = Intent(this, DetailActivity::class.java).apply {
    putExtra("userId", "123")
    putExtra("isAdmin", true)
}
startActivity(intent)

// Service 시작
val intent = Intent(this, DownloadService::class.java).apply {
    putExtra("url", "https://example.com/file.zip")
}
startService(intent)

// BroadcastReceiver에 전송
val intent = Intent(this, MyReceiver::class.java)
sendBroadcast(intent)
```

### 암시적 Intent

```kotlin
// 웹 페이지 열기
startActivity(Intent(Intent.ACTION_VIEW, Uri.parse("https://example.com")))

// 전화 걸기
startActivity(Intent(Intent.ACTION_DIAL, Uri.parse("tel:010-1234-5678")))

// 이메일 보내기
startActivity(Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"
    putExtra(Intent.EXTRA_EMAIL, arrayOf("to@example.com"))
    putExtra(Intent.EXTRA_SUBJECT, "제목")
    putExtra(Intent.EXTRA_TEXT, "내용")
})

// 사진 선택
startActivity(Intent(Intent.ACTION_PICK, MediaStore.Images.Media.EXTERNAL_CONTENT_URI))

// 공유
startActivity(Intent.createChooser(
    Intent(Intent.ACTION_SEND).apply {
        type = "text/plain"
        putExtra(Intent.EXTRA_TEXT, "공유할 텍스트")
    },
    "공유하기"
))
```

### Intent 구성 요소

```kotlin
val intent = Intent().apply {
    // Action — 수행할 동작
    action = Intent.ACTION_VIEW

    // Data — 작업 대상 URI
    data = Uri.parse("https://example.com")

    // Category — 컴포넌트 종류 힌트
    addCategory(Intent.CATEGORY_BROWSABLE)

    // Type — 데이터 MIME 타입
    type = "image/*"

    // Extras — 추가 데이터 (Key-Value)
    putExtra("key", "value")

    // Flags — 동작 방식 제어
    flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP
}
```

### Flags

| Flag | 동작 |
|------|------|
| `FLAG_ACTIVITY_NEW_TASK` | 새 Task에서 Activity 시작 |
| `FLAG_ACTIVITY_CLEAR_TOP` | 스택에서 위에 쌓인 Activity 모두 제거 |
| `FLAG_ACTIVITY_SINGLE_TOP` | 최상단에 있으면 재사용 |
| `FLAG_ACTIVITY_NO_HISTORY` | 백스택에 추가하지 않음 |
| `FLAG_ACTIVITY_CLEAR_TASK` | 기존 Task를 비우고 시작 |

```kotlin
// 로그아웃 후 로그인 화면으로 — 뒤로 가기로 못 돌아가게
val intent = Intent(this, LoginActivity::class.java).apply {
    flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
}
startActivity(intent)
```

### 데이터 전달

**Extras — 기본 타입**

```kotlin
// 전달
intent.putExtra("name", "Alice")
intent.putExtra("age", 27)
intent.putExtra("isAdmin", true)
intent.putExtra("scores", intArrayOf(90, 85, 92))

// 수신
val name = intent.getStringExtra("name")
val age = intent.getIntExtra("age", 0)  // 기본값 0
val isAdmin = intent.getBooleanExtra("isAdmin", false)
```

**Parcelable — 객체 전달**

```kotlin
@Parcelize
data class User(val id: String, val name: String) : Parcelable

// 전달
intent.putExtra("user", User("1", "Alice"))

// 수신
val user = intent.getParcelableExtra<User>("user")
// Android 13+
val user = intent.getParcelableExtra("user", User::class.java)
```

### 암시적 Intent 처리 — Intent Filter

다른 앱의 암시적 Intent를 내 앱이 받으려면 Manifest에 등록한다.

```xml
<activity android:name=".ShareActivity">
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
    </intent-filter>
</activity>
```

### 수신 가능한 앱이 있는지 확인

암시적 Intent는 처리할 앱이 없으면 크래시가 발생한다.

```kotlin
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://example.com"))

if (intent.resolveActivity(packageManager) != null) {
    startActivity(intent)
} else {
    // 처리할 앱 없음
}

// 또는 try-catch
try {
    startActivity(intent)
} catch (e: ActivityNotFoundException) {
    Toast.makeText(this, "앱을 찾을 수 없습니다", Toast.LENGTH_SHORT).show()
}
```

### PendingIntent

나중에 다른 컴포넌트(알림, 위젯, AlarmManager)가 대신 실행하도록 Intent를 위임한다.

```kotlin
// 알림 탭 시 Activity 열기
val intent = Intent(this, MainActivity::class.java)
val pendingIntent = PendingIntent.getActivity(
    this,
    0,
    intent,
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
)

val notification = NotificationCompat.Builder(this, CHANNEL_ID)
    .setContentTitle("제목")
    .setContentIntent(pendingIntent)  // 탭하면 MainActivity 실행
    .build()

// AlarmManager — 특정 시간에 BroadcastReceiver 실행
val pendingIntent = PendingIntent.getBroadcast(
    this, 0,
    Intent(this, AlarmReceiver::class.java),
    PendingIntent.FLAG_IMMUTABLE
)
alarmManager.setExact(AlarmManager.RTC_WAKEUP, triggerTime, pendingIntent)
```

## 💻 실전 패턴

### Activity Result API로 결과 받기

```kotlin
// 사진 찍기
private val takePicture = registerForActivityResult(
    ActivityResultContracts.TakePicture()
) { success ->
    if (success) { /* 사진 처리 */ }
}
takePicture.launch(imageUri)

// 갤러리에서 선택
private val pickImage = registerForActivityResult(
    ActivityResultContracts.PickVisualMedia()
) { uri ->
    uri?.let { /* 이미지 처리 */ }
}
pickImage.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly))
```

### Deep Link

```xml
<!-- Manifest -->
<activity android:name=".ProductActivity">
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https"
              android:host="example.com"
              android:pathPrefix="/product" />
    </intent-filter>
</activity>
```

```kotlin
// https://example.com/product/123 링크 클릭 시 앱 실행
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    val productId = intent.data?.lastPathSegment  // "123"
}
```

### Intent와 IPC

#### 왜 IPC가 필요한가

Android의 모든 앱은 독립된 프로세스에서 실행된다. 프로세스는 메모리가 완전히 격리되어 있어 직접 접근이 불가하다.

```
[앱 A 프로세스]          [앱 B 프로세스]
   메모리 공간               메모리 공간
   ───────────              ───────────
   객체, 변수                객체, 변수
        ↑                        ↑
        └──────── 벽 ────────────┘
                직접 접근 불가
```

#### 실제 전달 경로

Intent를 보낼 때 상대 컴포넌트에 직접 전달하는 게 아니라 **시스템 프로세스(system_server)를 거친다.**

```
[내 앱 프로세스]          [system_server]          [상대 앱 프로세스]
                               AMS
startActivity(intent)    (ActivityManager
       │                   Service)
       │   Binder IPC         │   Binder IPC
       └─────────────→        └─────────────→  Activity 시작
         "이 Intent 처리해줘"   "이 컴포넌트
                                시작시켜줘"
```

- **Binder** — Android의 IPC 메커니즘. Intent가 프로세스를 넘을 때 사용하는 실제 통신 수단
- **AMS(ActivityManagerService)** — Intent를 받아 적합한 컴포넌트를 찾고, 실행을 지시하는 중앙 라우터

#### Intent가 Parcelable인 이유

Binder로 데이터를 전송하려면 **직렬화(바이트 배열로 변환)** 가 필요하다. 프로세스 간 메모리를 공유할 수 없으므로 데이터를 복사해서 보내야 한다.

```
[내 앱]                                    [system_server]
Intent 객체              →  직렬화  →      바이트 배열
  action: "VIEW"              (Parcel)      역직렬화
  data: Uri(...)                           Intent 객체 재생성
  extras: Bundle
```

Intent에 담는 객체가 반드시 **Parcelable** 이어야 하는 이유가 이것이다. 일반 객체는 프로세스를 넘을 수 없다.

```kotlin
// Binder 전송 한도 1MB
// 초과하면 TransactionTooLargeException
intent.putExtra("bitmap", hugeBitmap)  // 위험

// 큰 데이터는 Uri로 참조만 넘겨야 함
intent.putExtra("imageUri", uri)       // 안전
```

#### 같은 앱 내부 Intent도 AMS를 거친다

```
[내 앱]          [system_server/AMS]          [내 앱]
ActivityA    →   Intent 라우팅, 백스택 관리   →  ActivityB 시작
             Binder IPC                 Binder IPC
```

AMS가 백스택, 생명주기, Task 관리를 담당하기 때문에 같은 앱이어도 시스템을 통해야 한다.

#### 정리

| | 역할 |
|--|--|
| **Intent** | 무엇을 할지 담은 메시지 (Parcelable로 직렬화 가능) |
| **Binder** | 프로세스 간 메시지를 실제로 전달하는 통신 수단 |
| **AMS** | Intent를 받아 컴포넌트를 찾고 실행을 지시하는 라우터 |

```
Intent 종류              IPC 발생 여부       경로
────────────────────────────────────────────────────
같은 앱 Activity 이동   O (AMS 경유)        앱 → AMS → 앱
다른 앱 Activity 이동   O (AMS 경유)        앱 → AMS → 상대 앱
sendBroadcast           O (AMS 경유)        앱 → AMS → 수신 앱들
startService            O (AMS 경유)        앱 → AMS → 앱
```

## 🔗 관련 개념

- [[03-Android-Framework/4대-컴포넌트-와-Intent/Activity]]
- [[03-Android-Framework/4대-컴포넌트-와-Intent/Service]]
- [[03-Android-Framework/4대-컴포넌트-와-Intent/BroadcastReceiver]]
- [[03-Android-Framework/4대-컴포넌트-와-Intent/ContentProvider]]

## 📚 더 보기

- [Android Docs - Intent](https://developer.android.com/guide/components/intents-filters)

---

**핵심 요약:** 명시적은 내부 이동, 암시적은 시스템/다른 앱 기능 활용. PendingIntent로 나중에 실행 위임. 암시적 Intent는 처리할 앱 존재 여부 항상 확인.
