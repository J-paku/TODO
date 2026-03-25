# Garoon × Pleasanter Server Script 연동 — 레슨 자료

## 레슨 1: 네트워크 구성

### 왜 Pleasanter에서 Garoon으로 직접 못 가?

```
[인터넷]
    │
    ├── Pleasanter 서버 (210.251.113.240)
    │   └── 외부에서 접근 가능 (SSL 증명서 있음)
    │       → 우리가 집에서도 볼 수 있는 이유
    │
    └── Garoon 서버
        └── 사내 네트워크에서만 접근 가능
            → 회사 WiFi 안에서만 볼 수 있는 이유
```

둘 다 같은 회사 서버실(랙)에 있지만, **네트워크가 분리**되어 있어서 서로 직접 통신이 안 됨.

정확한 분리 방식은 (VLAN / 방화벽 / 서브넷 등) 인프라 담당자만 아는 부분이라, 자료에는 **「ネットワーク分離」**라고만 쓰면 됨.

### Akamai Access의 역할

```
Pleasanter 서버 ──직접──▶ Garoon 서버    ❌ 불가

Pleasanter 서버 ──▶ Akamai Access ──▶ Garoon 서버    ✅ 가능
```

Akamai Access = **외부 경유 우회 터널**

- 회사가 Akamai라는 서비스에 돈 내고 쓰는 것
- Akamai가 "이 IP에서 오는 요청은 허가된 거다"라고 판단하면 Garoon까지 통과시켜줌
- 그래서 千本さん이 **Pleasanter IP(210.251.113.240)를 Akamai에 허가 등록**해줘야 하는 것

---

## 레슨 2: Pleasanter Server Script

### Server Script란

Pleasanter 레코드에 붙이는 **서버에서 실행되는 JavaScript 코드**.

```
브라우저에서 실행되는 JS    vs    Server Script
─────────────────────────    ─────────────────
사용자 PC에서 실행              Pleasanter 서버에서 실행
소스코드 노출됨                 소스코드 노출 안 됨
인증정보 넣으면 위험             인증정보 넣어도 안전 ✅
```

### 트리거 (발동 조건)

Server Script는 **특정 이벤트에 반응**해서 실행됨:

| 트리거 | 언제 발동 |
|--------|----------|
| レコード作成時 | 레코드 새로 만들 때 |
| レコード更新時 | 레코드 수정할 때 |
| **レコード読込時** | 레코드 읽을 때 ← **우리가 쓸 거** |
| レコード削除時 | 레코드 삭제할 때 |

우리는 **「レコード読込時」** 트리거를 사용. 브라우저가 Pleasanter API로 레코드를 조회하면 → 스크립트 발동 → httpClient로 Garoon 호출 → 결과 반환.

### httpClient란

Server Script 안에서 쓸 수 있는 **HTTP 요청 도구**.

```javascript
// Pleasanter 서버가 다른 서버에 HTTP 요청을 보내는 것
httpClient.RequestUri = 'https://어딘가의URL/api/...'
httpClient.RequestHeaders.Add('헤더이름', '헤더값')
const response = httpClient.Get()  // GET 요청 → 문자열로 응답 받음
```

핵심: **서버에서 실행**되니까 인증 헤더가 브라우저에 절대 안 보임.

### context 객체

Server Script에서 쓸 수 있는 **실행 환경 정보**:

| 속성 | 용도 | 예시 |
|------|------|------|
| `context.Condition` | 어떤 트리거로 발동됐는지 | `'WhenloadingRecord'` |
| `context.QueryStrings('key')` | URL 파라미터 읽기 | `?type=schedule` → `'schedule'` |
| `context.ResponseContents` | 응답 내용을 직접 지정 | JSON 문자열 넣으면 그대로 반환 |

```javascript
// 확인 방법 (디버깅용)
logs.LogInfo(context.Condition)
// → SysLog에 'WhenloadingRecord' 출력됨
```

---

## 레슨 3: 다중 실행 문제 (가장 중요)

### 문제 상황

Pleasanter API 호출 방법이 두 가지:

```
① SiteId + RecordId 지정  →  해당 레코드 1건만 → 스크립트 1번 실행 ✅
   GET /api/items/12345/678/get

② SiteId만 지정           →  사이트 전체 레코드 → 스크립트 N번 실행 ❌
   GET /api/items/12345/get
```

②번으로 호출하면 사이트에 레코드가 200개 있으면 **스크립트가 200번 발동**.
그 안에 httpClient가 있으면 → Garoon API도 200번 호출 → **서버 과부하**.

