---
tags: [android, content-provider, 4대컴포넌트]
---

# ContentProvider

## 💡 핵심 개념

앱 간 데이터를 공유하기 위한 컴포넌트. 데이터에 접근하는 표준 인터페이스를 제공하며, 연락처, 미디어, 캘린더 등 시스템 데이터도 ContentProvider를 통해 접근한다.

## 📌 왜 필요한가?

Android에서 앱은 샌드박스로 격리되어 있어 다른 앱의 데이터에 직접 접근할 수 없다. ContentProvider는 이 격리를 유지하면서도 명시적으로 허용된 데이터를 안전하게 공유할 수 있는 표준 방법을 제공한다.

## 🔍 자세히

### ContentProvider가 필요한 경우 vs 아닌 경우

```
필요한 경우:
  - 다른 앱에 데이터를 제공해야 할 때
  - 시스템 연락처, 미디어 파일에 접근할 때
  - FileProvider로 파일을 다른 앱에 공유할 때

필요 없는 경우:
  - 앱 내부에서만 DB를 사용할 때 (Room 직접 사용)
  - 앱 내부에서만 SharedPreferences를 쓸 때
```

### URI 구조

ContentProvider는 URI로 데이터를 식별한다.

```
content://com.example.app.provider/users/123
    ↑           ↑                    ↑    ↑
 scheme      authority             path  id

content://    — ContentProvider 스킴
authority     — 앱의 고유 식별자 (보통 패키지명)
path          — 데이터 테이블
id            — 특정 행 (없으면 전체)
```

### ContentProvider 구현

```kotlin
class UserProvider : ContentProvider() {

    private lateinit var db: UserDatabase

    companion object {
        const val AUTHORITY = "com.example.app.provider"
        val CONTENT_URI: Uri = Uri.parse("content://$AUTHORITY/users")

        private val uriMatcher = UriMatcher(UriMatcher.NO_MATCH).apply {
            addURI(AUTHORITY, "users", 1)      // 전체 목록
            addURI(AUTHORITY, "users/#", 2)    // 특정 id
        }
    }

    override fun onCreate(): Boolean {
        db = UserDatabase.getInstance(context!!)
        return true
    }

    override fun query(
        uri: Uri, projection: Array<String>?, selection: String?,
        selectionArgs: Array<String>?, sortOrder: String?
    ): Cursor? {
        return when (uriMatcher.match(uri)) {
            1 -> db.userDao().getAllCursor()
            2 -> db.userDao().getByIdCursor(uri.lastPathSegment)
            else -> null
        }
    }

    override fun insert(uri: Uri, values: ContentValues?): Uri? {
        val id = db.userDao().insert(values)
        context?.contentResolver?.notifyChange(uri, null)
        return ContentUris.withAppendedId(CONTENT_URI, id)
    }

    override fun update(uri: Uri, values: ContentValues?, selection: String?, selectionArgs: Array<String>?): Int {
        val count = db.userDao().update(values, selection, selectionArgs)
        context?.contentResolver?.notifyChange(uri, null)
        return count
    }

    override fun delete(uri: Uri, selection: String?, selectionArgs: Array<String>?): Int {
        val count = db.userDao().delete(selection, selectionArgs)
        context?.contentResolver?.notifyChange(uri, null)
        return count
    }

    override fun getType(uri: Uri): String? {
        return when (uriMatcher.match(uri)) {
            1 -> "vnd.android.cursor.dir/vnd.com.example.users"
            2 -> "vnd.android.cursor.item/vnd.com.example.users"
            else -> null
        }
    }
}
```

### Manifest 등록

```xml
<provider
    android:name=".UserProvider"
    android:authorities="com.example.app.provider"
    android:exported="true"
    android:readPermission="com.example.app.READ"
    android:writePermission="com.example.app.WRITE" />
```

### ContentResolver로 접근 (다른 앱에서)

```kotlin
// 다른 앱에서 ContentProvider 데이터 읽기
val cursor = contentResolver.query(
    Uri.parse("content://com.example.app.provider/users"),
    null, null, null, null
)

cursor?.use {
    while (it.moveToNext()) {
        val id = it.getString(it.getColumnIndexOrThrow("id"))
        val name = it.getString(it.getColumnIndexOrThrow("name"))
    }
}
```

### FileProvider — 파일 공유

실무에서 ContentProvider를 직접 구현하는 경우는 드물다. 가장 흔한 사용 사례는 **FileProvider**로 카메라, 파일 공유 등에 쓰인다.

```xml
<!-- Manifest -->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

```xml
<!-- res/xml/file_paths.xml -->
<paths>
    <external-cache-path name="images" path="." />
</paths>
```

```kotlin
// 카메라 앱에 파일 URI 전달
val file = File(externalCacheDir, "photo.jpg")
val uri = FileProvider.getUriForFile(this, "${packageName}.fileprovider", file)

val intent = Intent(MediaStore.ACTION_IMAGE_CAPTURE).apply {
    putExtra(MediaStore.EXTRA_OUTPUT, uri)
}
startActivity(intent)
```

### 시스템 ContentProvider 접근 예시

```kotlin
// 연락처 읽기
val cursor = contentResolver.query(
    ContactsContract.Contacts.CONTENT_URI,
    arrayOf(ContactsContract.Contacts.DISPLAY_NAME),
    null, null, null
)

// 미디어 파일 읽기
val cursor = contentResolver.query(
    MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
    arrayOf(MediaStore.Images.Media._ID, MediaStore.Images.Media.DISPLAY_NAME),
    null, null, null
)
```

## 💻 4대 컴포넌트 비교

| | Activity | Service | BroadcastReceiver | ContentProvider |
|--|--|--|--|--|
| 역할 | UI 화면 | 백그라운드 작업 | 이벤트 수신 | 데이터 공유 |
| 생명주기 | O | O | X (onReceive만) | onCreate만 |
| UI | O | X | X | X |
| 실무 빈도 | 매우 높음 | 높음 | 중간 | 낮음 (FileProvider 제외) |

## 🔗 관련 개념

- [[03-Android-Framework/4대-컴포넌트-와-Intent/Activity]]
- [[03-Android-Framework/4대-컴포넌트-와-Intent/Service]]
- [[03-Android-Framework/4대-컴포넌트-와-Intent/BroadcastReceiver]]

## 📚 더 보기

- [Android Docs - ContentProvider](https://developer.android.com/guide/topics/providers/content-providers)

---

**핵심 요약:** 앱 간 데이터 공유 표준. 직접 구현은 드물고 FileProvider가 가장 흔한 사용 사례. 시스템 데이터(연락처, 미디어)도 ContentResolver로 접근.
