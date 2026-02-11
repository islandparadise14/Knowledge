---
tags: [network, rest, api, retrofit, cs-기초]
---

# REST API

## 💡 핵심 개념

**REST(Representational State Transfer)**는 HTTP 기반의 아키텍처 스타일로, URL로 리소스를 식별하고 HTTP 메서드로 행위를 정의한다. 상태를 서버에 저장하지 않는(Stateless) 특성이 있으며, Android 앱의 서버 통신에서 가장 널리 사용되는 방식이다.

## 📌 왜 필요한가?

Android 앱의 대부분은 REST API로 서버와 통신한다. Retrofit을 사용해 REST API를 타입 안전하게 호출하는 것은 Android 개발의 핵심 역량이다.

## 🔍 자세히

### REST 6가지 원칙

```
1. Client-Server    : 클라이언트와 서버 분리
2. Stateless        : 각 요청은 독립적 (서버가 상태 저장 X)
3. Cacheable        : 응답은 캐싱 가능
4. Uniform Interface: 일관된 인터페이스
5. Layered System   : 계층화된 구조
6. Code on Demand   : (선택) 서버에서 코드 전송 가능
```

### RESTful URL 설계

```
좋은 예:
GET    /api/users          → 사용자 목록 조회
GET    /api/users/123      → 특정 사용자 조회
POST   /api/users          → 사용자 생성
PUT    /api/users/123      → 사용자 전체 수정
PATCH  /api/users/123      → 사용자 부분 수정
DELETE /api/users/123      → 사용자 삭제
GET    /api/users/123/posts → 특정 사용자의 게시물 목록

나쁜 예:
GET    /api/getUser?id=123
POST   /api/deleteUser
GET    /api/user_list
```

### Retrofit 인터페이스 정의

```kotlin
interface UserApiService {

    // GET - 목록 조회 (쿼리 파라미터)
    @GET("api/users")
    suspend fun getUsers(
        @Query("page") page: Int,
        @Query("size") size: Int = 20,
        @Query("sort") sort: String = "createdAt,desc"
    ): Response<PagedResponse<User>>

    // GET - 단건 조회 (경로 파라미터)
    @GET("api/users/{id}")
    suspend fun getUser(
        @Path("id") userId: String
    ): Response<User>

    // POST - 생성 (요청 바디)
    @POST("api/users")
    suspend fun createUser(
        @Body request: CreateUserRequest
    ): Response<User>

    // PUT - 전체 수정
    @PUT("api/users/{id}")
    suspend fun updateUser(
        @Path("id") userId: String,
        @Body request: UpdateUserRequest
    ): Response<User>

    // PATCH - 부분 수정
    @PATCH("api/users/{id}")
    suspend fun patchUser(
        @Path("id") userId: String,
        @Body fields: Map<String, @JvmSuppressWildcards Any>
    ): Response<User>

    // DELETE - 삭제
    @DELETE("api/users/{id}")
    suspend fun deleteUser(
        @Path("id") userId: String
    ): Response<Unit>

    // 파일 업로드 (Multipart)
    @Multipart
    @POST("api/users/{id}/avatar")
    suspend fun uploadAvatar(
        @Path("id") userId: String,
        @Part image: MultipartBody.Part
    ): Response<AvatarResponse>

    // 헤더 추가
    @Headers("Cache-Control: max-age=600")
    @GET("api/config")
    suspend fun getConfig(): Response<AppConfig>
}
```

### Retrofit 설정

```kotlin
// Retrofit 인스턴스 생성
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .client(okHttpClient)
    .addConverterFactory(MoshiConverterFactory.create())
    .build()

val apiService = retrofit.create(UserApiService::class.java)

// OkHttp 클라이언트 설정
val okHttpClient = OkHttpClient.Builder()
    .addInterceptor(AuthInterceptor(tokenManager))
    .addInterceptor(HttpLoggingInterceptor().apply {
        level = if (BuildConfig.DEBUG)
            HttpLoggingInterceptor.Level.BODY
        else
            HttpLoggingInterceptor.Level.NONE
    })
    .connectTimeout(15, TimeUnit.SECONDS)
    .readTimeout(15, TimeUnit.SECONDS)
    .build()
```

### 응답 처리 패턴

```kotlin
// sealed class로 API 결과 래핑
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val code: Int, val message: String) : ApiResult<Nothing>()
    data class Exception(val e: Throwable) : ApiResult<Nothing>()
}

// 안전한 API 호출 확장 함수
suspend fun <T> safeApiCall(
    call: suspend () -> Response<T>
): ApiResult<T> {
    return try {
        val response = call()
        if (response.isSuccessful) {
            ApiResult.Success(response.body()!!)
        } else {
            ApiResult.Error(response.code(), response.message())
        }
    } catch (e: IOException) {
        ApiResult.Exception(e)
    }
}

// 사용
class UserRepository(private val api: UserApiService) {
    suspend fun getUser(id: String): ApiResult<User> =
        safeApiCall { api.getUser(id) }
}
```

### Ktor를 사용한 REST 호출

```kotlin
// Ktor 클라이언트 설정
val client = HttpClient(Android) {
    install(ContentNegotiation) {
        json(Json {
            ignoreUnknownKeys = true
            prettyPrint = true
        })
    }
    install(Logging) {
        level = LogLevel.BODY
    }
    defaultRequest {
        url("https://api.example.com/")
        contentType(ContentType.Application.Json)
    }
}

// API 호출
suspend fun getUser(id: String): User {
    return client.get("api/users/$id").body()
}

suspend fun createUser(request: CreateUserRequest): User {
    return client.post("api/users") {
        setBody(request)
    }.body()
}
```

## 🔗 관련 개념

- [[00-CS-기초/네트워크/HTTP-HTTPS|HTTP/HTTPS]]
- [[00-CS-기초/네트워크/쿠키-세션-토큰|쿠키-세션-토큰 (인증)]]
- [[00-CS-기초/네트워크/WebSocket|WebSocket (실시간 통신)]]

## 📚 더 보기

- [Retrofit 공식 문서](https://square.github.io/retrofit/)
- [Ktor Client](https://ktor.io/docs/client-create-and-configure.html)

---

**핵심 요약:** REST API는 HTTP 메서드 + URL로 리소스를 조작하는 아키텍처. Retrofit 인터페이스로 타입 안전하게 정의하고, sealed class로 결과를 처리한다.
