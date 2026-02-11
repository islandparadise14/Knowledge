---
tags: [os, ipc, binder, android, aidl, intent, contentprovider]
---

# IPC (프로세스간 통신)

## 💡 핵심 개념

**IPC(Inter-Process Communication)**는 서로 다른 프로세스가 데이터를 주고받는 메커니즘이다. 프로세스는 각자 독립된 메모리 공간을 가지므로 직접 데이터를 공유할 수 없고, 커널을 통한 IPC가 필요하다. Android는 Linux의 전통적인 IPC 대신 **Binder**를 핵심 IPC 메커니즘으로 사용한다.

## 📌 왜 필요한가?

Android의 4대 컴포넌트(Activity, Service, BroadcastReceiver, ContentProvider) 간 통신, 시스템 서비스 호출, 앱 간 데이터 공유 모두 IPC 기반이다. Binder를 이해하면 Android 시스템의 동작 원리를 깊이 파악할 수 있다.

## 🔍 자세히

### Linux 전통 IPC 메커니즘

| 방식 | 설명 | 복사 횟수 |
|------|------|-----------|
| Pipe | 단방향 바이트 스트림 | 2회 |
| Message Queue | 커널이 관리하는 메시지 큐 | 2회 |
| Shared Memory | 같은 물리 메모리를 공유 | 0회 |
| Socket | 네트워크 인터페이스 기반 | 2회 |
| Signal | 비동기 알림 | N/A |
| Semaphore | 동기화 도구 (데이터 전송 X) | N/A |

**복사 2회의 의미:**
```
프로세스 A → (1회 복사) → 커널 버퍼 → (2회 복사) → 프로세스 B
```

### Android의 Binder IPC

Binder는 Android의 핵심 IPC 메커니즘이다. Linux 커널 드라이버(`/dev/binder`)로 구현되어 있다.

**Binder의 장점:**
- **1회 복사**: `mmap()`을 이용해 커널과 수신 프로세스가 같은 물리 메모리를 공유
- **보안**: 호출자의 UID/PID를 커널이 자동 검증
- **객체 지향**: 원격 객체의 메서드를 직접 호출하는 것처럼 사용
- **참조 카운팅**: 원격 객체의 생명주기 자동 관리

```
Binder 통신 흐름:

클라이언트 프로세스              서버 프로세스
┌─────────────┐              ┌─────────────┐
│  Proxy      │              │  Stub       │
│  (자동생성)  │              │  (자동생성)  │
└──────┬──────┘              └──────▲──────┘
       │                            │
       ▼                            │
┌──────────────────────────────────────┐
│         Binder Driver (/dev/binder)  │
│         커널 공간 (mmap 공유 메모리)    │
└──────────────────────────────────────┘
       1회 복사로 데이터 전달
```

### AIDL (Android Interface Definition Language)

프로세스 간 인터페이스를 정의하는 언어. 컴파일러가 Proxy/Stub 코드를 자동 생성한다.

```java
// IRemoteService.aidl
interface IRemoteService {
    int calculate(int a, int b);
    List<String> getItems();
}
```

```kotlin
// 서버 측 - Service에서 구현
class RemoteService : Service() {

    private val binder = object : IRemoteService.Stub() {
        override fun calculate(a: Int, b: Int): Int {
            return a + b
        }

        override fun getItems(): List<String> {
            return listOf("item1", "item2", "item3")
        }
    }

    override fun onBind(intent: Intent): IBinder = binder
}

// 클라이언트 측 - 다른 앱에서 바인딩
class ClientActivity : AppCompatActivity() {

    private var remoteService: IRemoteService? = null

    private val connection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName, service: IBinder) {
            remoteService = IRemoteService.Stub.asInterface(service)
        }

        override fun onServiceDisconnected(name: ComponentName) {
            remoteService = null
        }
    }

    override fun onStart() {
        super.onStart()
        val intent = Intent("com.example.REMOTE_SERVICE")
        intent.setPackage("com.example.server")
        bindService(intent, connection, Context.BIND_AUTO_CREATE)
    }

    fun doCalculation() {
        // 마치 로컬 메서드 호출처럼 사용 (실제로는 IPC)
        val result = remoteService?.calculate(3, 5)
    }
}
```

### Intent - 고수준 IPC

Intent는 Binder 위에 구축된 고수준 메시징 시스템이다:

```kotlin
// 명시적 Intent - 특정 컴포넌트 직접 호출
val intent = Intent(this, DetailActivity::class.java).apply {
    putExtra("user_id", 123)
}
startActivity(intent)

// 암시적 Intent - 액션 기반으로 시스템이 적절한 앱 선택
val shareIntent = Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"
    putExtra(Intent.EXTRA_TEXT, "공유할 텍스트")
}
startActivity(Intent.createChooser(shareIntent, "공유하기"))

// BroadcastReceiver로 브로드캐스트 수신
class NetworkReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == ConnectivityManager.CONNECTIVITY_ACTION) {
            // 네트워크 상태 변화 처리
        }
    }
}
```

