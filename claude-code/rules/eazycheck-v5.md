# EazyCheck v5.1

> 역할 분리: 설계 검증(DMAD 토론) + 실패 시뮬레이션(sim deep) + 도구(Check)
> Stage Final (check_final.py) 별도

---

## 설계 원칙

| 구분 | 담당 | 성격 |
|------|------|------|
| **GATE** | versions.json + patterns.json 로드 | 코드 전 1회 |
| **역추적** | 증상→경로→수정위치 확정 | 버그 키워드 감지 시, 1-5 파일 |
| **DMAD 토론** | 설계자(순방향) vs 사용자(역방향) 문답 | 코드 전, 3+ 파일 |
| **/sim** | 장기 실패 시뮬레이션 (3개월+10배) | 코드 후, 3+ 파일 자동 / 수동 호출 가능 |
| **Check** | pre-commit (도구 전용) | 코드 후 자동 |

---

## 통합 흐름

```
[사용자 요청]
    |
    v
[버그 키워드 감지?] 에러, 오류, 안되, 안됨, 문제, 실패, 버그, 왜, 500, 404, 터짐, crash, fail
    |
    +-- [YES + 1-5 파일] → 역추적 실행
    |     증상: {1줄} / 경로: {추적} / 수정 위치: {확정}
    |
    +-- [YES + 6+ 파일] → 스킵 (sim S2가 더 깊게 커버)
    |
    +-- [NO] → 스킵
    |
    v
[GATE] versions.json + patterns.json Read (1회)
    - 관련 패턴 경고 출력
    - 버전 확인
    |
    v
[파일 수 + 내용 판단]
    |
    +-- [1-2 파일] --> [코드 작성] --> [완료]
    |
    +-- [3-5 파일 + 신규기능/복합수정] --> [DMAD 자문자답 1라운드] --> [코드] --> [sim 1회] --> [완료]
    |
    +-- [3-5 파일 + 설정/배포/리팩토링] --> [토론 스킵] --> [코드] --> [sim 1회] --> [완료]
    |
    +-- [6+ 파일] --> [DMAD 토론 2라운드] --> [코드] --> [sim 루프 최대 2회] --> [완료]
```

### 3+ 파일 상세 흐름

```
[DMAD 토론] (코드 전, 설계자 vs 사용자 문답)
    [설계자 - 순방향] 기술적 최선 추구
      Q1. 더 단순한 방법이 있나?
      Q2. 기존 코드 어디와 부딪히나?
      Q3. 가장 위험한 가정 하나는?
    [사용자 - 역방향] 실제 사용 경험 관점
      Q1. 처음 보는 사람이 이 기능을 찾고 쓸 수 있나?
      Q2. 기대하는 가장 자연스러운 방식으로 동작하나?
      Q3. 실수하거나 실패했을 때 스스로 해결할 수 있나?
    규칙: 전부 수긍 금지, 매 라운드 반례 1개 이상
    v
[코드 작성]
    v
[sim deep] (코드 후, 장기 실패 시뮬레이션)
    S1. 3개월+10배 상황에서 코드를 따라가며 터지는 곳 찾기
    S2. 근본 원인 + 같은 유형 전체 탐색
    S3. 수정 -> 확인 시뮬레이션 (최대 2회 루프)
    -> 2회 후에도 실패: 사용자 승인 -> 디버깅 모드 전환
    v
[도착 후]
    - 오류 수정 (버그 발견 시)
    - 패턴 기록 (3Phase 패턴 저장)
    - 다음 세션에서 패턴 자동 로드
```

---

## GATE: 코드 전 정보 로드

> 기존 GATE 0/1/2 + 선제적 경고를 하나로 통합

### 실행 (필수)

```
Read: C:/Users/user/.claude/state/versions.json
Read: C:/Users/user/.claude/state/patterns.json
```

### 출력

```
[GATE 통과]
- 버전: Gemini default={값}, lowCost={값}
- 패턴: solved {N}개, preventive {M}개, mistakes {K}개
- 관련 경고: {있으면 패턴ID + symptom + solution}

코드 생성을 시작합니다.
```

### 규칙
- 세션 내 1회만 실행 (반복 불필요)
- 관련 패턴 발견 시 경고 출력 (코드 생성 차단 안 함)
- 세션 내 동일 패턴 2회 이상 발생 -> 근본 원인 재확인 요청

---

## GATE -1: Pre-Plan Gate

> Plan Mode에서 Build Mode로 전환할 때 반드시 검증

