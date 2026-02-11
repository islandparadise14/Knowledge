---
tags: [network, http, https, tls, android-security, cs-기초]
---

# HTTP / HTTPS

## 💡 핵심 개념

**HTTP(HyperText Transfer Protocol)**는 웹에서 데이터를 주고받는 프로토콜이고, **HTTPS**는 HTTP에 SSL/TLS 암호화를 추가한 보안 프로토콜이다. Android 9(API 28)부터 기본적으로 cleartext(HTTP) 트래픽이 차단되므로 HTTPS 사용이 사실상 필수다.

## 📌 왜 필요한가?

Android 앱의 모든 네트워크 통신은 HTTPS가 기본이다. Network Security Config를 이해해야 디버깅 환경 설정과 보안 정책을 올바르게 구성할 수 있다.

## 🔍 자세히

### HTTP 요청/응답 구조

```
요청 (Request):
┌─────────────────────────────────────┐
│ GET /api/users/123 HTTP/1.1         │  ← 요청 라인 (메서드, 경로, 버전)
│ Host: api.example.com               │  ← 헤더
│ Authorization: Bearer eyJhbGci...   │
│ Accept: application/json            │
│ Content-Type: application/json      │
│                                     │
│ {"name": "홍길동"}                   │  ← 바디 (POST/PUT일 때)
└─────────────────────────────────────┘

응답 (Response):
┌─────────────────────────────────────┐
│ HTTP/1.1 200 OK                     │  ← 상태 라인
│ Content-Type: application/json      │  ← 헤더
│ Cache-Control: max-age=3600         │
│                                     │
│ {"id": 123, "name": "홍길동"}       │  ← 바디
└─────────────────────────────────────┘
```

### HTTP 메서드

| 메서드 | 용도 | 멱등성 | 안전 | Retrofit 예시 |
|--------|------|--------|------|--------------|
| GET | 조회 | O | O | `@GET` |
| POST | 생성 | X | X | `@POST` |
| PUT | 전체 수정 | O | X | `@PUT` |
| PATCH | 부분 수정 | X | X | `@PATCH` |
| DELETE | 삭제 | O | X | `@DELETE` |

### HTTP 상태 코드

```kotlin
// Retrofit에서 상태 코드 처리
val response = apiService.getUser("123")

when (response.code()) {
    200 -> { /* OK - 성공 */ }
    201 -> { /* Created - 리소스 생성 */ }
    204 -> { /* No Content - 성공, 바디 없음 */ }
    400 -> { /* Bad Request - 잘못된 요청 */ }
    401 -> { /* Unauthorized - 인증 필요 */ }
    403 -> { /* Forbidden - 권한 없음 */ }
    404 -> { /* Not Found - 리소스 없음 */ }
    429 -> { /* Too Many Requests - 요청 제한 초과 */ }
    500 -> { /* Internal Server Error - 서버 오류 */ }
    503 -> { /* Service Unavailable - 서비스 점검 */ }
}
```

### HTTPS 동작 과정

```
클라이언트(앱)                       서버
    │                                 │
    │──── ClientHello ───────────────>│  1. TLS 버전, 암호 스위트 제안
    │<─── ServerHello + 인증서 ──────│  2. 인증서 전달
    │                                 │
    │  [인증서 검증 - CA 체인 확인]    │  3. 신뢰할 수 있는 인증서인지
    │                                 │
    │──── 키 교환 ──────────────────>│  4. 대칭키 생성을 위한 교환
    │<─── Finished ─────────────────│  5. 핸드셰이크 완료
    │                                 │
    │<═══ 암호화된 데이터 통신 ═══════>│  6. 대칭키로 암호화 통신
```

### Android cleartext 트래픽 정책

```xml
<!-- AndroidManifest.xml -->
<!-- API 28+ 에서 HTTP 사용 시 필요 (권장하지 않음) -->
<application
    android:usesCleartextTraffic="true">
</application>
```

### Network Security Config (권장 방식)

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <!-- 디버그 빌드에서만 HTTP 허용 -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="user" />  <!-- 사용자 설치 인증서 신뢰 -->
        </trust-anchors>
    </debug-overrides>

    <!-- 특정 도메인만 HTTP 허용 -->
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">localhost</domain>
        <domain includeSubdomains="true">10.0.2.2</domain>
    </domain-config>

    <!-- 기본: HTTPS만 허용 -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

```xml
<!-- AndroidManifest.xml에서 참조 -->
<application
    android:networkSecurityConfig="@xml/network_security_config">
</application>
```

### OkHttp에서 HTTPS 설정

```kotlin
// 커스텀 TLS 설정
val client = OkHttpClient.Builder()
    .connectionSpecs(listOf(
        ConnectionSpec.MODERN_TLS,  // TLS 1.2+ 만 허용
        ConnectionSpec.CLEARTEXT    // 개발용 HTTP 허용
    ))
    .build()

// Certificate Pinning - 중간자 공격 방지
val client = OkHttpClient.Builder()
    .certificatePinner(
        CertificatePinner.Builder()
            .add("api.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
            .add("api.example.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=")
            .build()
    )
    .build()
```

### HTTP/1.1 vs HTTP/2

```
HTTP/1.1:
- 하나의 연결에 하나의 요청/응답
- Head-of-line 블로킹 문제
- 텍스트 기반 프로토콜

HTTP/2 (OkHttp 기본 지원):
- 멀티플렉싱: 하나의 연결에 여러 요청 동시 처리
- 헤더 압축 (HPACK)
- 서버 푸시
- 바이너리 프로토콜
```

```kotlin
// OkHttp는 자동으로 HTTP/2 협상
val client = OkHttpClient.Builder()
    .protocols(listOf(Protocol.HTTP_2, Protocol.HTTP_1_1))
    .build()
```

## 🔗 관련 개념

- [[00-CS-기초/네트워크/SSL-TLS|SSL/TLS]]
- [[00-CS-기초/네트워크/REST-API|REST API]]
- [[00-CS-기초/네트워크/TCP-vs-UDP|TCP vs UDP]]
- [[00-CS-기초/보안-기초/암호화-기초|암호화 기초]]

## 📚 더 보기

- [Android Network Security Config](https://developer.android.com/privacy-and-security/security-config)
- [OkHttp HTTPS](https://square.github.io/okhttp/features/https/)

---

**핵심 요약:** Android 9+는 HTTPS 필수. Network Security Config로 도메인별 보안 정책 설정. OkHttp는 HTTP/2, Certificate Pinning을 기본 지원.
