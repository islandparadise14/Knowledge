---
tags: [network, websocket, realtime, okhttp, cs-기초]
---

# WebSocket

## 💡 핵심 개념

**WebSocket**은 클라이언트와 서버 간 양방향 실시간 통신을 제공하는 프로토콜이다. HTTP와 달리 한 번 연결하면 양쪽에서 자유롭게 데이터를 주고받을 수 있다. 채팅, 실시간 알림, 주식 시세 등 지속적인 데이터 스트리밍에 사용된다.

## 📌 왜 필요한가?

HTTP는 요청-응답 방식이라 서버가 먼저 데이터를 보낼 수 없다. 채팅이나 실시간 알림처럼 서버에서 클라이언트로 즉시 데이터를 전달해야 하는 경우 WebSocket이 필수적이다.

## 🔍 자세히

### HTTP vs WebSocket

```
HTTP (요청-응답):
클라이언트 ──요청──> 서버
클라이언트 <──응답── 서버
(매번 새 요청 필요)

WebSocket (양방향):
클라이언트 ──HTTP 업그레이드──> 서버  (핸드셰이크)
클라이언트 <════════════════> 서버   (양방향 통신)
(연결 유지, 자유롭게 데이터 교환)
```

### WebSocket 핸드셰이크

```
클라이언트 → 서버:
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

서버 → 클라이언트:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### OkHttp WebSocket 구현

```kotlin
class ChatWebSocket(
    private val client: OkHttpClient
) {
    private var webSocket: WebSocket? = null
    private val _messages = MutableSharedFlow<ChatMessage>()
    val messages: SharedFlow<ChatMessage> = _messages.asSharedFlow()

    private val _connectionState = MutableStateFlow(ConnectionState.DISCONNECTED)
    val connectionState: StateFlow<ConnectionState> = _connectionState.asStateFlow()

    enum class ConnectionState {
        CONNECTING, CONNECTED, DISCONNECTED, FAILED
    }

    fun connect(url: String, token: String) {
        _connectionState.value = ConnectionState.CONNECTING

        val request = Request.Builder()
            .url(url)
            .addHeader("Authorization", "Bearer $token")
            .build()

        webSocket = client.newWebSocket(request, object : WebSocketListener() {

            override fun onOpen(webSocket: WebSocket, response: Response) {
                _connectionState.value = ConnectionState.CONNECTED
            }

            override fun onMessage(webSocket: WebSocket, text: String) {
                // 텍스트 메시지 수신
                val message = Json.decodeFromString<ChatMessage>(text)
                _messages.tryEmit(message)
            }

            override fun onMessage(webSocket: WebSocket, bytes: ByteString) {
                // 바이너리 메시지 수신
            }

            override fun onClosing(webSocket: WebSocket, code: Int, reason: String) {
                webSocket.close(1000, null)
                _connectionState.value = ConnectionState.DISCONNECTED
            }

            override fun onFailure(
                webSocket: WebSocket,
                t: Throwable,
                response: Response?
            ) {
                _connectionState.value = ConnectionState.FAILED
                // 재연결 로직
                reconnect(url, token)
            }
        })
    }

    fun sendMessage(message: String) {
        webSocket?.send(message)
    }

    fun disconnect() {
        webSocket?.close(1000, "사용자가 연결을 종료했습니다")
        webSocket = null
    }

    private fun reconnect(url: String, token: String) {
        // 지수 백오프로 재연결
        CoroutineScope(Dispatchers.IO).launch {
            delay(3000)
            connect(url, token)
        }
    }
}
```

### ViewModel에서 WebSocket 사용

```kotlin
class ChatViewModel(
    private val chatWebSocket: ChatWebSocket
) : ViewModel() {

    val messages = chatWebSocket.messages
        .scan(emptyList<ChatMessage>()) { acc, message -> acc + message }
        .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())

    val connectionState = chatWebSocket.connectionState

    fun connect() {
        chatWebSocket.connect(
            url = "wss://chat.example.com/ws",
            token = "eyJhbGciOi..."
        )
    }

    fun send(text: String) {
        val json = Json.encodeToString(
            ChatMessage(content = text, timestamp = System.currentTimeMillis())
        )
        chatWebSocket.sendMessage(json)
    }

    override fun onCleared() {
        chatWebSocket.disconnect()
    }
}
```

### Ktor WebSocket 클라이언트

```kotlin
val client = HttpClient(CIO) {
    install(WebSockets) {
        pingInterval = 15_000  // 15초마다 ping
    }
}

suspend fun connectChat() {
    client.webSocket("wss://chat.example.com/ws") {
        // 메시지 전송
        send(Frame.Text("""{"type":"join","room":"general"}"""))

        // 메시지 수신
        for (frame in incoming) {
            when (frame) {
                is Frame.Text -> {
                    val text = frame.readText()
                    println("수신: $text")
                }
                is Frame.Binary -> { /* 바이너리 처리 */ }
                is Frame.Close -> { println("연결 종료") }
                else -> { }
            }
        }
    }
}
```

### WebSocket vs 대안 기술

```
WebSocket:
  - 양방향 실시간 통신
  - 지속 연결 유지
  - 채팅, 게임, 실시간 협업

SSE (Server-Sent Events):
  - 서버 → 클라이언트 단방향
  - HTTP 기반 (별도 프로토콜 불필요)
  - 실시간 알림, 뉴스 피드

Polling:
  - 주기적으로 HTTP 요청
  - 구현 간단, 리소스 낭비
  - 빈도 낮은 업데이트

Long Polling:
  - 서버가 새 데이터 있을 때까지 응답 지연
  - WebSocket 미지원 환경의 대안
```

### 주의사항

```kotlin
// 1. 앱 생명주기 관리
class ChatActivity : AppCompatActivity() {
    override fun onStart() {
        super.onStart()
        viewModel.connect()  // 포그라운드에서 연결
    }

    override fun onStop() {
        super.onStop()
        viewModel.disconnect()  // 백그라운드에서 해제
    }
}

// 2. 배터리 고려
// 백그라운드에서 WebSocket 유지 시 FCM 푸시로 대체 권장

// 3. 네트워크 변경 대응
// Wi-Fi ↔ 셀룰러 전환 시 재연결 필요
```

## 🔗 관련 개념

- [[00-CS-기초/네트워크/HTTP-HTTPS|HTTP/HTTPS (WebSocket 핸드셰이크)]]
- [[00-CS-기초/네트워크/TCP-vs-UDP|TCP vs UDP (WebSocket은 TCP 기반)]]
- [[00-CS-기초/네트워크/REST-API|REST API (요청-응답 방식)]]

## 📚 더 보기

- [OkHttp WebSocket](https://square.github.io/okhttp/features/websocket/)
- [Ktor WebSocket Client](https://ktor.io/docs/client-websockets.html)

---

**핵심 요약:** WebSocket은 양방향 실시간 통신 프로토콜. OkHttp/Ktor로 구현하며, 생명주기 관리와 재연결 로직이 핵심.
