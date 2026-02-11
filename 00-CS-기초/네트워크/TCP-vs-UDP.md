---
tags: [network, tcp, udp, transport, cs-기초]
---

# TCP vs UDP

## 💡 핵심 개념

**TCP(Transmission Control Protocol)**는 연결 지향적이고 신뢰성 있는 전송 프로토콜이며, **UDP(User Datagram Protocol)**는 비연결형의 빠른 전송 프로토콜이다. 둘 다 OSI 4계층(전송 계층)에서 동작하며, 용도에 따라 선택한다.

## 📌 왜 필요한가?

Android 앱의 HTTP 통신(Retrofit, OkHttp)은 TCP 기반이고, 실시간 스트리밍이나 게임은 UDP를 사용한다. 프로토콜 특성을 이해해야 네트워크 성능을 최적화할 수 있다.

## 🔍 자세히

### TCP vs UDP 비교

| 특성 | TCP | UDP |
|------|-----|-----|
| 연결 방식 | 연결 지향 (3-way handshake) | 비연결형 |
| 신뢰성 | 보장 (재전송, 순서 보장) | 미보장 |
| 속도 | 상대적으로 느림 | 빠름 |
| 헤더 크기 | 20바이트 | 8바이트 |
| 흐름 제어 | 있음 | 없음 |
| 혼잡 제어 | 있음 | 없음 |
| 용도 | HTTP, HTTPS, FTP, 이메일 | DNS, 스트리밍, 게임, VoIP |

### TCP 3-Way Handshake

```
클라이언트                    서버
    │                          │
    │──── SYN ────────────────>│  1. 연결 요청
    │                          │
    │<─── SYN + ACK ──────────│  2. 요청 수락 + 확인
    │                          │
    │──── ACK ────────────────>│  3. 확인 응답
    │                          │
    │    (연결 수립 완료)        │
```

### TCP 4-Way Handshake (연결 종료)

```
클라이언트                    서버
    │                          │
    │──── FIN ────────────────>│  1. 종료 요청
    │<─── ACK ────────────────│  2. 확인
    │<─── FIN ────────────────│  3. 서버도 종료 요청
    │──── ACK ────────────────>│  4. 최종 확인
    │                          │
```

### Android에서 TCP 사용 (OkHttp)

```kotlin
// OkHttp는 TCP 기반 HTTP 통신
// 연결 풀링으로 TCP 핸드셰이크 비용 절감
val client = OkHttpClient.Builder()
    .connectionPool(ConnectionPool(
        maxIdleConnections = 5,    // 유휴 연결 최대 수
        keepAliveDuration = 5,     // 연결 유지 시간
        timeUnit = TimeUnit.MINUTES
    ))
    .connectTimeout(10, TimeUnit.SECONDS)  // TCP 연결 타임아웃
    .readTimeout(30, TimeUnit.SECONDS)     // 데이터 읽기 타임아웃
    .writeTimeout(30, TimeUnit.SECONDS)    // 데이터 쓰기 타임아웃
    .build()
```

### Android에서 UDP 사용

```kotlin
// UDP 소켓으로 데이터 전송 (반드시 백그라운드 스레드에서)
suspend fun sendUdpMessage(message: String, host: String, port: Int) =
    withContext(Dispatchers.IO) {
        val socket = DatagramSocket()
        try {
            val data = message.toByteArray()
            val address = InetAddress.getByName(host)
            val packet = DatagramPacket(data, data.size, address, port)
            socket.send(packet)
        } finally {
            socket.close()
        }
    }

// UDP로 데이터 수신
suspend fun receiveUdpMessage(port: Int): String =
    withContext(Dispatchers.IO) {
        val socket = DatagramSocket(port)
        try {
            val buffer = ByteArray(1024)
            val packet = DatagramPacket(buffer, buffer.size)
            socket.receive(packet)  // 블로킹
            String(packet.data, 0, packet.length)
        } finally {
            socket.close()
        }
    }
```

### HTTP/3과 QUIC

```
HTTP/1.1, HTTP/2 → TCP 기반
HTTP/3           → UDP 기반 (QUIC 프로토콜)

QUIC의 장점:
- UDP 위에 신뢰성 구현 (TCP의 장점 + UDP의 속도)
- 0-RTT 연결 설정 (재연결 시)
- 멀티플렉싱 (Head-of-line 블로킹 해결)
- 연결 마이그레이션 (Wi-Fi ↔ 셀룰러 전환 시 유지)
```

```kotlin
// OkHttp에서 HTTP/3 지원 (실험적)
val client = OkHttpClient.Builder()
    .protocols(listOf(Protocol.HTTP_3, Protocol.HTTP_2, Protocol.HTTP_1_1))
    .build()
```

### 실무에서의 선택 기준

```
TCP를 선택하는 경우:
├── API 통신 (Retrofit, OkHttp)
├── 파일 다운로드/업로드
├── WebSocket (채팅)
└── 데이터 정확성이 중요한 경우

UDP를 선택하는 경우:
├── 실시간 영상/음성 스트리밍
├── 온라인 게임 (위치 업데이트)
├── DNS 조회
└── IoT 센서 데이터 (일부 유실 허용)
```

## 🔗 관련 개념

- [[00-CS-기초/네트워크/OSI-7계층|OSI 7계층]]
- [[00-CS-기초/네트워크/HTTP-HTTPS|HTTP/HTTPS]]
- [[00-CS-기초/네트워크/WebSocket|WebSocket]]

## 📚 더 보기

- [OkHttp Connection Pool](https://square.github.io/okhttp/features/connections/)
- [HTTP/3 explained](https://http3-explained.haxx.se/)

---

**핵심 요약:** TCP는 신뢰성 보장(HTTP 통신), UDP는 속도 우선(스트리밍). Android HTTP 통신은 TCP 기반이며, OkHttp 연결 풀링으로 성능 최적화.