### 해결: 3단 방어

**1단: 구조적 방어** — 레코드를 1개만 만든다

```
[Garoon 프록시 전용 사이트]
└── RecordId: 1  ← 이것 하나만 존재

프론트엔드 코드:
GET /api/items/{SiteId}/1/get?type=schedule
                        ↑
                   항상 RecordId 지정
```

레코드가 1개뿐이니까, 실수로 SiteId만 호출해도 1번만 실행됨.

**2단: 코드 방어** — 스크립트 첫 줄에서 걸러낸다

```javascript
// "레코드 읽기"가 아닌 다른 이벤트로 발동하면 → 즉시 종료
if (context.Condition !== 'WhenloadingRecord') return

// URL에 type 파라미터가 없으면 → 일반 열람이므로 종료
const requestType = context.QueryStrings('type')
if (!requestType) return
```

**3단: 런타임 방어** — 짧은 시간 내 중복 실행 차단

```javascript
const lastRun = model.ClassA  // 레코드에 저장된 마지막 실행 시각
const now = new Date()

if (lastRun && (now - new Date(lastRun)) < 5000) {
    // 5초 이내 재실행 → Garoon 호출 안 하고 캐시 반환
    context.ResponseContents = JSON.stringify({ cached: true })
    return
}
```

---

## 레슨 4: Script.json (타임아웃)

Pleasanter 서버 설정 파일. **서버 관리자가 관리**하는 것.

```
Pleasanter 설치 폴더/App_Data/Parameters/Script.json
```

| 설정 | 의미 | 누가 설정 |
|------|------|----------|
| `ServerScriptTimeOut` | 스크립트 전체 실행 제한 (ms) | 서버 관리자 |
| `ServerScriptHttpClientTimeOut` | httpClient 기본 타임아웃 (ms) | 서버 관리자 |
| `ServerScriptHttpClientTimeOutMin` | httpClient 타임아웃 최솟값 | 서버 관리자 |
| `ServerScriptHttpClientTimeOutMax` | httpClient 타임아웃 최댓값 | 서버 관리자 |

```
스크립트 시작 ─────────────────────────────── 스크립트 종료
│            ServerScriptTimeOut               │
│                                              │
│   httpClient.Get() ──── 응답 대기 ──── 완료  │
│   │  ServerScriptHttpClientTimeOut  │        │
```

두 개가 별도라서:
- httpClient가 타임아웃 → httpClient만 실패, 스크립트는 catch로 에러 처리 가능
- 스크립트 전체가 타임아웃 → 전부 실패

→ 서버 관리자에게 **현재 값 확인** 필요.

---

## 레슨 5: Garoon 인증 방식

```
X-Cybozu-Authorization: Base64(유저명:비밀번호)
```

예시:
```
유저: ********
비번: ************

Base64("********:************") = "XXXXXXXXXXXXXXXXXXXXXXXX"
```

이 값을 httpClient 헤더에 넣어서 보내면 Garoon이 인증 처리함.

**Stateless 방식** — 매 요청마다 헤더를 보내야 함. 세션 같은 건 없음.

현행(iron-session)과의 차이:

| | 현행 (Next.js) | 이행 후 (Pleasanter) |
|---|---|---|
| 인증 저장 | iron-session (AES-256 암호화) | Pleasanter 레코드 (Base64) |
| 실행 위치 | Next.js 서버 | Pleasanter 서버 |
| 브라우저 노출 | ❌ 없음 | ❌ 없음 (동일) |
| Node.js 필요 | ✅ 필요 | ❌ 불필요 |

---

## 회의에서 나올 수 있는 질문 & 답변

| 예상 질문 | 답변 포인트 |
|----------|------------|
| 보안은 괜찮아? | httpClient는 서버사이드 실행, 인증정보 브라우저 노출 없음 (Pleasanter 공식 확인) |
| 다중 실행 어떻게 막아? | 3단 방어: 구조(1레코드) + 코드(가드) + 런타임(시간 락) |
| Akamai 안 되면? | 현행 Next.js 방식 유지하면 됨. 롤백 리스크 없음 |
| 성능은? | 44회 → 1~3회로 줄어듦. 검증에서 실측 예정 |
| 타임아웃 걸리면? | Script.json 설정 조정 가능, try-catch로 에러 처리 |
| 언제 완료돼? | Phase 1(검증) → Phase 2(구현) → Phase 3(이행). Akamai 허가 후 순차 진행 |