### 체크리스트
- [ ] 경쟁 제품/기존 방식 분석이 있는가?
- [ ] 근본 원인이 명확한가?
- [ ] 품질 기준이 구체적 수치로 있는가?
- [ ] 도메인 특화 규칙이 포함되었는가?
- [ ] 검증 방법이 구체적인가?

### 자동 트리거
- ExitPlanMode 실행 직전
- "이 플랜으로 진행" 키워드 감지 시

### 출력

```
[GATE -1 검증]
[ ] 경쟁/기존 분석: [OK/MISSING]
[ ] 근본 원인: [OK/MISSING]
[ ] 품질 기준: [OK/MISSING]
[ ] 도메인 규칙: [OK/MISSING]
[ ] 검증 방법: [OK/MISSING]

결과: {N}/5 통과
-> 3개 미만: 플랜 보완 필요
-> 3개 이상: Build 전환 가능 (미통과 경고)
-> 5개 전부: [GATE -1 통과]
```

---

## DMAD 토론 (3+ 파일, 코드 전) - 강제!

> DMAD = Diverse Multi-Agent Debate (ICLR 2025)
> 추론 방향이 다른 두 역할(설계자/사용자)이 문답하는 구조
> **3+ 파일 생성/수정 시 토론 없이 코드 작성 금지!**

### 규모별 방식
| 파일 수 | 변경 유형 | 방식 | 토큰 |
|---------|----------|------|------|
| 1-2 | 전체 | 토론 없음 | 최소 |
| 3-5 | 신규기능/복합수정 | DMAD 자문자답 (같은 컨텍스트) | 중간 |
| 3-5 | 설정/배포/리팩토링 | 토론 스킵 + sim만 | 절감 (~20%) |
| 6+ | 전체 | DMAD 토론 2라운드 | 높음 |

### DMAD 시작 전 필수 준비

**[강제] 토론 시작 전 변경 대상 파일 + 직접 의존 파일 전부 Read**
- Read 완료 후에만 토론 시작 (sim S1과 동일 원칙)
- 파일 읽기 없이 기억으로 코드 언급 금지
- 출력: `[DMAD 준비] {파일명} N개 Read 완료`

### 역할 정의

**설계자 (순방향: 설계 → 구현 → 결과 예측)**
- Q1. 이 문제를 해결하는 더 단순한 방법이 있나?
- Q2. 이 설계가 기존 코드의 어디와 부딪히나?
- Q3. 이 설계에서 가장 위험한 가정 하나는?
- Q4. 이 구현이 기존 코드베이스에 이미 있는 로직과 중복되는가?
- 규칙: 사용자 반박에 전부 수긍 금지, 최소 1개는 기술적 근거로 방어
- [필수] 문제 제기 시 반드시 해당 코드 라인 직접 인용: `파일명:라인: 코드내용` → 인용 없는 주장 금지

**사용자 (역방향: 실제로 쓸 때 → 이 설계로 가능한가?)**
- Q1. 처음 보는 사람이 이 기능을 찾고 쓸 수 있나?
- Q2. 사용자가 기대하는 가장 자연스러운 방식으로 동작하나?
- Q3. 실수하거나 실패했을 때 스스로 해결할 수 있나?
- Q4. 실패했을 때 사용자가 알 수 있나? 에러가 조용히 삼켜지는 곳은 없나?
- 규칙: 기술 용어를 모르는 최종 사용자 관점, 매 라운드 실사용 반례 1개 이상
- [필수] 문제 제기 시 반드시 해당 코드 라인 직접 인용: `파일명:라인: 코드내용` → 인용 없는 주장 금지

### 스킵 허용 조건
- 동일 패턴 반복 적용 (리네이밍, 포맷팅)
- 기존 플랜 기반 실행 중 (설계 검토 완료된 플랜)
- 사용자가 "토론 스킵" 명시 요청
- 설정/배포/단순 리팩토링 (신규 기능 없는 3-5파일 변경)
- 스킵 시 반드시 출력: `[토론 스킵] 사유: {구체적 사유}`

### 실행

```
[DMAD 토론]
[Round 1]
  설계자: 설계안 + 기술 근거
  사용자: 반박 + 실사용 반례 1개 이상

[Round 2]
  설계자: 반례 반영 수정안 (전부 수긍 금지, 최소 1개 방어)
  사용자: 수정안 재검토

[합의] 설계자 기술 근거 + 사용자 실사용 검증 둘 다 통과 -> 코드 작성 시작
```

