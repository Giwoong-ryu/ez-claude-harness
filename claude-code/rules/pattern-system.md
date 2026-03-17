# 사용자 패턴 축적-분석-업데이트 시스템

> Hook `pattern_trigger.py`가 긍정 표현 감지 시 이 파일을 참조.
> Opus가 세션 내에서 직접 분석한다 (스킬/스크립트/저렴한 모델 위임 금지).

---

## 목적별 패턴 분리 (핵심 원칙)

**같은 사용자라도 목적이 다르면 패턴이 다르다.** 하나의 파일에 전부 넣지 않는다.

```
memory/patterns/
├── 3주차_강의_규칙파일스킬.json    <- 3주차: 규칙 파일 + 스킬 세팅
├── QT_SaaS_영상제작.json          <- QT Video SaaS 제품 개발
├── 강의제조기_시스템.json          <- 강의 생성 파이프라인
├── 중급자_바이브코딩.json          <- 바이브코딩 실습 (대상: 중급자)
└── _common.json                   <- 목적 무관 공통 (항상 로드)
```

**파일명 규칙:** AI가 Glob/검색으로 빠르게 찾을 수 있는 이름. 핵심 키워드를 파일명에 포함.

---

## 세션 시작 시 로드 절차

```
[1] Read: memory/patterns/INDEX.json -> 전체 패턴 파일 목록
[2] Task 식별: keywords + working_dir + active_files 교차 매칭
    - keywords: 사용자 첫 메시지/열린 파일에서 키워드 추출
    - working_dir: 현재 작업 디렉토리와 INDEX의 working_dir 비교
    - 2개 이상 일치하면 해당 파일 로드
[3] _common.json은 always_load=true -> 항상 로드
[4] _buffer.json 잔류 데이터 정리 (이전 세션 미처리분)
    - _buffer.json 존재 시 Read
    - 현재 세션과 같은 task의 items → 목적별 패턴 파일로 이동
    - task 불명/매칭 불가 items → _common.json의 corrections에 추가
    - 정리 후 _buffer.json 클리어 (또는 삭제)
[5] 매칭되는 파일 없으면 -> 새 파일 생성 (목적 키워드로 파일명)
[6] INDEX.json에 새 파일 등록
```

**파일 선택 규칙:**
- 같은 강의라도 주차/대상이 다르면 별도 파일
- 새 파일 생성 시 INDEX.json에 keywords + working_dir 등록

---

## 3단계 흐름

```
[Phase 1: 축적] 사용자가 수정/원칙/거부/결정 -> 즉시 _buffer.json에 저장
     |
[Phase 2: 승인 트리거] Hook이 긍정/결정 표현 감지 -> 알림 주입
     |
[Phase 3: 분석+업데이트] _buffer.json → 분석 → 목적별 패턴 파일로 이동 → _buffer.json 클리어
```

---

## Phase 1: 축적 (즉시 파일 저장)

사용자가 아래 행동을 할 때마다 **즉시** `memory/patterns/_buffer.json`에 append한다.
긍정 트리거나 PreCompact를 기다리지 않는다. 발생 즉시 저장.

### 축적 대상 4가지

| 유형 | 감지 기준 | 축적 내용 |
|------|----------|----------|
| **교정** | AI 출력물을 구체적으로 수정 요청 | ai_did(AI가 한 것) + user_said(사용자 요구) + correct_answer(정답) |
| **원칙** | 앞으로도 적용될 규칙/기준 제시 | principle(규칙 내용) + severity(critical/high/medium) |
| **거부** | AI 제안을 명시적으로 부정 | rejected(거부된 내용) + reason(이유) |
| **결정** | 여러 옵션 중 하나를 확정 | what(선택) + why(이유) + alternatives(탈락 대안) + reverse_when(되돌림 조건) |

### 판단 기준표 (저장 vs 비저장)

| 구분 | 저장 O (패턴) | 저장 X (비해당) |
|------|-------------|---------------|
| **교정** | "이거 빼고 저걸로 바꿔" (AI 출력 수정) | "좀 더 설명해줘" (추가 요청) |
| **원칙** | "항상 ~해", "~는 절대 하지마" (반복 적용) | "이번에만 ~해줘" (일회성) |
| **거부** | "이건 아닌데", "필요없어" (명시적 부정) | "다른 건 없어?" (탐색/비교) |
| **결정** | "이걸로 하자", "B로 가자" (확정) | "어떤게 나아?" (질문) |

### 핵심 구분 원칙

