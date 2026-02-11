---
tags: [network, dns, domain, cs-기초]
---

# DNS

## 💡 핵심 개념

**DNS(Domain Name System)**는 도메인 이름(예: api.example.com)을 IP 주소(예: 93.184.216.34)로 변환하는 시스템이다. 인터넷의 전화번호부 역할을 하며, 모든 네트워크 요청의 첫 단계에서 동작한다.

## 📌 왜 필요한가?

Android 앱에서 API 호출 시 DNS 조회가 가장 먼저 일어난다. DNS 실패는 `UnknownHostException`을 발생시키며, DNS 조회 속도는 앱의 네트워크 성능에 직접 영향을 준다.

## 🔍 자세히

### DNS 조회 과정

```
1. 앱에서 api.example.com 요청
   │
2. 로컬 DNS 캐시 확인 (디바이스)
   │ (없으면)
3. 재귀 DNS 서버에 질의 (ISP 또는 설정된 DNS)
   │ (없으면)
4. 루트 네임서버 → .com 네임서버 → example.com 네임서버
   │
5. IP 주소 반환: 93.184.216.34
   │
6. 캐시에 저장 (TTL 동안)
```

### DNS 레코드 타입

| 타입 | 설명 | 예시 |
|------|------|------|
| A | IPv4 주소 | `93.184.216.34` |
| AAAA | IPv6 주소 | `2606:2800:220:1:...` |
| CNAME | 별칭 도메인 | `www.example.com → example.com` |
| MX | 메일 서버 | `mail.example.com` |
| TXT | 텍스트 정보 | SPF, DKIM 검증 |
| NS | 네임 서버 | `ns1.example.com` |

### OkHttp에서 DNS 커스터마이징

```kotlin
// 커스텀 DNS 설정
val customDns = object : Dns {
    override fun lookup(hostname: String): List<InetAddress> {
        return try {
            // 시스템 DNS 사용
            Dns.SYSTEM.lookup(hostname)
        } catch (e: UnknownHostException) {
            // 폴백: 직접 조회
            listOf(InetAddress.getByName(hostname))
        }
    }
}

val client = OkHttpClient.Builder()
    .dns(customDns)
    .build()
```

### DNS over HTTPS (DoH)

```kotlin
// OkHttp DNS over HTTPS - 보안 DNS 조회
// DNS 조회 자체도 암호화하여 ISP 등의 감청 방지
val bootstrapClient = OkHttpClient.Builder()
    .protocols(listOf(Protocol.HTTP_2, Protocol.HTTP_1_1))
    .build()

val dns = DnsOverHttps.Builder()
    .client(bootstrapClient)
    .url("https://dns.google/dns-query".toHttpUrl())
    .bootstrapDnsHosts(
        InetAddress.getByName("8.8.4.4"),
        InetAddress.getByName("8.8.8.8")
    )
    .build()

val client = OkHttpClient.Builder()
    .dns(dns)
    .build()
```

### Android Private DNS (API 28+)

```
Android 9+에서는 시스템 설정에서 Private DNS를 설정할 수 있다.
설정 → 네트워크 → Private DNS → dns.google 등 입력

이 설정은 앱 레벨이 아닌 시스템 레벨에서 적용되며,
DNS over TLS (DoT)를 사용한다.
```

### DNS 캐싱과 성능

```kotlin
// OkHttp는 내부적으로 DNS 캐싱을 하지 않음
// 커스텀 DNS 캐시 구현
class CachingDns(private val ttlMillis: Long = 5 * 60 * 1000) : Dns {
    private val cache = ConcurrentHashMap<String, CacheEntry>()

    data class CacheEntry(
        val addresses: List<InetAddress>,
        val expireTime: Long
    )

    override fun lookup(hostname: String): List<InetAddress> {
        val cached = cache[hostname]
        if (cached != null && cached.expireTime > System.currentTimeMillis()) {
            return cached.addresses
        }

        val addresses = Dns.SYSTEM.lookup(hostname)
        cache[hostname] = CacheEntry(
            addresses = addresses,
            expireTime = System.currentTimeMillis() + ttlMillis
        )
        return addresses
    }
}

val client = OkHttpClient.Builder()
    .dns(CachingDns(ttlMillis = 10 * 60 * 1000)) // 10분 캐시
    .build()
```

### DNS 관련 에러 처리

```kotlin
try {
    val response = apiService.getData()
} catch (e: UnknownHostException) {
    // DNS 조회 실패
    // 원인: 인터넷 미연결, 잘못된 도메인, DNS 서버 장애
    when {
        !isNetworkAvailable() -> showError("인터넷에 연결되어 있지 않습니다")
        else -> showError("서버를 찾을 수 없습니다. 잠시 후 다시 시도해주세요")
    }
}

private fun isNetworkAvailable(): Boolean {
    val cm = getSystemService(ConnectivityManager::class.java)
    val capabilities = cm.getNetworkCapabilities(cm.activeNetwork)
    return capabilities?.hasCapability(
        NetworkCapabilities.NET_CAPABILITY_INTERNET
    ) == true
}
```

## 🔗 관련 개념

- [[00-CS-기초/네트워크/OSI-7계층|OSI 7계층]]
- [[00-CS-기초/네트워크/HTTP-HTTPS|HTTP/HTTPS]]
- [[00-CS-기초/네트워크/TCP-vs-UDP|TCP vs UDP (DNS는 주로 UDP 사용)]]

## 📚 더 보기

- [OkHttp DNS](https://square.github.io/okhttp/features/dns/)
- [Android Private DNS](https://developer.android.com/privacy-and-security/security-config)

---

**핵심 요약:** DNS는 도메인을 IP로 변환하는 시스템. OkHttp에서 커스텀 DNS, DNS over HTTPS로 성능과 보안을 개선할 수 있다.
