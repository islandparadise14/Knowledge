---
tags: [os, linux, android, kernel, adb, shell]
---

# Linux 기초

## 💡 핵심 개념

**Linux**는 오픈소스 운영체제 커널로, 프로세스 관리, 메모리 관리, 파일 시스템, 네트워크, 디바이스 드라이버 등 OS의 핵심 기능을 제공한다. Android는 Linux 커널 위에 구축되었으며, Linux의 프로세스 모델, 보안 체계, 파일 시스템을 그대로 활용한다.

## 📌 왜 필요한가?

Android 개발에서 ADB shell, 로그캣, 파일 권한, 프로세스 디버깅 등은 모두 Linux 명령어와 개념에 기반한다. Linux를 이해하면 Android의 내부 동작을 더 깊이 파악할 수 있다.

## 🔍 자세히

### Android와 Linux의 관계

```
┌─────────────────────────────────┐
│         Android Apps            │
├─────────────────────────────────┤
│      Android Framework          │
│  (AMS, WMS, PMS, System UI)     │
├─────────────────────────────────┤
│    Android Runtime (ART)        │
│    Native Libraries (Bionic)    │
├─────────────────────────────────┤
│           HAL                   │
╞═════════════════════════════════╡
│       Linux Kernel              │  ← 여기가 Linux
│  Binder, Ashmem, LMK, Wakelocks│
│  파일시스템, 네트워크, 스케줄러    │
└─────────────────────────────────┘
```

Android가 Linux 커널에 추가한 주요 요소:
- **Binder**: Android 전용 IPC 드라이버
- **Ashmem**: Anonymous Shared Memory (공유 메모리)
- **LMK**: Low Memory Killer
- **Wakelocks**: 전원 관리
- **ION/DMA-BUF**: 그래픽 메모리 할당

### 필수 Linux 명령어 (ADB Shell)

#### 파일 시스템 탐색

```bash
# 디렉토리 이동과 목록 확인
adb shell ls -la /data/data/com.example.app/
adb shell ls -la /sdcard/

# 파일 내용 확인
adb shell cat /proc/cpuinfo
adb shell cat /proc/meminfo

# 파일 복사/이동/삭제
adb shell cp /sdcard/file.txt /sdcard/backup/
adb shell mv /sdcard/old.txt /sdcard/new.txt
adb shell rm /sdcard/temp.txt

# 파일 크기 확인
adb shell du -sh /data/data/com.example.app/

# 파일 찾기
adb shell find /data/data/com.example.app/ -name "*.db"
```

#### 프로세스 관리

```bash
# 실행 중인 프로세스 목록
adb shell ps -A | grep "com.example"

# 프로세스 상세 정보
adb shell cat /proc/<PID>/status
adb shell cat /proc/<PID>/maps     # 메모리 맵
adb shell cat /proc/<PID>/fd       # 열린 파일 디스크립터

# 프로세스 종료
adb shell kill -9 <PID>

# 실시간 리소스 모니터링
adb shell top -n 1 -s cpu
```

#### 메모리 정보

```bash
# 시스템 전체 메모리
adb shell cat /proc/meminfo

# 앱별 메모리 사용량
adb shell dumpsys meminfo com.example.app

# 메모리 요약
adb shell dumpsys meminfo --summary
```

#### 네트워크

```bash
# 네트워크 인터페이스 확인
adb shell ifconfig
adb shell ip addr

# 네트워크 연결 상태
adb shell netstat -tlnp

# DNS 확인
adb shell nslookup google.com

# 포트 포워딩 (개발 PC → 에뮬레이터)
adb forward tcp:8080 tcp:8080
```

### Linux 파일 권한 체계

```
ls -la 출력:
-rw-rw---- 1 u0_a123 u0_a123_cache 4096 Feb 01 12:00 config.json
│├─┤├─┤├─┤   │         │
│ │  │  │    UID       GID
│ │  │  └── Others 권한
│ │  └───── Group 권한
│ └──────── Owner 권한
└────────── 파일 타입
```

Android의 UID 체계:
```
UID 0         → root
UID 1000      → system
UID 10000+    → 앱 (u0_a0 = 10000, u0_a1 = 10001, ...)
                각 앱은 설치 시 고유 UID를 받음
```

```bash
# 앱의 UID 확인
adb shell dumpsys package com.example.app | grep userId

# 특정 UID로 실행 중인 프로세스 확인
adb shell ps -A | grep u0_a123
```

### /proc 가상 파일 시스템

`/proc`는 커널 정보를 파일처럼 노출하는 가상 파일 시스템이다:

| 경로 | 내용 |
|------|------|
| /proc/cpuinfo | CPU 정보 (코어 수, 아키텍처) |
| /proc/meminfo | 메모리 사용 현황 |
| /proc/version | 커널 버전 |
| /proc/[PID]/status | 프로세스 상태 |
| /proc/[PID]/maps | 메모리 매핑 정보 |
| /proc/[PID]/oom_score_adj | LMK 우선순위 |
| /proc/[PID]/cmdline | 실행 명령줄 |