### ContentProvider - 구조화된 데이터 공유

앱 간 구조화된 데이터를 공유하는 IPC 메커니즘:

```kotlin
// ContentProvider 구현 (데이터 제공 앱)
class BookProvider : ContentProvider() {

    override fun query(
        uri: Uri, projection: Array<String>?,
        selection: String?, selectionArgs: Array<String>?,
        sortOrder: String?
    ): Cursor? {
        return when (uriMatcher.match(uri)) {
            BOOKS -> database.query("books", projection,
                selection, selectionArgs, null, null, sortOrder)
            BOOK_ID -> {
                val id = uri.lastPathSegment
                database.query("books", projection,
                    "_id=?", arrayOf(id), null, null, sortOrder)
            }
            else -> null
        }
    }

    override fun getType(uri: Uri): String = when (uriMatcher.match(uri)) {
        BOOKS -> "vnd.android.cursor.dir/vnd.example.books"
        BOOK_ID -> "vnd.android.cursor.item/vnd.example.books"
        else -> throw IllegalArgumentException()
    }

    // insert, update, delete 구현 생략
    override fun onCreate(): Boolean = true
    override fun insert(uri: Uri, values: ContentValues?): Uri? = null
    override fun delete(uri: Uri, sel: String?, args: Array<String>?): Int = 0
    override fun update(uri: Uri, values: ContentValues?, sel: String?, args: Array<String>?): Int = 0
}

// 데이터 소비 앱에서 접근
fun queryBooks(context: Context): List<String> {
    val books = mutableListOf<String>()
    val cursor = context.contentResolver.query(
        Uri.parse("content://com.example.bookprovider/books"),
        arrayOf("title"), null, null, null
    )
    cursor?.use {
        while (it.moveToNext()) {
            books.add(it.getString(0))
        }
    }
    return books
}
```

### Messenger - 경량 IPC

AIDL보다 간단한 단방향 메시지 기반 IPC:

```kotlin
// 서버 Service
class MessengerService : Service() {
    private val handler = object : Handler(Looper.getMainLooper()) {
        override fun handleMessage(msg: Message) {
            when (msg.what) {
                MSG_SAY_HELLO -> {
                    Toast.makeText(applicationContext, "Hello!", Toast.LENGTH_SHORT).show()
                }
            }
        }
    }

    private val messenger = Messenger(handler)

    override fun onBind(intent: Intent): IBinder = messenger.binder

    companion object {
        const val MSG_SAY_HELLO = 1
    }
}
```

### IPC 방식 비교 (Android)

| 방식 | 용도 | 복잡도 | 양방향 |
|------|------|--------|--------|
| Intent | Activity/BR 시작 | 낮음 | 제한적 |
| ContentProvider | 구조화된 데이터 공유 | 중간 | CRUD |
| Messenger | 간단한 메시지 전달 | 중간 | 단방향 |
| AIDL | 복잡한 인터페이스 | 높음 | 양방향 |
| Bound Service | 같은 앱 내 서비스 바인딩 | 낮음 | 양방향 |

### Parcelable - IPC 직렬화

Binder를 통해 객체를 전달하려면 **Parcelable**로 직렬화해야 한다:

```kotlin
@Parcelize
data class User(
    val id: Long,
    val name: String,
    val email: String
) : Parcelable

// Intent로 전달
intent.putExtra("user", user)

// 수신 측
val user = intent.getParcelableExtra<User>("user")
```

> Serializable보다 Parcelable이 빠른 이유: Serializable은 리플렉션을 사용하고 중간 객체를 생성하지만, Parcelable은 명시적 직렬화로 오버헤드가 적다.

## 🔗 관련 개념

- [[00-CS-기초/운영체제/프로세스-관리|프로세스 관리]]
- [[00-CS-기초/운영체제/커널-모드-vs-유저-모드|커널 모드 vs 유저 모드]]
- [[00-CS-기초/동시성/프로세스-vs-스레드|프로세스 vs 스레드]]
- [[00-CS-기초/메모리/가상-메모리|가상 메모리]]

## 📚 더 보기

- [Android Developer - AIDL](https://developer.android.com/develop/background-work/services/aidl)
- [Android Developer - Bound Services](https://developer.android.com/develop/background-work/services/bound-services)
- [Android Binder Framework](https://source.android.com/docs/core/architecture/hidl/binder-ipc)

---

**핵심 요약:** Android는 Binder IPC를 핵심으로 사용 (1회 복사, UID 보안). Intent/ContentProvider/AIDL/Messenger 모두 Binder 위에 구축. 객체 전달은 Parcelable로 직렬화.