```
[1] "이번만" vs "앞으로도"
    → 일회성 지시 = 저장 안 함
    → 반복 적용될 규칙 = 원칙으로 저장

[2] "질문" vs "지시"
    → 질문/탐색 = 저장 안 함
    → 명확한 지시/확정 = 저장

[3] "추가 요청" vs "수정 요청"
    → "더 해줘" = 저장 안 함
    → "이거 틀렸어, 이렇게 바꿔" = 교정으로 저장

[4] 같은 지적 반복
    → 기존 항목의 frequency 카운트 +1
    → frequency 3+ 이면 severity 자동 상향
```

### _buffer.json 형식

```json
{
  "session_id": "현재 세션 ID",
  "task": "현재 작업 목적 (INDEX.json 매칭용)",
  "items": [
    {
      "type": "correction|principle|rejection|decision",
      "timestamp": "2026-03-14T15:30:00",
      "data": { ... }
    }
  ]
}
```

저장 빈도: 세션당 약 3~5건 (전체 메시지의 1~3%). 부담 없는 수준.

### 동시 세션 안전

- _buffer.json은 세션 단위가 아닌 **항목(item) 단위 append**
- 각 item에 session_id 포함 → Phase 3에서 현재 세션 items만 처리
- 파일 읽기→쓰기 사이 충돌 최소화: 기존 내용 읽고 items에 append 후 전체 쓰기

---

## Phase 2: 저장 트리거 = 긍정 표현

Hook `pattern_trigger.py`가 자동 감지. AI가 놓칠 수 없다.

| 긍정 신호 | 예시 |
|----------|------|
| **명시적 승인** | "좋다", "괜찮다", "됐어", "OK", "끝이야", "이거면 돼" |
| **부분 승인** | "좋은데", "괜찮은데 ~", "이건 맞아" |
| **완성도 언급** | "80%다", "거의 다 됐어", "이 부분만 고치면" |
| **다음 전환** | "그럼 이제 ~하자", "다음은" (이전 작업에 만족했다는 의미) |
| **컨텍스트 압축 임박** | 대화가 길어져 압축 직전 (Phase 1이 즉시 저장하므로 데이터 손실 없음) |

**부정("별로", "아닌데")은 트리거가 아니다** -> Phase 1 축적을 계속할 뿐.
긍정이 나와야 "여기까지가 사용자가 원하는 방향"으로 확정되어 저장 가치가 생긴다.

### Phase 3 실행 보장 (3중 안전망)

```
[1차] Hook 트리거 → AI가 즉시 Phase 3 실행 (정상 경로)
[2차] PreCompact hook → 압축 전 _buffer에 미처리 있으면 Phase 3 강제 안내
[3차] 다음 세션 시작 → 로드 절차 [4]에서 잔류 buffer 자동 정리
```

"좋아, 다음 ~해줘" 같이 긍정+즉시 요청이 붙은 경우:
→ Phase 3은 1줄 보고만 (분석은 가볍게, 사용자 요청 처리가 우선)

---

## Phase 3: 체크포인트 저장 + 차이점 분석

> Opus 직접 실행. 이유: 한국어 뉘앙스 판단, 맥락 이해, 추가 비용 0, 유지보수 최소.

긍정 트리거 감지 시 자동 실행:

```
[1] 체크포인트 생성
    - 현재 만족도 수준 기록 (80%, 90%, 100% 등)
    - 긍정 부분: 사용자가 만족한 방향/결과 저장
    - 부족 부분: 사용자가 언급한 남은 문제 간단히 기재

[2] 이전 체크포인트와 비교 (있는 경우)
    - 이전(80%) -> 현재(90%): 뭐가 달라졌는지 diff
    - 개선된 10%가 뭔지 분석 -> 이것이 사용자가 중시하는 패턴
    - 여전히 부족한 10%도 기록 -> 다음 작업의 우선순위

[3] 패턴 추출
    - 체크포인트 간 개선 항목 -> 사용자가 중요하게 보는 것
    - 반복 지적 항목 -> severity 상향
    - 새로운 원칙 -> principles에 추가

[4] 목적별 패턴 파일 업데이트
    - checkpoints[] 배열에 체크포인트 추가
    - corrections/principles/preferences 갱신
    - 기술/구조 선택이 있었으면 decisions[] 추가 (대안+되돌림 조건 포함)

[5] 관련 시스템 파일 업데이트
    - 슬라이드 -> slide-writing-rules.md
    - 코드 -> patterns.json
    - 강의 -> lecture_config.json
    - 작업 방식 -> CLAUDE.md rules/

[6] _buffer.json 클리어
    - 처리 완료된 items 제거 (현재 세션 것만)
    - 다른 세션의 미처리 items는 보존

[7] 1줄 보고: "[PATTERN] 체크포인트 저장 (80%->90%, +N개 패턴)"
```

