---
tags: [CS, 보안, 암호화, 대칭키, 비대칭키, AES, RSA]
---

# 대칭키 vs 비대칭키

## 💡 핵심 개념

대칭키(Symmetric Key) 암호화는 암호화와 복호화에 동일한 키를 사용하는 방식이고, 비대칭키(Asymmetric Key) 암호화는 공개키(Public Key)와 개인키(Private Key) 쌍을 사용하는 방식이다. 실무에서는 두 방식을 조합하여 사용하는 하이브리드 암호화가 일반적이다.

## 📌 왜 필요한가?

안드로이드 앱에서 HTTPS 통신은 비대칭키로 키를 교환한 뒤 대칭키로 데이터를 암호화한다. 로컬 데이터 암호화(EncryptedSharedPreferences)에는 대칭키(AES)를, 디지털 서명이나 키 교환에는 비대칭키(RSA/ECDSA)를 사용한다.

## 🔍 자세히

### 비교 표

| 특성 | 대칭키 | 비대칭키 |
|------|--------|----------|
| 키 개수 | 1개 (동일 키) | 2개 (공개키 + 개인키) |
| 속도 | 빠름 | 느림 (100~1000배) |
| 대표 알고리즘 | AES, ChaCha20 | RSA, ECDSA, Ed25519 |
| 키 길이 | 128/256비트 | 2048/4096비트(RSA) |
| 주요 용도 | 데이터 암호화 | 키 교환, 디지털 서명 |
| 키 분배 문제 | 있음 | 없음 (공개키 공유) |

### 대칭키 암호화 (AES-GCM)

```kotlin
// Android Keystore에서 AES 대칭키 생성
fun generateAesKey(): SecretKey {
    val keyGenerator = KeyGenerator.getInstance(
        KeyProperties.KEY_ALGORITHM_AES,
        "AndroidKeyStore"
    )
    keyGenerator.init(
        KeyGenParameterSpec.Builder("aes_key",
            KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
        ).apply {
            setBlockModes(KeyProperties.BLOCK_MODE_GCM)
            setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
            setKeySize(256)
        }.build()
    )
    return keyGenerator.generateKey()
}

// 암호화
fun encryptWithAes(plaintext: String, key: SecretKey): EncryptedData {
    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    cipher.init(Cipher.ENCRYPT_MODE, key)
    return EncryptedData(
        iv = cipher.iv,
        ciphertext = cipher.doFinal(plaintext.toByteArray())
    )
}

data class EncryptedData(val iv: ByteArray, val ciphertext: ByteArray)
```

### 비대칭키 암호화 (RSA)

```kotlin
// Android Keystore에서 RSA 키쌍 생성
fun generateRsaKeyPair(): KeyPair {
    val keyPairGenerator = KeyPairGenerator.getInstance(
        KeyProperties.KEY_ALGORITHM_RSA,
        "AndroidKeyStore"
    )
    keyPairGenerator.initialize(
        KeyGenParameterSpec.Builder("rsa_key",
            KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
                or KeyProperties.PURPOSE_SIGN or KeyProperties.PURPOSE_VERIFY
        ).apply {
            setDigests(KeyProperties.DIGEST_SHA256)
            setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_RSA_OAEP)
            setSignaturePaddings(KeyProperties.SIGNATURE_PADDING_RSA_PSS)
            setKeySize(2048)
        }.build()
    )
    return keyPairGenerator.generateKeyPair()
}

// 공개키로 암호화 → 개인키로 복호화
fun encryptWithRsa(data: ByteArray, publicKey: PublicKey): ByteArray {
    val cipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding")
    cipher.init(Cipher.ENCRYPT_MODE, publicKey)
    return cipher.doFinal(data)
}
```

### 하이브리드 암호화 (TLS/HTTPS 동작 원리)

```
1. 클라이언트 → 서버: 연결 요청 (Client Hello)
2. 서버 → 클라이언트: 서버 인증서 (공개키 포함)
3. 클라이언트: 대칭키(세션 키) 생성 → 서버 공개키로 암호화하여 전송
4. 이후 통신: 대칭키(세션 키)로 데이터 암호화/복호화
```

비대칭키로 안전하게 대칭키를 교환하고, 이후 빠른 대칭키로 실제 데이터를 암호화하는 방식이다.

### 디지털 서명 (비대칭키 활용)

```kotlin
// 개인키로 서명
fun sign(data: ByteArray, privateKey: PrivateKey): ByteArray {
    val signature = Signature.getInstance("SHA256withRSA/PSS")
    signature.initSign(privateKey)
    signature.update(data)
    return signature.sign()
}

// 공개키로 검증
fun verify(data: ByteArray, signatureBytes: ByteArray, publicKey: PublicKey): Boolean {
    val signature = Signature.getInstance("SHA256withRSA/PSS")
    signature.initVerify(publicKey)
    signature.update(data)
    return signature.verify(signatureBytes)
}
```

### Android에서의 선택 기준

- **로컬 데이터 암호화** → AES-256-GCM (대칭키)
- **서버와 키 교환** → RSA 또는 ECDH (비대칭키)
- **APK 서명** → RSA/ECDSA (비대칭키)
- **BiometricPrompt 연동** → Keystore 대칭키 + 생체 인증 바인딩

## 🔗 관련 개념

- [[00-CS-기초/보안-기초/암호화-기초|암호화 기초]]
- [[00-CS-기초/보안-기초/해싱|해싱]]
- [[00-CS-기초/보안-기초/OAuth-JWT|OAuth와 JWT]]
- [[00-CS-기초/네트워크/SSL-TLS|SSL/TLS]]

## 📚 더 보기

- [Android Keystore Key Generation](https://developer.android.com/training/articles/keystore#GeneratingANewPrivateKey)
- [AES-GCM 설명 (NIST)](https://csrc.nist.gov/publications/detail/sp/800-38d/final)

---

**핵심 요약:** 대칭키는 빠르지만 키 분배가 어렵고, 비대칭키는 느리지만 키 분배가 안전하므로, 실무에서는 비대칭키로 대칭키를 교환하는 하이브리드 방식을 사용한다.
