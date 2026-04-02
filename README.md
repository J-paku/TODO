# PiPiT-web AI 하네스 환경 해설서

> 이 문서는 AI 에이전트가 이 프로젝트에서 어떻게 제어·검증·협업되는지를 정리한 것이다.
> `/gg`(코드 수정), `/review`(코드 리뷰) 뿐 아니라 **일반 코딩 세션 전체**에 적용되는 하네스 구조를 설명한다.

---

## 1. 하네스 엔지니어링이란

```
Prompt Engineering → Context Engineering → Harness Engineering
     (어떻게 말할까)     (무엇을 보여줄까)       (무엇을 방지/측정/수정할까)
```

하네스 엔지니어링은 "AI가 실수하지 않도록 시스템 차원에서 울타리를 치는 것"이다.
프롬프트 하나를 잘 쓰는 것이 아니라, **파일 구조·훅·명세·검증 루프** 전체를 설계하여 AI가 일관되게 좋은 코드를 생산하도록 만든다.

---

## 2. 전체 구조 한눈에 보기

```
┌──────────────────────────────────────────────────────────────────┐
│                         사용자 (朴)                               │
│  "로그인 버튼 추가해줘 ㄱㄱ"  /  "/gg ..."  /  "/review"         │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│  ① 명세 계층 (Context = AI가 읽는 지식)                          │
│                                                                  │
│  CLAUDE.md (글로벌)  ← 프로젝트 전체 룰, 코드 스타일, 언어 규칙   │
│  CLAUDE.md (프로젝트) ← Phase 1~4 실행 프로토콜                   │
│  AGENTS.md           ← Task→Guide 매핑, Key Paths, Doc Sync 의무 │
│  MEMORY.md           ← 세션 간 지속 학습 (절대 규칙, Codex 설정)   │
│  docs/*.md (13개+)   ← 도메인별 상세 가이드                       │
│  .agents/skills/     ← Next.js 베스트 프랙티스 13개 가이드         │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│  ② 안전장치 계층 (Guardrails = AI가 쓸 때마다 자동 검사)          │
│                                                                  │
│  PostToolUse Hooks (Edit/Write 실행 후 자동 트리거):              │
│  ┌────────────────────────────────────────────────────────┐      │
│  │ validate-code-style.sh    → any/unknown/;/" /barrel 금지│      │
│  │ validate-import-alias.sh  → ../../ 이상 상대경로 금지    │      │
│  │ validate-new-file.sh      → 신규파일 헤더 주석 필수      │      │
│  │ validate-magic-strings.sh → 하드코드 매직스트링 금지     │      │
│  │ check-component-structure.sh → 공유 컴포넌트 구조 안내   │      │
│  │ sync-docs.sh              → 문서 변경 시 동기화 알림     │      │
│  └────────────────────────────────────────────────────────┘      │
│  ※ 위반 시 exit 2 → 도구 실행 거부, AI가 자동으로 수정 후 재시도  │
│                                                                  │
│  settings.local.json  → Bash 명령어 화이트리스트 (88개+)          │
│  MCP 서버             → context7 (라이브러리 문서 자동 조회)       │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│  ③ 실행 계층 (팀 프로토콜 = /gg, /review 시 발동)                │
│                                                                  │
│  team-protocol.md → 공통 인프라 정의                              │
│  gg.md            → 코드 수정 전용 워크플로우                     │
│  review.md        → 코드 리뷰 전용 워크플로우                     │
│                                                                  │
│  ┌─────────────────────────────────────────────────────┐         │
│  │  Opus (팀장 — 감독 전용, 코드 작성 금지)             │         │
│  │  역할: 분석 → 플랜 → 배분 → 최종 판정               │         │
│  ├─────────────────────────────────────────────────────┤         │
│  │  워커 (1:1 동등, 병렬 실행)                          │         │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │         │
│  │  │  Codex   │  │  Sonnet  │  │  Haiku   │          │         │
│  │  │ 로직/타입 │  │ UI/패턴  │  │ 경량작업  │          │         │
│  │  └──────────┘  └──────────┘  └──────────┘          │         │
│  ├─────────────────────────────────────────────────────┤         │
│  │  검증: Sonnet (별도 인스턴스)                        │         │
│  │  폴백: 워커 실패 → 상대 워커 → Opus 최후 수단        │         │
│  └─────────────────────────────────────────────────────┘         │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│  ④ 검증 계층 (최종 품질 게이트)                                   │
│                                                                  │
│  - Sonnet 검증 (code_modification / code_review 모드)             │
│  - npx tsc --noEmit (타입 체크)                                   │
│  - npm run build (빌드 확인)                                      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. 일반 코딩 세션 (팀 프로토콜 없이)

`/gg`이나 `/review`를 사용하지 않는 **평범한 대화**에서도 다음이 자동으로 적용된다:

### 3.1 Phase 프로토콜 (CLAUDE.md 프로젝트)

```
Phase 1: 분석 → AGENTS.md 읽기, git status 확인, 영향 범위 파악
Phase 2: 플랜 제출 → 사용자 승인 대기
Phase 3: 실행 → 승인된 파일만 수정
Phase 4: 문서 동기화 → 구조 변경 시 관련 문서 업데이트
```

- Phase 2 생략 조건: 사용자가 "바로 실행", "플랜 필요없어" 라고 말한 경우
- Codex 에이전트는 Phase 2 생략 가능 (내부에서 자체 계획)

### 3.2 PostToolUse 훅 (항상 활성)

AI가 `Edit` 또는 `Write` 도구를 사용할 때마다 **자동으로 실행**되는 셸 스크립트:

| 훅 | 검사 대상 | 위반 시 |
|---|----------|--------|
| `validate-code-style.sh` | `: any`, `: unknown`, `;`, `"`, `from '.'` | **exit 2** (쓰기 거부 + 자동 수정 유도) |
| `validate-import-alias.sh` | `../../` 이상 상대 import | **exit 2** (쓰기 거부) |
| `validate-new-file.sh` | 신규 .ts/.tsx 파일 헤더 주석 누락 | **exit 2** (쓰기 거부) |
| `validate-magic-strings.sh` | refactor-backlog.md에 정의된 매직 스트링 | **exit 2** (쓰기 거부) |
| `check-component-structure.sh` | components/ 하위 파일 구조 | 정보 제공만 (exit 0) |
| `sync-docs.sh` | .md 파일 변경 | 동기화 알림 (exit 0) |

