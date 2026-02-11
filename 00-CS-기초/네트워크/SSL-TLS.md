---
tags: [network, ssl, tls, https, security, cs-기초]
---

# SSL / TLS

## 💡 핵심 개념

**SSL(Secure Sockets Layer)**과 **TLS(Transport Layer Security)**는 네트워크 통신을 암호화하는 프로토콜이다. SSL은 더 이상 사용되지 않고 TLS가 후속 표준이지만, 관습적으로 "SSL"이라는 용어가 함께 쓰인다. HTTPS = HTTP + TLS이며, 데이터의 기밀성, 무결성, 인증을 보장한다.

## 📌 왜 필요한가?

Android 앱의 모든 API 통신은 TLS로 암호화되어야 한다. Certificate Pinning, Network Security Config 등 TLS 관련 설정을 이해해야 보안 취약점을 방지할 수 있다.

## 🔍 자세히

### SSL/TLS 버전 역사

```
SSL 2.0 (1995) → 취약, 사용 금지
SSL 3.0 (1996) → POODLE 공격, 사용 금지
TLS 1.0 (1999) → 지원 중단 중
TLS 1.1 (2006) → 지원 중단 중
TLS 1.2 (2008) → 현재 널리 사용
TLS 1.3 (2018) → 최신, 가장 안전하고 빠름

Android 지원:
- API 16+: TLS 1.0, 1.1, 1.2
- API 29+: TLS 1.3
```

### TLS 핸드셰이크 과정

```
클라이언트                              서버
    │                                    │
    │─── ClientHello ──────────────────>│
    │    (TLS 버전, 암호 스위트 목록,     │
    │     클라이언트 랜덤)                │
    │                                    │
    │<── ServerHello + 인증서 ─────────│
    │    (선택된 암호 스위트,             │
    │     서버 랜덤, X.509 인증서)       │
    │                                    │
    │    [인증서 검증]                    │
    │    - CA 서명 확인                   │
    │    - 만료일 확인                    │
    │    - 도메인 일치 확인               │
    │                                    │
    │─── 키 교환 (Pre-Master Secret) ──>│
    │                                    │
    │    [양쪽 모두 Master Secret 생성]   │
    │    [세션 키 파생]                   │
    │                                    │
    │─── Finished (암호화) ────────────>│
    │<── Finished (암호화) ────────────│
    │                                    │
    │<══ 암호화된 통신 시작 ═════════════>│
```

### TLS 1.3 개선점

```
TLS 1.2: 2-RTT 핸드셰이크 (왕복 2회)
TLS 1.3: 1-RTT 핸드셰이크 (왕복 1회)
         0-RTT 재연결 (이전 연결 재사용)

개선사항:
- 불안전한 암호 스위트 제거
- 핸드셰이크 전체 암호화
- 더 빠른 연결 설정
```

### Certificate Pinning (인증서 고정)

```kotlin
// 중간자(MITM) 공격 방지
// 서버 인증서의 공개키 해시를 앱에 고정
val client = OkHttpClient.Builder()
    .certificatePinner(
        CertificatePinner.Builder()
            // 메인 인증서 핀
            .add(
                "api.example.com",
                "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA="
            )
            // 백업 핀 (인증서 교체 대비)
            .add(
                "api.example.com",
                "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB="
            )
            .build()
    )
    .build()

// 핀 해시 확인 방법 (터미널):
// openssl s_client -connect api.example.com:443 | \
//   openssl x509 -pubkey -noout | \
//   openssl pkey -pubin -outform der | \
//   openssl dgst -sha256 -binary | \
//   openssl enc -base64
```

### Network Security Config로 TLS 설정

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <!-- Certificate Pinning (선언적 방식) -->
    <domain-config>
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set expiration="2025-12-31">
            <pin digest="SHA-256">AAAAAAAAAA...=</pin>
            <pin digest="SHA-256">BBBBBBBBBB...=</pin>
        </pin-set>
    </domain-config>

    <!-- 디버그 빌드에서 사용자 인증서 신뢰 (Charles Proxy 등) -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="user" />
            <certificates src="system" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

### 커스텀 TrustManager (주의 필요)

```kotlin
// 자체 서명 인증서를 사용하는 개발 환경 전용
// 프로덕션에서는 절대 인증서 검증을 비활성화하지 말 것!

// 올바른 방법: 특정 CA 인증서만 신뢰
fun createCustomCaSslClient(context: Context): OkHttpClient {
    val cf = CertificateFactory.getInstance("X.509")
    val caInput = context.resources.openRawResource(R.raw.my_ca)
    val ca = cf.generateCertificate(caInput)
    caInput.close()

    val keyStore = KeyStore.getInstance(KeyStore.getDefaultType())
    keyStore.load(null, null)
    keyStore.setCertificateEntry("ca", ca)

    val tmf = TrustManagerFactory.getInstance(
        TrustManagerFactory.getDefaultAlgorithm()
    )
    tmf.init(keyStore)

    val sslContext = SSLContext.getInstance("TLS")
    sslContext.init(null, tmf.trustManagers, null)

    return OkHttpClient.Builder()
        .sslSocketFactory(
            sslContext.socketFactory,
            tmf.trustManagers[0] as X509TrustManager
        )
        .build()
}
```

### SSL/TLS 관련 에러

```kotlin
try {
    val response = client.newCall(request).execute()
} catch (e: SSLHandshakeException) {
    // 원인: 인증서 만료, 자체 서명, 도메인 불일치
    // 해결: 서버 인증서 갱신, Network Security Config 확인
} catch (e: SSLPeerUnverifiedException) {
    // 원인: Certificate Pinning 실패
    // 해결: 핀 해시 업데이트
} catch (e: SSLProtocolException) {
    // 원인: TLS 버전 불일치
    // 해결: 서버/클라이언트 TLS 버전 맞춤
}
```

### OkHttp TLS 버전 설정

```kotlin
// 특정 TLS 버전만 허용
val spec = ConnectionSpec.Builder(ConnectionSpec.MODERN_TLS)
    .tlsVersions(TlsVersion.TLS_1_2, TlsVersion.TLS_1_3)
    .cipherSuites(
        CipherSuite.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
        CipherSuite.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
    )
    .build()

val client = OkHttpClient.Builder()
    .connectionSpecs(listOf(spec))
    .build()
```

## 🔗 관련 개념

- [[00-CS-기초/네트워크/HTTP-HTTPS|HTTP/HTTPS]]
- [[00-CS-기초/보안-기초/암호화-기초|암호화 기초]]
- [[00-CS-기초/보안-기초/대칭키-vs-비대칭키|대칭키 vs 비대칭키]]
- [[00-CS-기초/네트워크/OSI-7계층|OSI 7계층 (6계층)]]

## 📚 더 보기

- [Android Network Security Config](https://developer.android.com/privacy-and-security/security-config)
- [OkHttp HTTPS](https://square.github.io/okhttp/features/https/)
- [OWASP Certificate Pinning](https://owasp.org/www-community/controls/Certificate_and_Public_Key_Pinning)

---

**핵심 요약:** TLS는 HTTPS의 핵심 암호화 프로토콜. Android에서는 Network Security Config와 Certificate Pinning으로 보안을 강화하고, 인증서 검증을 절대 비활성화하지 않는다.
