# Activity

> Android 앱의 화면을 담당하는 핵심 컴포넌트

## Activity란?

- 사용자와 상호작용하는 단일 화면
- 앱의 진입점 (Entry Point)
- 생명주기를 가진 컴포넌트

## 기본 구조

```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        // 초기화 코드
    }
}
```

## Manifest 등록

```xml
<activity
    android:name=".MainActivity"
    android:exported="true"
    android:launchMode="standard">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

## Launch Mode

| Mode | 동작 |
|------|------|
| `standard` | 항상 새 인스턴스 생성 |
| `singleTop` | 스택 최상단에 있으면 재사용 |
| `singleTask` | 태스크 내 하나만 존재 |
| `singleInstance` | 독립된 태스크에 하나만 존재 |

## Intent로 시작

```kotlin
// 명시적 Intent
val intent = Intent(this, SecondActivity::class.java)
intent.putExtra("key", "value")
startActivity(intent)

// 암시적 Intent
val intent = Intent(Intent.ACTION_VIEW)
intent.data = Uri.parse("https://example.com")
startActivity(intent)
```

## 결과 받기 (Activity Result API)

```kotlin
private val launcher = registerForActivityResult(
    ActivityResultContracts.StartActivityForResult()
) { result ->
    if (result.resultCode == RESULT_OK) {
        val data = result.data
        // 결과 처리
    }
}

// 실행
launcher.launch(Intent(this, SecondActivity::class.java))
```

## Task와 Back Stack

- **Task**: 사용자가 수행하는 작업 단위
- **Back Stack**: Activity가 쌓이는 LIFO 구조
- 뒤로 가기 → 스택에서 pop

---

## 관련 문서
- [[03-Android-Framework/생명주기/Activity-Lifecycle|Activity Lifecycle]]
- [[03-Android-Framework/생명주기/Configuration-Changes|Configuration Changes]]

[← Android Framework](03-Android-Framework/README.md)