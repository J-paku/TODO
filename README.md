
# 📘 React Native + Android Studio 세팅 요약 (2025-04-17)

---

## 📦 1. Node.js 설치

- [Node.js 공식사이트](https://nodejs.org/ja/)  
- 권장 버전: **Node.js 18.x LTS**  
- 설치 후 확인:

```bash
node -v
npm -v
```

---

## 🧰 2. Android Studio 설치 및 설정

- Android Studio 설치 후, 다음 설정 진행:

### ✅ Android SDK 설정

- SDK Manager → **Android 12L (API 32)** 또는 **Android 12 (API 31)** 설치
- 필수 항목:
  - Android SDK Platform 31
  - Intel x86 Atom System Image
  - Android Emulator
  - Android SDK Command-line Tools (latest)

### ✅ 환경변수 설정

```bash
ANDROID_SDK_ROOT=C:\Users\朴裁弘\AppData\Local\Android\Sdk
```

> *일부 도구에서 해당 경로 필요*

---

## ⚒️ 3. JDK 설치

- JDK 17 설치 (JDK 20, 21은 호환성 문제 있음)

```bash
java -version
```

→ `openjdk 17` 확인

---

## 🚧 4. React Native CLI 세팅

### ⚠️ **구버전 CLI 설치 금지 (`react-native-cli`)**

- 아래처럼 설치하지 않도록 주의:

```bash
npm install -g react-native-cli ❌ 사용 금지
```

### ✅ 대신 아래 명령어 사용:

```bash
npx @react-native-community/cli init MyTestApp
```

---

## 💻 5. 프로젝트 생성

```bash
cd C:\workspace
npx @react-native-community/cli init MyTestApp
cd MyTestApp
```

---

## 🚀 6. Android 에뮬레이터 실행

- AVD Manager → 에뮬레이터 실행
- 기기 사양 예시: Pixel 5, Android 31, x86_64

---

## ▶️ 7. 앱 실행

```bash
npx react-native run-android
```

---

## ❌ 빌드 오류 (원인)

- `C:\Users\朴裁弘` 경로가 prefab, CMake, Hermes에서 깨짐
- Java subprocess → 사용자 경로 인코딩 오류 발생

### 🛑 에러 메시지 예시

```
'C:\Users\�p�ٍO\.gradle\...' 경로에서 C++ prefab 실패
```

---

## ✅ 대응

- ASCII 전용 사용자 계정 생성 필요
- 관리자에게 요청 완료 (2025-04-17 기준)

---

## 🧪 (추가) prefab 에러 우회를 위한 Java 임시 디렉토리 변경

### 🔥 문제 원인

React Native 빌드 시 CMake prefab 에러는 주로 Java가 임시 파일을 저장하려는 위치가 다음과 같기 때문입니다:

```
C:\Users\朴裁弘\AppData\Local\Temp
```

→ 이 경로는 Java 내부에서 자동으로 설정되며, 한자가 포함되어 있을 경우 prefab 실행 시 경로 인코딩 오류가 발생합니다.

---

### 🛠 해결 방법 (우회)

#### ✅ 방법 1: 세션 단위로 임시 디렉토리 설정 (CMD에서 실행)

```cmd
set JAVA_TOOL_OPTIONS=-Djava.io.tmpdir=C:\temp
```

- Java의 임시 디렉토리를 `C:\temp` 등 ASCII-only 경로로 강제 지정
- 반드시 `C:\temp` 폴더는 미리 생성되어 있어야 함

#### ✅ 방법 2: gradlew 실행 시 명시적 지정

```cmd
gradlew -Djava.io.tmpdir=C:\temp assembleDebug
```

#### ✅ 방법 3: gradle.properties에 추가

```properties
org.gradle.jvmargs=-Djava.io.tmpdir=C:\temp
```

---

### ⚠ 주의 사항

- 위 방법은 일부 prefab 관련 에러를 우회할 수 있으나, 완전한 해결은 아닙니다.
- Java가 내부적으로 여전히 사용자 경로를 참조할 경우 실패할 가능성은 존재합니다.

---

