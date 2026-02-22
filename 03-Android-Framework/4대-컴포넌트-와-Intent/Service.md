---
tags: [android, service, 4대컴포넌트, background]
---

# Service

## 💡 핵심 개념

화면 없이 백그라운드에서 실행되는 컴포넌트. 음악 재생, 파일 다운로드, 위치 추적처럼 UI 없이 오래 실행되는 작업에 사용한다.

## 📌 왜 필요한가?

Activity는 화면이 꺼지면 종료될 수 있다. Service는 UI와 독립적으로 실행되므로 앱이 백그라운드로 가도 작업을 유지할 수 있다.

## 🔍 자세히

### Service의 3가지 종류

```
Started Service   — startService()로 시작. 명시적으로 중지할 때까지 실행
Bound Service     — bindService()로 시작. 컴포넌트와 연결되어 통신 가능
Foreground Service — 알림을 띄우며 실행. 시스템이 함부로 종료 못함
```

### Started Service

```kotlin
class DownloadService : Service() {

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val url = intent?.getStringExtra("url")

        // 백그라운드 작업 (코루틴 권장)
        CoroutineScope(Dispatchers.IO).launch {
            download(url)
            stopSelf()  // 작업 완료 후 스스로 종료
        }

        return START_STICKY  // 시스템이 종료해도 재시작
    }

    override fun onBind(intent: Intent?): IBinder? = null  // Started는 null
}

// 시작
startService(Intent(this, DownloadService::class.java).apply {
    putExtra("url", "https://example.com/file.zip")
})

// 중지
stopService(Intent(this, DownloadService::class.java))
```

**onStartCommand 반환값**

| 값 | 동작 |
|----|------|
| `START_STICKY` | 종료되면 재시작, Intent는 null로 전달 |
| `START_NOT_STICKY` | 종료되면 재시작 안 함 |
| `START_REDELIVER_INTENT` | 종료되면 재시작, 마지막 Intent 재전달 |

### Foreground Service

백그라운드 제한을 피하기 위해 알림을 띄우며 실행. 음악 앱, 운동 앱의 위치 추적 등에 사용.

```kotlin
class MusicService : Service() {

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val notification = NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("음악 재생 중")
            .setContentText("노래 제목")
            .setSmallIcon(R.drawable.ic_music)
            .build()

        startForeground(NOTIFICATION_ID, notification)  // 포그라운드로 전환

        // 음악 재생 로직
        return START_STICKY
    }

    override fun onBind(intent: Intent?): IBinder? = null
}

// Android 9+ — Manifest에 권한 필요
// <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />

// Android 14+ — 타입 명시 필요
// <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
```

### Bound Service

컴포넌트(Activity 등)와 클라이언트-서버 관계로 통신. 연결한 컴포넌트가 모두 언바인드되면 자동 종료.

```kotlin
class TimerService : Service() {
    private val binder = TimerBinder()

    inner class TimerBinder : Binder() {
        fun getService(): TimerService = this@TimerService
    }

    var elapsedSeconds = 0
        private set

    override fun onBind(intent: Intent): IBinder = binder
}

// Activity에서 사용
class MainActivity : AppCompatActivity() {
    private var timerService: TimerService? = null
    private var isBound = false

    private val connection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName, binder: IBinder) {
            timerService = (binder as TimerService.TimerBinder).getService()
            isBound = true
        }

        override fun onServiceDisconnected(name: ComponentName) {
            isBound = false
        }
    }

    override fun onStart() {
        super.onStart()
        bindService(Intent(this, TimerService::class.java), connection, BIND_AUTO_CREATE)
    }

    override fun onStop() {
        super.onStop()
        if (isBound) {
            unbindService(connection)
            isBound = false
        }
    }
}
```

### Service 생명주기

```
Started Service
  startService() → onCreate() → onStartCommand() → 실행중 → stopSelf()/stopService() → onDestroy()

Bound Service
  bindService() → onCreate() → onBind() → 실행중 → 모두 unbind → onUnbind() → onDestroy()

둘 다 사용
  startService() + bindService() → 모두 unbind해도 stopService() 전까지 유지
```

### Manifest 등록

```xml
<service
    android:name=".MusicService"
    android:exported="false"
    android:foregroundServiceType="mediaPlayback" />
```

## 💻 실전 패턴

### WorkManager vs Service

| | Service | WorkManager |
|--|--|--|
| 즉시 실행 | O | △ (OS 스케줄링) |
| 앱 종료 후 유지 | Foreground만 | O |
| 제약 조건 | X | O (네트워크, 충전 중 등) |
| 사용 사례 | 음악 재생, 실시간 위치 | 백업, 동기화, 업로드 |

음악 재생처럼 **사용자가 인지하는 실시간 작업** → Foreground Service
백업처럼 **나중에 해도 되는 작업** → WorkManager

## 🔗 관련 개념

- [[03-Android-Framework/4대-컴포넌트-와-Intent/Activity]]
- [[03-Android-Framework/4대-컴포넌트-와-Intent/BroadcastReceiver]]

## 📚 더 보기

- [Android Docs - Service](https://developer.android.com/guide/components/services)

---

**핵심 요약:** UI 없는 백그라운드 작업. 오래 실행하려면 Foreground Service + 알림. 단순 백그라운드 작업은 WorkManager 고려.