### 출력

```
[토론 결과]
설계자 제안: {설계안 요약}
사용자 반박: {반례 + 지적}
최종 합의: {수정 반영된 최종안}
```

---

## Check: 도구 기반 검증 (코드 후)

> 순수 도구만. 시나리오 판단 없음.

### Check 1: Pre-commit Hooks

#### 도구
- Ruff: Python 린터/포매터
- check-ast: AST 검증
- Custom Rules: patterns.json 기반 하드코딩 탐지

#### 설정: `.pre-commit-config.yaml`
```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.14.11
    hooks:
      - id: ruff-check
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-ast
      - id: check-json
      - id: end-of-file-fixer
  - repo: local
    hooks:
      - id: custom-hardcode-check
        name: Check Hardcoded Extensions
        entry: python tools/check_hardcode.py
        language: system
        files: \.py$
```

#### 탐지 패턴
| 규칙 ID | 패턴 | 자동 수정 |
|---------|------|----------|
| hardcoded-extensions | `\.(mp3|m4a|wav|srt)` | os.path.splitext() 제안 |
| api-key-leak-check | `(AIzaSy|sk-|ghp_)` | [금지] 수동 수정 필요 |
| import-outside-try | 표준 라이브러리 try 안 import | [OK] 자동 이동 |

#### 한계
- `git commit --no-verify`로 우회 가능
- 우회 시 Check 2에서 동일 검사 재실행

---

### Check 2: GitHub Actions

#### 트리거
```
pull_request: [opened, synchronize]
```

#### 실행
1. Claude Code Action - AI 자동 코드 리뷰
2. auto_impact_checker.py - 파일 형식 변경 영향 분석
3. Bandit - 보안 스캔

#### 설정: `.github/workflows/claude-review.yml`
```yaml
name: Claude Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Claude Code Review
        uses: anthropics/claude-code-action@v1
        with:
          prompt: |
            Review for: hardcoded extensions, API key exposure, security vulnerabilities
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      - name: Impact Analysis
        run: python tools/auto_impact_checker.py
      - name: Security Scan
        run: |
          pip install bandit
          bandit -r . -ll
```

---

### Check 3: E2E 통합 테스트

#### 목적
API 계약 검증만으로는 발견 불가능한 내부 구현 하드코딩 탐지

```python
# API 계약: POST /api/upload  -> [OK] 통과
# 내부 구현: audio_path.replace('.mp3', '.srt')  -> [FAIL] M4A 실패
```

#### 테스트 위치
```
tests/e2e/test_patterns_validation.py   -> 16개 패턴 검증
tests/integration/                      -> 실제 API 동작 검증
```

#### CI 통합
```yaml
- name: E2E Tests
  run: |
    pip install pytest pytest-asyncio
    pytest tests/e2e/ tests/integration/ -v
```

---

## /sim: 장기 실패 시뮬레이션 (코드 후)

> 상세: skills/sim/SKILL.md
> "적용 후 장기 시뮬레이션 돌려서 문제점 및 보완점 체크하고 수정 후 정리해줘"

### 3단계 루프

```
S1. 실패 시뮬레이션
    - 3개월+10배 상황 설정 -> 코드를 한 줄씩 따라가며 터지는 곳 찾기
    - 추상적 "터질 수 있다" 금지 -> "이 파일 N번째 줄에서 터진다" 구체적으로

S2. 근본 원인
    - 터진 지점의 근본 원인 파악
    - 같은 유형이 코드 어디에 더 있나 전체 탐색 (grep)
    - CoVe 검증 통과한 항목만 보고

S3. 수정 + 확인
    - 근본 원인 기반 수정 (표면 땜질 금지)
    - 수정 후 git commit 필수: "[sim] {파일}:{라인} {수정 내용 한줄}"
    - 수정 후 S1 재시뮬레이션 (최대 2회 루프)
    - regression 체크: 수정 전 동작하던 기능이 수정 후에도 동작하는지 확인 (변경 실패율 +30% 방지)
    - 2회 후에도 실패: 사용자 승인 -> 디버깅 모드 전환
```

### 토론과의 역할 분리

| | DMAD 토론 (코드 전) | /sim (코드 후) |
|---|------|--------|
| 대상 | 설계안 (코드 없음) | 완성된 코드 |
| 방식 | 설계자 vs 사용자 문답 | 코드를 따라가며 실패 시뮬레이션 |
| 질문 | 더 단순한 방법? 기존과 충돌? 위험한 가정? | 3개월+10배에서 어디서 터지나? 근본 원인은? |
| 성격 | 방향 검증 | 구현 검증 |

