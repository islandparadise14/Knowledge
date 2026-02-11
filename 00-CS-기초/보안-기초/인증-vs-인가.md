---
tags: [CS, 보안, 인증, 인가, Authentication, Authorization]
---

# 인증 vs 인가

## 💡 핵심 개념

인증(Authentication)은 "너는 누구인가?"를 확인하는 과정이고, 인가(Authorization)는 "너는 무엇을 할 수 있는가?"를 결정하는 과정이다. 인증이 먼저 이루어지고, 그 결과를 바탕으로 인가가 수행된다. 두 개념은 반드시 구분해서 이해해야 한다.

## 📌 왜 필요한가?

안드로이드 앱에서 로그인(인증)과 권한 관리(인가)는 모든 서비스의 기본이다. Firebase Auth, OAuth, BiometricPrompt 등 인증 수단을 이해하고, API 접근 권한과 사용자 역할 기반 UI 분기 등 인가 로직을 올바르게 설계해야 보안 사고를 방지할 수 있다.

## 🔍 자세히

### 비교 표

| 구분 | 인증 (Authentication) | 인가 (Authorization) |
|------|----------------------|---------------------|
| 질문 | "너는 누구인가?" | "너는 무엇을 할 수 있는가?" |
| 시점 | 먼저 수행 | 인증 이후 수행 |
| 방법 | 비밀번호, 생체인식, 토큰 | 역할(Role), 권한(Permission), 스코프 |
| HTTP 코드 | 401 Unauthorized | 403 Forbidden |
| 예시 | 로그인 | 관리자 페이지 접근 |

### Android에서의 인증 방법들

```kotlin
// 1. BiometricPrompt - 생체 인증
class BiometricAuthManager(
    private val activity: FragmentActivity
) {
    fun authenticate(onSuccess: () -> Unit, onError: (String) -> Unit) {
        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("생체 인증")
            .setSubtitle("지문 또는 얼굴로 인증하세요")
            .setNegativeButtonText("취소")
            .setAllowedAuthenticators(BiometricManager.Authenticators.BIOMETRIC_STRONG)
            .build()

        val biometricPrompt = BiometricPrompt(
            activity,
            ContextCompat.getMainExecutor(activity),
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(
                    result: BiometricPrompt.AuthenticationResult
                ) {
                    onSuccess()
                }

                override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                    onError(errString.toString())
                }
            }
        )

        biometricPrompt.authenticate(promptInfo)
    }
}
```

### 생체 인증 + Keystore 바인딩

```kotlin
// 생체 인증 성공 시에만 키 사용 가능하도록 바인딩
fun createBiometricBoundKey() {
    val keyGenerator = KeyGenerator.getInstance(
        KeyProperties.KEY_ALGORITHM_AES,
        "AndroidKeyStore"
    )
    keyGenerator.init(
        KeyGenParameterSpec.Builder("bio_key",
            KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
        ).apply {
            setBlockModes(KeyProperties.BLOCK_MODE_GCM)
            setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
            setUserAuthenticationRequired(true) // 인증 필수
            setInvalidatedByBiometricEnrollment(true) // 새 생체 등록 시 무효화
        }.build()
    )
    keyGenerator.generateKey()
}
```

### 인가 - 역할 기반 접근 제어 (RBAC)

```kotlin
// 사용자 역할 정의
enum class UserRole {
    GUEST, USER, PREMIUM, ADMIN
}

// 인가 체크 유틸리티
class AuthorizationManager(
    private val userRepository: UserRepository
) {
    fun hasPermission(requiredRole: UserRole): Boolean {
        val currentRole = userRepository.getCurrentUserRole()
        return currentRole.ordinal >= requiredRole.ordinal
    }

    fun requireRole(requiredRole: UserRole) {
        if (!hasPermission(requiredRole)) {
            throw UnauthorizedException("${requiredRole.name} 권한이 필요합니다")
        }
    }
}

// ViewModel에서 인가 적용
class AdminViewModel(
    private val authManager: AuthorizationManager
) : ViewModel() {

    fun deleteUser(userId: String) {
        authManager.requireRole(UserRole.ADMIN)
        // 관리자만 실행 가능한 로직
    }
}
```

### API 레벨에서의 인증/인가

```kotlin
// OkHttp Interceptor로 토큰 자동 첨부 (인증)
class AuthInterceptor(
    private val tokenManager: TokenManager
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val token = tokenManager.getAccessToken()
        val request = chain.request().newBuilder()
            .addHeader("Authorization", "Bearer $token")
            .build()

        val response = chain.proceed(request)

        return when (response.code) {
            401 -> handleUnauthorized(chain, request) // 인증 실패 → 토큰 갱신
            403 -> throw ForbiddenException()          // 인가 실패 → 권한 없음
            else -> response
        }
    }

    private fun handleUnauthorized(
        chain: Interceptor.Chain,
        originalRequest: Request
    ): Response {
        val newToken = tokenManager.refreshToken()
        val newRequest = originalRequest.newBuilder()
            .header("Authorization", "Bearer $newToken")
            .build()
        return chain.proceed(newRequest)
    }
}
```

### 흔한 실수

1. **클라이언트에서만 인가 처리**: 서버에서도 반드시 권한 검증 필요
2. **401과 403 혼용**: 인증 실패(401)와 인가 실패(403)를 명확히 구분
3. **토큰에 과도한 정보 저장**: JWT에 민감 정보를 넣지 않기

## 🔗 관련 개념

- [[00-CS-기초/보안-기초/OAuth-JWT|OAuth와 JWT]]
- [[00-CS-기초/보안-기초/암호화-기초|암호화 기초]]
- [[00-CS-기초/보안-기초/해싱|해싱]]
- [[00-CS-기초/네트워크/쿠키-세션-토큰|쿠키, 세션, 토큰]]

## 📚 더 보기

- [Android BiometricPrompt 가이드](https://developer.android.com/training/sign-in/biometric-auth)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)

---

**핵심 요약:** 인증은 사용자 신원 확인(401), 인가는 접근 권한 확인(403)이며, 안드로이드에서는 BiometricPrompt로 로컬 인증을, OAuth/JWT로 서버 인증과 인가를 처리한다.