핵심: **AI가 잘못된 코드를 쓰면 도구 자체가 거부된다.** AI는 수정 후 다시 시도할 수밖에 없다.

### 3.3 MEMORY.md (세션 간 지속)

AI가 세션을 넘어 기억하는 규칙들:

- 응답 언어: 한국어
- 일본어/영어 혼합 시 반각 스페이스 금지
- `Co-Authored-By` 절대 붙이지 말 것
- Codex 모델은 `gpt-5.3-codex`만 사용
- 타임아웃 ≠ 실패 (파일 변경 여부로 판단)

---

## 4. /gg 워크플로우 (코드 수정 팀)

```
사용자 의뢰 → Step 1: 분석 → Step 2: 플랜 → Step 3: 워커 병렬 실행
                                                   │
                                    ┌──────────────┼──────────────┐
                                    ▼              ▼              ▼
                                  Codex          Sonnet          Haiku
                                (worktree)     (worktree)     (worktree)
                                    │              │              │
                                    └──────────────┼──────────────┘
                                                   ▼
                               Step 4: Sonnet 검증 (별도 인스턴스)
                                                   │
                                    ┌──── ACCEPT ──┤── MODIFY/REJECT
                                    ▼              ▼
                               Step 5: 통합    에스컬레이션
                                    │           (재작업/폴백)
                                    ▼
                         Step 6: 리포트 + tsc 타입 체크
```

### 워커 배분 기준

