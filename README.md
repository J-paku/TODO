# 목표
현재 프로젝트의 AI 하네스 환경을 분석하여, “실전용 고신뢰 하네스”로 끌어올리기 위한 부족한 계층을 설계하고 구현하라.

# 현재 상태 요약
이 프로젝트는 이미 다음을 갖추고 있다:
- 명세 계층: 글로벌/프로젝트 CLAUDE.md, AGENTS.md, MEMORY.md, docs, skills
- PostToolUse Guardrails: 코드 스타일, import alias, 신규 파일 헤더, magic strings, component structure, docs sync
- 실행 계층: /gg, /review, team-protocol
- 멀티 워커 구조: Opus(감독), Codex(로직/타입), Sonnet(UI/패턴), Haiku(경량작업), Sonnet 검증
- 최종 품질 게이트: tsc, build
- docs sync 의무, worktree 기반 병렬 실행, fallback/escalation 체인

하지만 아래가 부족하다:
1. PreToolUse 정책 엔진
2. 자동 테스트 계층
3. CI 파이프라인
4. Eval 체계
5. Observability / trace / run log
6. 역할별 권한 분리 강화
7. task-to-doc routing 최적화

# 절대 목표
“AI가 잘못 쓴 뒤 잡는 구조”에서 끝내지 말고,
“애초에 잘못된 실행을 막고, 실행 결과를 측정하고, 반복적으로 개선 가능한 하네스”를 만든다.

# 해야 할 일
다음 순서로 진행하라.

## Phase 1. 현황 분석
아래 파일과 구조를 먼저 확인하라.
- ~/.claude/CLAUDE.md
- ~/.claude/commands/gg.md
- ~/.claude/commands/review.md
- ~/.claude/commands/team-protocol.md
- ~/.claude/projects/{project}/memory/*
- {project}/.claude/settings.local.json
- {project}/.claude/hooks/*
- {project}/PiPiT-web/CLAUDE.md
- {project}/PiPiT-web/AGENTS.md
- {project}/PiPiT-web/README.md
- {project}/PiPiT-web/docs/*
- {project}/.agents/skills/*

분석 결과를 다음 표로 정리하라.
- existing layer
- current implementation
- weakness
- risk
- proposed improvement
- priority (P0/P1/P2)

## Phase 2. 설계안 제시
다음 7개 산출물을 제안하라.

1. PreToolUse policy design
2. Test automation design
3. CI workflow design
4. Eval benchmark design
5. Observability schema
6. Agent capability / permission matrix
7. Task routing / doc routing design

각 항목은 아래 형식으로 정리하라.
- why
- current gap
- target design
- files to create/update
- rollout order
- risk

## Phase 3. 우선 구현 범위 확정
P0만 먼저 구현하라. P0는 아래로 제한한다.
- PreToolUse guard 초안
- run log / trace JSONL 설계 및 최소 구현
- Vitest 기반 최소 테스트 러너 도입
- GitHub Actions 최소 파이프라인
- agent capability matrix 문서화
- task-to-doc routing 강화

P1/P2는 구현하지 말고 설계와 TODO만 남겨라.

## Phase 4. 실제 수정
아래 원칙을 지켜라.
- 최소 수정 우선
- 기존 CLAUDE.md / AGENTS.md / MEMORY.md 체계 존중
- 기존 PostToolUse 훅과 충돌하지 말 것
- 기존 /gg, /review 흐름을 깨지 말 것
- docs sync 의무를 준수할 것
- 구조 변경이 있으면 관련 문서도 함께 갱신할 것
- docs-only 변경은 분리 가능하게 설계할 것

## Phase 5. 구현 대상
필수 구현 예시:
- `.claude/hooks/preflight-guard.sh` 또는 동등한 PreToolUse 정책 스크립트
- `.claude/lib/run-trace.*`
- `.claude/evals/README.md`
- `.github/workflows/ai-harness-ci.yml`
- `docs/ai-harness-observability.md`
- `docs/ai-harness-eval-plan.md`
- `AGENTS.md` 내 task → required docs mapping 강화
- 필요 시 `settings.local.json` 갱신

실제 경로는 현행 구조를 보고 가장 맞는 위치로 조정해도 된다.

## Phase 6. 검증
반드시 아래를 실행하라.
- git status
- typecheck
- test
- build
- 훅 충돌 여부 확인

실행 결과를 요약하라.
- changed files
- validation result
- remaining risks
- deferred items

# 중요한 제약
- any 금지
- unknown 금지 (예외 최소화)
- 세미콜론 금지
- 더블 쿼트 금지 (예외 필요 시 근거 명시)
- 상대경로 ../../ 이상 금지
- 신규 파일 첫 줄 설명 코멘트 필수
- 일본어 코멘트 규칙 유지
- 기존 프로젝트 패턴 존중
- 임의로 대규모 리팩터링 금지

# 산출 형식
최종 응답은 아래 구조로 작성하라.

## 1. Gap Analysis
## 2. P0 Implementation Plan
## 3. Files to Create/Update
## 4. Code Changes
## 5. Validation Results
## 6. Risks / Deferred Items

# 판단 기준
좋은 결과물은 다음을 만족해야 한다.
- 기존 하네스를 부수지 않는다
- AI 실수를 “사후 교정”뿐 아니라 “사전 차단”한다
- 워커 실행을 추적 가능하게 만든다
- 품질을 측정 가능한 형태로 바꾼다
- 향후 Eval/Observability 확장이 쉬운 구조를 남긴다

기존 구조를 칭찬만 하지 말고, 실제 운영 중 깨질 지점을 냉정하게 찾아서 “왜 지금 이 상태로는 부족한지”를 먼저 증명한 뒤 구현하라.