```bash
# 커널 버전 확인
adb shell cat /proc/version
# Linux version 5.15.x-android13-...

# CPU 코어 수
adb shell cat /proc/cpuinfo | grep processor | wc -l

# 앱의 OOM 점수 확인 (낮을수록 중요)
adb shell cat /proc/<PID>/oom_score_adj
```

### 유용한 ADB 디버깅 명령어

```bash
# 시스템 서비스 목록
adb shell service list

# Activity 스택 확인
adb shell dumpsys activity activities | grep "Running"

# 앱 데이터 초기화
adb shell pm clear com.example.app

# 앱 강제 종료
adb shell am force-stop com.example.app

# Activity 시작
adb shell am start -n com.example.app/.MainActivity

# Intent 브로드캐스트
adb shell am broadcast -a com.example.ACTION_TEST

# 스크린샷
adb shell screencap /sdcard/screen.png
adb pull /sdcard/screen.png

# 로그 필터링
adb logcat -s "MyTag"
adb logcat *:E  # Error 이상만

# 버그 리포트
adb bugreport > bugreport.zip
```

### 시그널 (Signal)

프로세스에 비동기 알림을 보내는 메커니즘:

| 시그널 | 번호 | 동작 | Android 대응 |
|--------|------|------|-------------|
| SIGKILL | 9 | 즉시 종료 (무시 불가) | LMK가 앱 종료 시 |
| SIGTERM | 15 | 종료 요청 | 정상 종료 요청 |
| SIGSEGV | 11 | Segmentation Fault | 네이티브 크래시 |
| SIGSTOP | 19 | 일시 정지 | 디버거 attach |
| SIGCONT | 18 | 재개 | 디버거 resume |

```bash
# 프로세스에 시그널 보내기
adb shell kill -SIGTERM <PID>  # 종료 요청
adb shell kill -9 <PID>       # 강제 종료
```

### 파이프와 리다이렉션

```bash
# 파이프 (|) - 한 명령의 출력을 다른 명령의 입력으로
adb shell ps -A | grep "com.example"
adb shell logcat -d | grep "ERROR"

# 리다이렉션
adb shell logcat -d > logcat.txt          # 출력을 파일로
adb shell logcat -d 2>&1 > all_logs.txt   # stderr 포함

# 여러 명령 연결
adb shell "ps -A | grep camera | awk '{print \$2}'"
```

### 환경 변수와 셸

```bash
# 환경 변수 확인
adb shell env
adb shell echo $PATH

# Android 주요 환경 변수
# ANDROID_DATA=/data
# ANDROID_ROOT=/system
# EXTERNAL_STORAGE=/sdcard

# 셸 종류 확인
adb shell echo $SHELL
# /system/bin/sh (mksh 또는 toybox sh)
```

### Kotlin에서 Linux 정보 접근

```kotlin
// CPU 코어 수
val cores = Runtime.getRuntime().availableProcessors()

// 현재 프로세스의 PID
val pid = android.os.Process.myPid()

// 현재 스레드의 TID
val tid = android.os.Process.myTid()

// 현재 앱의 UID
val uid = android.os.Process.myUid()

// 시스템 프로퍼티 읽기
fun getSystemProperty(key: String): String? {
    return try {
        val process = Runtime.getRuntime().exec("getprop $key")
        process.inputStream.bufferedReader().readLine()
    } catch (e: Exception) {
        null
    }
}

// /proc에서 메모리 정보 읽기
fun getMemoryInfo(): ActivityManager.MemoryInfo {
    val activityManager = getSystemService(ACTIVITY_SERVICE) as ActivityManager
    val memInfo = ActivityManager.MemoryInfo()
    activityManager.getMemoryInfo(memInfo)
    // memInfo.availMem - 사용 가능한 메모리
    // memInfo.totalMem - 전체 메모리
    // memInfo.lowMemory - 저메모리 상태 여부
    return memInfo
}
```

## 🔗 관련 개념

- [[00-CS-기초/운영체제/프로세스-관리|프로세스 관리]]
- [[00-CS-기초/운영체제/파일-시스템|파일 시스템]]
- [[00-CS-기초/운영체제/커널-모드-vs-유저-모드|커널 모드 vs 유저 모드]]
- [[00-CS-기초/운영체제/IPC-프로세스간-통신|IPC 프로세스간 통신]]
- [[00-CS-기초/운영체제/스케줄링|스케줄링]]

## 📚 더 보기

- [Android Source - Kernel](https://source.android.com/docs/core/architecture/kernel)
- [ADB Command Reference](https://developer.android.com/tools/adb)
- [The Linux Command Line](https://linuxcommand.org/tlcl.php)

---

**핵심 요약:** Android는 Linux 커널 기반. ADB shell로 Linux 명령어 사용 가능. /proc로 시스템 정보 접근. 각 앱은 고유 UID로 격리.
