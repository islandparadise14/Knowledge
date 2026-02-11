---
tags: [network, osi, layer, protocol, cs-기초]
---

# OSI 7계층

## 💡 핵심 개념

**OSI(Open Systems Interconnection) 7계층 모델**은 네트워크 통신을 7개의 계층으로 나눈 표준 참조 모델이다. 각 계층은 독립적인 역할을 수행하며, 상위 계층은 하위 계층의 서비스를 이용한다. 실무에서는 TCP/IP 4계층 모델을 더 많이 사용하지만, 네트워크 문제를 분석할 때 OSI 모델이 유용하다.

## 📌 왜 필요한가?

Android 앱에서 네트워크 오류를 디버깅할 때 어떤 계층에서 문제가 발생했는지 파악해야 한다. OkHttp 인터셉터, DNS 오류, SSL 핸드셰이크 실패 등 모두 특정 계층과 관련된다.

## 🔍 자세히

### 7계층 구조

| 계층 | 이름 | 역할 | 프로토콜/기술 | Android 관련 |
|------|------|------|-------------|-------------|
| 7 | 응용(Application) | 사용자 인터페이스 | HTTP, FTP, DNS | Retrofit, OkHttp |
| 6 | 표현(Presentation) | 데이터 변환, 암호화 | SSL/TLS, JPEG, JSON | Gson, Moshi, kotlinx.serialization |
| 5 | 세션(Session) | 연결 관리 | NetBIOS, RPC | WebSocket 세션 |
| 4 | 전송(Transport) | 종단간 통신 | TCP, UDP | Socket, DatagramSocket |
| 3 | 네트워크(Network) | 라우팅, IP 주소 | IP, ICMP | ConnectivityManager |
| 2 | 데이터링크(Data Link) | 프레임 전송 | Ethernet, Wi-Fi | WifiManager |
| 1 | 물리(Physical) | 비트 전송 | 케이블, 무선 신호 | 하드웨어 레벨 |

### TCP/IP 4계층과 매핑

```
OSI 7계층              TCP/IP 4계층
─────────────────────────────────────
7. 응용 계층    ─┐
6. 표현 계층    ─┼── 응용 계층 (Application)
5. 세션 계층    ─┘
4. 전송 계층    ──── 전송 계층 (Transport)
3. 네트워크 계층 ──── 인터넷 계층 (Internet)
2. 데이터링크   ─┐
1. 물리 계층    ─┴── 네트워크 접근 계층 (Network Access)
```

### Android에서 계층별 동작 예시

```kotlin
// 7계층: 응용 - Retrofit으로 API 호출
interface ApiService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): User
}

// OkHttp 인터셉터로 각 계층의 동작을 관찰할 수 있다
val client = OkHttpClient.Builder()
    .addInterceptor(HttpLoggingInterceptor()) // 7계층: HTTP 요청/응답 로깅
    .addNetworkInterceptor { chain ->         // 네트워크 레벨 인터셉터
        val request = chain.request()
        println("URL: ${request.url}")        // 7계층: URL
        println("Host: ${request.url.host}")  // 3계층: DNS -> IP 변환 후
        chain.proceed(request)
    }
    .build()
```

### 네트워크 오류와 계층

```kotlin
try {
    val response = apiService.getUser("123")
} catch (e: UnknownHostException) {
    // 7계층(DNS) - 도메인을 IP로 변환 실패
} catch (e: ConnectException) {
    // 4계층(TCP) - 서버에 연결할 수 없음
} catch (e: SSLHandshakeException) {
    // 6계층(SSL/TLS) - 인증서 검증 실패
} catch (e: SocketTimeoutException) {
    // 4계층(TCP) - 연결 또는 읽기 타임아웃
} catch (e: HttpException) {
    // 7계층(HTTP) - 4xx, 5xx 응답
}
```

### 데이터 캡슐화

```
[HTTP 데이터]                          ← 7계층: 응용
[TCP 헤더 | HTTP 데이터]               ← 4계층: 전송 (세그먼트)
[IP 헤더 | TCP 헤더 | HTTP 데이터]     ← 3계층: 네트워크 (패킷)
[프레임 헤더 | IP | TCP | HTTP | FCS]  ← 2계층: 데이터링크 (프레임)
[비트 스트림]                          ← 1계층: 물리
```

### ConnectivityManager로 네트워크 상태 확인

```kotlin
val connectivityManager = getSystemService(ConnectivityManager::class.java)

val networkCallback = object : ConnectivityManager.NetworkCallback() {
    override fun onAvailable(network: Network) {
        // 네트워크 연결됨 (2~3계층)
    }

    override fun onCapabilitiesChanged(
        network: Network,
        capabilities: NetworkCapabilities
    ) {
        val hasInternet = capabilities
            .hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
        val isWifi = capabilities
            .hasTransport(NetworkCapabilities.TRANSPORT_WIFI)
        val isCellular = capabilities
            .hasTransport(NetworkCapabilities.TRANSPORT_CELLULAR)
    }

    override fun onLost(network: Network) {
        // 네트워크 끊김
    }
}

connectivityManager.registerDefaultNetworkCallback(networkCallback)
```

## 🔗 관련 개념

- [[00-CS-기초/네트워크/TCP-vs-UDP|TCP vs UDP (4계층)]]
- [[00-CS-기초/네트워크/HTTP-HTTPS|HTTP/HTTPS (7계층)]]
- [[00-CS-기초/네트워크/DNS|DNS (7계층)]]
- [[00-CS-기초/네트워크/SSL-TLS|SSL/TLS (6계층)]]

## 📚 더 보기

- [Android ConnectivityManager](https://developer.android.com/reference/android/net/ConnectivityManager)
- [OkHttp Interceptors](https://square.github.io/okhttp/features/interceptors/)

---

**핵심 요약:** OSI 7계층은 네트워크를 계층별로 분리한 모델. Android 네트워크 에러 디버깅 시 어떤 계층 문제인지 파악하면 빠르게 해결 가능.
