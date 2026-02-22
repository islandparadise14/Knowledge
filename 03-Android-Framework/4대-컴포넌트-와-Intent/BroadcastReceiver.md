---
tags: [android, broadcast-receiver, 4대컴포넌트]
---

# BroadcastReceiver

## 💡 핵심 개념

시스템 또는 앱에서 발생하는 이벤트(브로드캐스트)를 수신하는 컴포넌트. 배터리 부족, 네트워크 변경, 부팅 완료 등 시스템 이벤트에 반응하거나 앱 간 메시지를 주고받을 수 있다.

## 📌 왜 필요한가?

앱이 실행 중이지 않을 때도 특정 시스템 이벤트에 반응해야 할 때 필요하다. Activity나 Service와 달리 이벤트가 발생할 때만 깨어나므로 리소스 효율적이다.

## 🔍 자세히

### 기본 구조

```kotlin
class MyReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        // 이벤트 처리
        // onReceive는 Main Thread에서 실행됨
        // 10초 이상 걸리면 ANR → 무거운 작업은 Service나 코루틴으로
        when (intent.action) {
            Intent.ACTION_BOOT_COMPLETED -> startMyService(context)
            Intent.ACTION_BATTERY_LOW -> notifyUser(context)
        }
    }
}
```

### 등록 방법 2가지

**1. 정적 등록 (Manifest) — 앱이 꺼져있어도 수신**

```xml
<receiver
    android:name=".MyReceiver"
    android:exported="false">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

**2. 동적 등록 (코드) — 앱 실행 중에만 수신**

```kotlin
class MainActivity : AppCompatActivity() {
    private val receiver = NetworkReceiver()

    override fun onStart() {
        super.onStart()
        val filter = IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION)
        registerReceiver(receiver, filter)
    }

    override fun onStop() {
        super.onStop()
        unregisterReceiver(receiver)  // 반드시 해제
    }
}
```

| | 정적 등록 | 동적 등록 |
|--|--|--|
| 앱 꺼져있을 때 | O (일부 제한) | X |
| 등록/해제 제어 | X | O |
| Android 8+ 제한 | 암시적 브로드캐스트 대부분 불가 | 제한 없음 |

### 주요 시스템 브로드캐스트

```kotlin
// 부팅 완료 (정적 등록 가능)
Intent.ACTION_BOOT_COMPLETED

// 네트워크 상태 변경 (동적 등록만)
ConnectivityManager.CONNECTIVITY_ACTION

// 배터리 상태
Intent.ACTION_BATTERY_LOW
Intent.ACTION_BATTERY_OKAY

// 화면 켜짐/꺼짐 (동적 등록만)
Intent.ACTION_SCREEN_ON
Intent.ACTION_SCREEN_OFF

// 앱 설치/삭제
Intent.ACTION_PACKAGE_ADDED
Intent.ACTION_PACKAGE_REMOVED
```

### 커스텀 브로드캐스트 전송

```kotlin
// 전송
val intent = Intent("com.example.MY_ACTION").apply {
    putExtra("data", "값")
}
sendBroadcast(intent)

// 수신
class MyReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val data = intent.getStringExtra("data")
    }
}
```

### LocalBroadcastManager (앱 내부 통신)

앱 내부에서만 브로드캐스트를 주고받을 때 사용. 보안상 안전하고 성능도 좋다.
단, **deprecated** 되었으므로 새 코드에서는 SharedFlow 사용을 권장한다.

```kotlin
// 권장 대안 — SharedFlow
class EventBus {
    private val _events = MutableSharedFlow<AppEvent>()
    val events = _events.asSharedFlow()

    suspend fun emit(event: AppEvent) = _events.emit(event)
}
```

### Android 8+ 백그라운드 제한

Android 8(Oreo)부터 대부분의 암시적 브로드캐스트를 Manifest에 정적 등록할 수 없다.

```
정적 등록 가능한 예외 목록:
  - ACTION_BOOT_COMPLETED
  - ACTION_LOCKED_BOOT_COMPLETED
  - SMS/MMS 수신
  - 블루투스 페어링 요청 등 일부

정적 등록 불가 (동적 등록만):
  - CONNECTIVITY_ACTION
  - ACTION_SCREEN_ON/OFF
  - ACTION_BATTERY_CHANGED 등 대부분
```

## 💻 실전 예제

### 네트워크 상태 모니터링

```kotlin
class NetworkReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val cm = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        val isConnected = cm.activeNetworkInfo?.isConnected == true

        // ViewModel이나 Repository에 상태 전달
    }
}

// Activity에서 등록
registerReceiver(
    NetworkReceiver(),
    IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION)
)
```

### 부팅 시 서비스 시작

```kotlin
class BootReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == Intent.ACTION_BOOT_COMPLETED) {
            context.startForegroundService(
                Intent(context, MyService::class.java)
            )
        }
    }
}
```

## 🔗 관련 개념

- [[03-Android-Framework/4대-컴포넌트-와-Intent/Activity]]
- [[03-Android-Framework/4대-컴포넌트-와-Intent/Service]]

## 📚 더 보기

- [Android Docs - BroadcastReceiver](https://developer.android.com/guide/components/broadcasts)

---

**핵심 요약:** 시스템/앱 이벤트 수신. onReceive는 Main Thread — 무거운 작업 금지. Android 8+부터 정적 등록 제한. 앱 내부 통신은 SharedFlow 권장.
