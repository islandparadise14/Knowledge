---
tags: [CS, 보안, OAuth, JWT, 토큰, 인증]
---

# OAuth와 JWT

## 💡 핵심 개념

OAuth 2.0은 제3자 애플리케이션이 사용자의 리소스에 안전하게 접근할 수 있도록 하는 인가 프레임워크이다. JWT(JSON Web Token)는 당사자 간에 JSON 객체로 정보를 안전하게 전달하기 위한 토큰 형식이다. OAuth는 "프로토콜"이고 JWT는 "토큰 형식"으로, 서로 다른 계층의 개념이지만 함께 사용되는 경우가 많다.

## 📌 왜 필요한가?

안드로이드 앱에서 구글/카카오/네이버 소셜 로그인은 모두 OAuth 2.0 기반이다. 서버와의 인증에는 JWT 기반 Access Token/Refresh Token 패턴이 표준이므로, 두 개념을 명확히 이해해야 안전한 인증 시스템을 구현할 수 있다.

## 🔍 자세히

### OAuth 2.0 흐름 (Authorization Code + PKCE)

안드로이드 앱은 **Public Client**이므로 client_secret을 안전하게 저장할 수 없다. 따라서 **PKCE(Proof Key for Code Exchange)** 확장을 반드시 사용해야 한다.

```
1. 앱: code_verifier(랜덤) 생성 → SHA-256 해싱 → code_challenge
2. 앱 → 인증서버: 인가 요청 (code_challenge 포함)
3. 사용자: 브라우저에서 로그인 및 권한 동의
4. 인증서버 → 앱: Authorization Code 반환
5. 앱 → 인증서버: Code + code_verifier로 토큰 요청
6. 인증서버: code_verifier를 SHA-256 해싱하여 code_challenge와 비교
7. 인증서버 → 앱: Access Token + Refresh Token 발급
```

### AppAuth 라이브러리로 OAuth 구현

```kotlin
// build.gradle
// implementation("net.openid:appauth:0.11.1")

class OAuthManager(private val context: Context) {

    private val serviceConfig = AuthorizationServiceConfiguration(
        Uri.parse("https://accounts.google.com/o/oauth2/v2/auth"), // 인가 엔드포인트
        Uri.parse("https://oauth2.googleapis.com/token")           // 토큰 엔드포인트
    )

    fun createAuthRequest(): AuthorizationRequest {
        return AuthorizationRequest.Builder(
            serviceConfig,
            "your_client_id",
            ResponseTypeValues.CODE,
            Uri.parse("com.yourapp:/oauth2callback") // 리다이렉트 URI
        ).apply {
            setScope("openid profile email")
            setCodeVerifier() // PKCE 자동 생성
        }.build()
    }

    fun startAuth(activity: Activity) {
        val authService = AuthorizationService(context)
        val authRequest = createAuthRequest()
        val authIntent = authService.getAuthorizationRequestIntent(authRequest)
        activity.startActivityForResult(authIntent, RC_AUTH)
    }
}
```

### JWT 구조

JWT는 Header, Payload, Signature 세 부분으로 구성되며, 점(.)으로 구분된다.

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0.dozjgNryP4J3jVmNHl0w5N_XgL0n3I9PlFUP0THsR8U
[    Header    ]     [       Payload        ]     [          Signature          ]
```

```kotlin
// JWT 디코딩 (검증은 서버에서!)
fun decodeJwtPayload(jwt: String): JSONObject {
    val parts = jwt.split(".")
    require(parts.size == 3) { "Invalid JWT format" }

    val payload = String(
        Base64.decode(parts[1], Base64.URL_SAFE or Base64.NO_PADDING),
        Charsets.UTF_8
    )
    return JSONObject(payload)
}

// Payload 예시
// {
//   "sub": "user123",
//   "name": "홍길동",
//   "iat": 1704067200,
//   "exp": 1704070800,
//   "roles": ["user", "premium"]
// }
```

### 토큰 저장 베스트 프랙티스

```kotlin
class SecureTokenManager(context: Context) {

    // EncryptedSharedPreferences로 토큰 저장
    private val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()

    private val securePrefs = EncryptedSharedPreferences.create(
        context,
        "auth_tokens",
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )

    fun saveTokens(accessToken: String, refreshToken: String) {
        securePrefs.edit()
            .putString("access_token", accessToken)
            .putString("refresh_token", refreshToken)
            .putLong("token_saved_at", System.currentTimeMillis())
            .apply()
    }

    fun getAccessToken(): String? = securePrefs.getString("access_token", null)
    fun getRefreshToken(): String? = securePrefs.getString("refresh_token", null)

    fun clearTokens() {
        securePrefs.edit().clear().apply()
    }
}
```

### Access Token 갱신 (Silent Refresh)

```kotlin
class TokenAuthenticator(
    private val tokenManager: SecureTokenManager,
    private val authApi: AuthApi
) : Authenticator {

    override fun authenticate(route: Route?, response: Response): Request? {
        // 401 응답 시 자동으로 호출됨
        if (response.code != 401) return null

        // 무한 루프 방지
        if (response.request.header("X-Retry") != null) {
            tokenManager.clearTokens()
            return null // 로그아웃 처리
        }

        val refreshToken = tokenManager.getRefreshToken() ?: return null

        // 동기적으로 토큰 갱신
        val newTokens = runBlocking {
            authApi.refreshToken(RefreshRequest(refreshToken))
        }

        tokenManager.saveTokens(newTokens.accessToken, newTokens.refreshToken)

        return response.request.newBuilder()
            .header("Authorization", "Bearer ${newTokens.accessToken}")
            .header("X-Retry", "true")
            .build()
    }
}
```

### 토큰 저장 시 하지 말아야 할 것

| 저장 방식 | 안전성 | 이유 |
|-----------|--------|------|
| 일반 SharedPreferences | 위험 | 루팅 시 평문 노출 |
| 일반 파일 | 위험 | 다른 앱에서 접근 가능 |
| SQLite (평문) | 위험 | DB 파일 탈취 가능 |
| EncryptedSharedPreferences | 안전 | Keystore 기반 암호화 |
| AccountManager | 안전 | 시스템 레벨 보안 |

## 🔗 관련 개념

- [[00-CS-기초/보안-기초/인증-vs-인가|인증 vs 인가]]
- [[00-CS-기초/보안-기초/암호화-기초|암호화 기초]]
- [[00-CS-기초/보안-기초/해싱|해싱]]
- [[00-CS-기초/네트워크/쿠키-세션-토큰|쿠키, 세션, 토큰]]
- [[00-CS-기초/네트워크/HTTP-HTTPS|HTTP/HTTPS]]

## 📚 더 보기

- [OAuth 2.0 for Mobile Apps (RFC 8252)](https://datatracker.ietf.org/doc/html/rfc8252)
- [PKCE (RFC 7636)](https://datatracker.ietf.org/doc/html/rfc7636)
- [AppAuth for Android](https://github.com/openid/AppAuth-Android)
- [JWT.io](https://jwt.io/)

---

**핵심 요약:** OAuth 2.0은 소셜 로그인 등 제3자 인가 프레임워크이고 JWT는 토큰 형식이며, 안드로이드에서는 PKCE를 필수로 사용하고 토큰은 반드시 EncryptedSharedPreferences에 저장해야 한다.