| 워커 | 강점 | 실행 방법 |
|------|------|----------|
| Codex (gpt-5.3-codex) | 로직, 타입, 알고리즘, 리팩터링 | `codex exec --model gpt-5.3-codex --full-auto "{지시}"` (Bash, background, 180초) |
| Sonnet | 컴포넌트, UI, 프로젝트 패턴 활용 | Task tool (model: sonnet, isolation: worktree) |
| Haiku | 단순 UI, Tailwind, import 정리 | Task tool (model: haiku, isolation: worktree) |

### 팀 규모

| 규모 | Codex | Sonnet | Haiku | 검증 |
|------|-------|--------|-------|------|
| 소규모 (1~2 파일) | x1 | x1 | — | 1회 |
| 중규모 (3~5 파일) | x1 | x1 | x0~1 | 1회 |
| 대규모 (6+ 파일) | x1~2 | x1~2 | x0~1 | 1회 |

### 에스컬레이션 체인

```
워커 타임아웃 → git status로 파일 확인
  ├── 파일 변경 있음 → 성공 처리 → Sonnet 검증
  └── 파일 변경 없음 → 상대 워커에게 재배분
                          └── 양쪽 실패 → Opus 최후 수단 (코드 직접 작성)
```

---

## 5. /review 워크플로우 (코드 리뷰 팀)

```
git diff → Step 1: 변경 파일 수집
              │
              ▼
     Step 2: 리뷰어 3팀 병렬 실행
     ┌────────────┬────────────┬────────────┐
     │  Codex     │  Sonnet    │  Haiku     │
     │ 버그/성능   │ 아키텍처    │ 린트       │
     └─────┬──────┴─────┬──────┴─────┬──────┘
           │            │            │
           ▼            ▼            ▼
     Step 3: 검증
     ├── Codex+Sonnet finding → Sonnet 검증
     ├── Haiku finding → 검증 없이 CONFIRMED
     └── 전원 "No issues found" → 검증 스킵
              │
              ▼
     Step 4: Opus 최종 판정
     Step 5: 리포트 (APPROVE / REQUEST_CHANGES / COMMENT)
     Step 6: REQUEST_CHANGES → 워커에게 수정 위임
```

### 리뷰 관점

| 리뷰어 | 관점 |
|--------|------|
| Codex | 버그, 엣지케이스, null 안전성, 성능, React Hooks 규칙, API 패턴, 타입 안전성 |
| Sonnet | 아키텍처 일관성, 기존 패턴과의 일관성, 의존성 방향, 재사용성 |
| Haiku | `: any`, `: unknown`, `;`, `"`, `../../` import, `from '.'`, 비일본어 코멘트 |

---

## 6. 토큰 절감 전략

AI 에이전트의 토큰 사용을 최소화하기 위한 규칙:

1. **Opus는 최소한의 파일만 읽는다** — 워커가 직접 Read tool로 읽음
2. **워커 프롬프트에 파일 내용을 넣지 않는다** — 경로만 전달
3. **프로젝트 룰은 5줄 이내 요약** — 해당 작업에 필요한 것만
4. **참고 코드는 파일 경로만 전달**
5. **관련 파일 2~3개를 1워커에 묶어** 에이전트 수 최소화
6. **스타일 검증은 PostToolUse 훅에 위임** — 별도 리뷰어 불필요
7. **클린 리뷰 시 검증 스킵** — 양쪽 모두 "No issues found"면 Sonnet 검증 생략

---

## 7. 프로젝트 코드 규칙 요약

코드를 쓸 때 AI가 반드시 지켜야 하는 규칙:

| 규칙 | 내용 | 강제 수단 |
|------|------|----------|
| `any` 금지 | 구체적 타입 정의 필수 | 훅(exit 2) |
| `unknown` 금지 | catch/서드파티 제외 시 사용 금지 | 훅(exit 2) |
| 세미콜론 금지 | ASI 의존 (for문 제외) | 훅(exit 2) |
| 더블 쿼트 금지 | 싱글 쿼트 사용 (JSON/라이브러리 제외) | 훅(exit 2) |
| 배럴 import 금지 | `from '.'` 대신 `from './index'` | 훅(exit 2) |
| 깊은 상대경로 금지 | `../../` 이상 → `@/` alias | 훅(exit 2) |
| 신규 파일 헤더 필수 | 첫 줄에 `//` 설명 코멘트 | 훅(exit 2) |
| 매직 스트링 금지 | 정의된 상수 사용 | 훅(exit 2) |
| 코멘트 일본어 필수 | 다른 언어 금지 | CLAUDE.md + 리뷰 |
| Hooks `use` prefix | `useSomething` 형식 | CLAUDE.md |
| Props 인터페이스 상단 정의 | 파일 상부에 위치 | CLAUDE.md |

---

## 8. 문서 동기화 의무

구조적 변경(hook/컴포넌트/lib/타입 추가·삭제·이름변경) 시 반드시 관련 문서를 업데이트하고 **별도 docs-only 커밋**으로 분리한다.

| 변경 유형 | 업데이트 대상 |
|----------|-------------|
| Hook 추가/삭제/이름변경 | README.md, AGENTS.md |
| Component 추가/삭제/이동 | README.md, AGENTS.md |
| lib/ 유틸 추가/삭제 | README.md, AGENTS.md |
| z-index 변경 | docs/z-index-layer-guide.md |
| 접근성 변경 | docs/accessibility-a11y-v1.md |
| Claim SWR 변경 | docs/claim-swr-architecture.md |
| 매직 스트링 해소 | docs/refactor-backlog.md |

---

## 9. 파일 맵

```
~/.claude/
├── CLAUDE.md                    ← 글로벌 프로젝트 룰 (모든 세션에 로드)
├── commands/
│   ├── gg.md                    ← /gg 슬래시 커맨드 정의
│   ├── review.md                ← /review 슬래시 커맨드 정의
│   └── team-protocol.md         ← 팀 프로토콜 공통 인프라
└── projects/{project}/
    └── memory/
        ├── MEMORY.md            ← 세션 간 지속 메모리 (항상 로드)
        ├── harness-engineering.md
        ├── project-harness-status.md
        └── codex-review-config.md

{project}/
├── .claude/
│   ├── settings.local.json      ← Bash 화이트리스트, MCP 설정
│   └── hooks/                   ← PostToolUse 자동 검증 훅 6개
│       ├── validate-code-style.sh
│       ├── validate-import-alias.sh
│       ├── validate-new-file.sh
│       ├── validate-magic-strings.sh
│       ├── check-component-structure.sh
│       └── sync-docs.sh
├── PiPiT-web/
│   ├── CLAUDE.md                ← 프로젝트별 Phase 프로토콜
│   ├── AGENTS.md                ← Task→Guide 매핑 + Doc Sync 의무
│   ├── README.md                ← 프로젝트 구조 + 기술 스택
│   └── docs/*.md                ← 도메인별 가이드 13개+
└── .agents/skills/              ← Next.js 베스트 프랙티스
```

---

## 10. 성숙도 현황

```
안전장치 (Guardrails)       🟢 4/5  — 훅 6개, 화이트리스트 88개+
명세/작업 분해 (Plan&Spec)  🟢 4/5  — CLAUDE.md 3개, AGENTS.md, docs 13개+, skills 13개
검증 루프 (Testing/CI)      🟡 2/5  — ESLint+Prettier+tsc 있음, 테스트/CI 없음
품질 평가 (Eval)            ❌ 0/5  — 미설정
관측 가능성 (Observability) ❌ 0/5  — 미설정
```

### 강점
- Claude Code 에이전트 통합 설계가 매우 정교함 (훅 기반 자동 검증)
- 문서화가 상세하고 에이전트 친화적
- 다중 AI 모델 협업 구조 (Opus+Codex+Sonnet+Haiku)

### 개선 여지
- 자동 테스트 도입 (Vitest 추천)
- CI/CD 파이프라인 (GitHub Actions)
- 에러 추적 (Sentry)
- 코드 커버리지 측정