---

## 3단계 방어 정리

| Layer | 구성요소 | 타이밍 | 우회 가능 |
|-------|---------|--------|----------|
| Layer 1 | Check 1 (pre-commit) | git commit | [OK] --no-verify |
| Layer 2 | Check 2 (GitHub Actions) | PR 생성 | [금지] CI 강제 |
| Layer 3 | Check 3 (E2E) | PR 생성 | [금지] CI 강제 |

---

## Stage Final: check_final.py 5단계

> EazyCheck/sim과 별개. 커밋 전 프로젝트 루트에서 실행.

```bash
python check_final.py
```

### 검증 순서

```
[1/5] Semgrep 정적 분석
      -> .semgrep/auto_rules.yml 기반
      -> API 키 노출, 하드코딩 패턴 탐지

[2/5] Hypothesis 속성 테스트
      -> tests/property_*.py 실행
      -> 현재: SKIP (테스트 파일 없음)

[3/5] E2E 테스트
      -> pytest tests/e2e/ 실행
      -> 16개 패턴 검증

[4/5] Mutation Testing (mutmut)
      -> 현재: SKIP (Windows 미지원)

[5/5] Ruff 코드 스타일
      -> 700+ 린트 규칙
      -> 자동 수정 가능 (--fix)
```

### 도구 현황

| 도구 | 상태 | 비고 |
|------|------|------|
| Semgrep | [OK] | 298개 이슈 탐지 |
| Hypothesis | [SKIP] | tests/property_*.py 필요 |
| E2E Tests | [OK] | 16/16 통과 |
| mutmut | [SKIP] | Windows 미지원 |
| Ruff | [WARN] | 스타일 이슈 (기능 무관) |

---

## 변경 이력

### v5.0 -> v5.1 (2026-03-16)

| 항목 | v5.0 | v5.1 | 효과 |
|------|------|------|------|
| 토론 분기 기준 | 파일 수만 | 파일 수 + 변경 유형 | 설정/배포 토큰 ~20% 절감 |
| 설계자 질문 | 3개 | 4개 (+코드 중복 탐색) | 코드 중복 4x 완전 커버 |
| 사용자 질문 | 3개 | 4개 (+silent failure) | 에러 핸들링 미흡 2x 강화 |
| S3 확인 시뮬레이션 | 재시뮬레이션만 | regression 체크 추가 | 변경 실패율 +30% 대응 |
| AI 코드 오류 커버리지 | 5.5/6개 | 7.5/8개 | CodeRabbit 데이터 기반 |

### v4 -> v5.0 (EazyCheck) 변경 요약 (2026-03-15)

| v4 | v5 (EazyCheck) | 변경 |
|----|----|------|
| GATE 0/1/2 + Pillar 1 + auto_warning | GATE 통합 1개 | 3곳 -> 1곳 |
| 토론 (architect + reviewer 병렬) | DMAD 토론 (설계자 vs 사용자 문답) + 규모별 방식 | 전환 |
| Pillar 1 (경고) | GATE에 흡수 | 제거 |
| Pillar 2 (pre-commit) | Check 1 (pre-commit) | 개명 |
| Pillar 3 (GitHub Actions) | Check 2 (GitHub Actions) | 개명 |
| Pillar 4 (/v 종합) | sim + Check 도구에 분산 | 제거 |
| Pillar 5 (E2E) | Check 3 (E2E) | 개명 |
| Pillar 6 (장기 영향) | sim deep (3단계 실패 시뮬레이션 루프, 최대 2회) | 전환 |
| sim_trigger.py | 제거 | 에이전트 자동 체인 |
| code_review_automation | 토론 + Check에 분산 | 제거 |

---

## 관련 파일

| 파일 | 용도 |
|------|------|
| `~/.claude/state/patterns.json` | 패턴 DB |
| `~/.claude/state/versions.json` | 버전 잠금 |
| `~/.claude/state/routing.json` | 토론 패턴 + 라우팅 |
| `~/.claude/skills/sim/SKILL.md` | /sim 시나리오 스킬 |
| `check_final.py` | Stage Final 실행 스크립트 |
| `.pre-commit-config.yaml` | Pre-commit 설정 |
| `.github/workflows/claude-review.yml` | GitHub Actions |
| `tests/e2e/` | E2E 테스트 |