### 축적된 피드백이 없을 때

Hook이 긍정을 감지해도 **축적된 교정/원칙/거부가 없으면** Phase 3를 실행하지 않는다.
단순 동의("좋아", "OK")만으로는 저장할 패턴이 없다.

---

## 저장 형식 (목적별 파일)

```json
{
  "_meta": { "purpose": "3주차 강의 준비", "created": "2026-03-14" },
  "checkpoints": [
    { "id": "CP001", "satisfaction": "80%", "positive": "...", "gaps": "...", "diff_from_prev": "..." }
  ],
  "corrections": [{ "ai_did": "...", "user_said": "...", "correct_answer": "...", "pattern": "..." }],
  "decisions": [{
    "id": "D001",
    "date": "2026-03-14",
    "what": "무엇을 결정했는지",
    "why": "왜 이걸 선택했는지",
    "alternatives": ["검토했지만 탈락한 대안들"],
    "reverse_when": "이 결정을 뒤집어야 하는 조건"
  }],
  "principles": [{ "principle": "...", "severity": "critical/high/medium" }],
  "patterns_discovered": ["정확성 > 편의성", "완결성 중시"]
}
```

---

## _common.json (목적 무관 공통)

```json
{
  "communication": ["확신 없으면 모른다고 하고 조사해라", "선택지 기반 제시"],
  "workflow": ["파일 수정 시 관련 파일 전체 동기화", "기존 도구 우선 활용"],
  "anti_patterns": ["추측으로 자신있게 말하기", "뻔한 팁 수준의 해결책"]
}
```

---

## 시스템 파일 업데이트 범위

| 패턴 유형 | 업데이트 대상 |
|----------|-------------|
| 슬라이드/강의 교정 | slide-writing-rules.md, lecture_config.json |
| 코드 품질 교정 | patterns.json, eazycheck-v5.md |
| 작업 방식 교정 | CLAUDE.md (해당 섹션) |
| 목적별 패턴 | memory/patterns/{purpose}.json |
| 공통 패턴 | memory/patterns/_common.json |

---

## 파일 관리 (월 1회)

- checkpoints 20개 초과 시 → 오래된 것 `_archive/` 폴더로 이동
- corrections에서 frequency 5+ 이고 이미 시스템 파일에 반영된 것 → 삭제 (중복)
- 목적 파일이 6개월 미사용 → `_archive/`로 이동

---

## memory/ vs patterns/ 역할 분리

| 저장소 | 용도 | 형식 | 예시 |
|--------|------|------|------|
| `memory/*.md` | AI auto-memory (프로젝트 정보, 참조, 사용자 프로필) | 마크다운 | feedback_ag_cli_issues.md |
| `memory/patterns/*.json` | 행동 패턴 (교정/원칙/거부/결정, 체크포인트) | JSON | 3주차_강의.json |

**중복 금지 규칙:**
- Phase 3에서 패턴 저장 시, 같은 내용이 memory/ feedback 파일에 이미 있으면 → patterns/에만 저장 (feedback 파일에 추가 금지)
- memory/ feedback 파일은 "왜 이런 결정을 했는지" 맥락용. patterns/는 "다음에 어떻게 할지" 규칙용.
- 새 세션에서 둘 다 로드하되, **patterns/ JSON이 규칙 적용의 기준** (feedback MD는 참고용)

---

## 세션 모드별 패턴 로드

| 모드 | 로드 범위 | 이유 |
|------|----------|------|
| **Plan-Light** | _common.json만 | 질문/논의에 무거운 패턴 불필요 |
| **Plan-Deep** | _common.json만 | 분석 단계, 아직 작업 목적 미확정 가능 |
| **Build** | _common.json + 목적별 파일 | 실작업에서 과거 교정/원칙 적용 필수 |

---

## 패턴 활용 (새 세션)

- 세션 시작 시 INDEX.json -> 매칭 파일 로드
- principles의 severity=critical -> 항상 적용
- corrections의 frequency 2+ -> 같은 실수 사전 차단
- rejections -> 해당 유형 콘텐츠 사전 필터링
