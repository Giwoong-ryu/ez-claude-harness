# Claude Harness

Claude Code가 문제를 잘못 고치거나 고쳤다가 다른 곳을 망가뜨리는 걸 막는 규칙 파일 모음입니다.

CLAUDE.md + 라우팅 설정 + 검증 규칙으로 구성되며, 단일 프로젝트에서 실제로 운용하며 다듬어온 설정입니다.

**핵심 가치: 버그가 발생하기 전에 잡습니다.** 코드 완성 후 배포 전, 실패 시나리오를 미리 돌려 잠재적 문제를 찾고 근본 원인까지 수정합니다. 코드를 잘 모르는 상태에서 AI에게 맡기는 경우에도 유효합니다. 표면만 고치고 끝내는 패턴을 구조적으로 차단하기 때문에, 직접 원인을 추적하기 어려울수록 더 크게 작동합니다.

---

## 왜 만들었나

Claude Code를 쓰다 보면 반복되는 실패 패턴이 있습니다:

- 파일을 읽지 않고 기억으로 설명하다가 틀림 (할루시네이션)
- 증상 위치만 고치고 근본 원인은 그대로 (증상 패치)
- 3+ 파일 변경 시 설계 검토 없이 코드 작성 후 구조 문제 뒤늦게 발견
- 수정 후 같은 유형 버그가 다른 파일에서 재발

이 패턴들을 구조적으로 차단하는 게 이 하네스의 목적입니다.

외부 참고로, [2026년 CodeRabbit 리포트](https://www.coderabbit.ai/blog/ai-code-review-stats)는 AI 생성 코드가 인간 대비 전체 이슈 1.7배, 보안 취약점 2.74배, 에러 핸들링 미흡 2배임을 측정했습니다. 이 수치는 AI가 생성 속도 대비 검증 절차를 거치지 않을 때 발생하는 문제를 잘 보여줍니다.

---

## 구성요소

| 단계 | 이름 | 역할 | 트리거 |
|------|------|------|--------|
| 0 | **역추적** | 버그 근본 원인 확정 | 에러/오류 키워드 감지 |
| 1 | **GATE** | 버전 + 패턴 로드, 관련 경고 출력 | 코드 생성 전 1회 |
| 2 | **DMAD 토론** | 설계자 vs 사용자 관점 문답 | 3+ 파일, 신규기능/복합수정 |
| 3 | **sim** | 3개월+10배 장기 실패 시뮬레이션 | 3+ 파일 변경 후 자동 |
| 4 | **Check** | pre-commit + CI + E2E 도구 검증 | 코드 생성 후 |
| 5 | **smartLoop** | 실패 유형별 최소 지점에서 재시작 | sim/Check 실패 시 |

### 역추적 (Reverse Trace)

버그 수정 전 근본 원인을 확정합니다.

```
[역추적]
증상: {에러/문제 1줄}
경로: {어떤 입력 → 어디를 거쳐 → 여기서 터짐}
수정 위치: {근본 원인 위치 확정}
→ 수정 위치 확정 후 코드 작성
```

트리거: `에러, 오류, 안되, 문제, 실패, 버그, 500, 404, crash, fail`
1-5 파일에 적용. 6+ 파일은 sim S2가 더 깊게 커버.

### DMAD 토론

코드 작성 전, 두 관점이 문답합니다.

**설계자 (순방향 - 기술 관점):** 더 단순한 방법? 기존 코드와 충돌? 가장 위험한 가정?

**사용자 (역방향 - 실사용 관점):** 처음 보는 사람이 쓸 수 있나? 실패했을 때 알 수 있나?

핵심 규칙:
- 토론 시작 전 변경 대상 파일 전부 Read 필수
- 문제 제기 시 `파일명:라인: 코드내용` 형식으로 직접 인용
- 인용 없는 주장 금지 (오독 할루시네이션 전파 차단)

이 인용 규칙이 왜 필요한지는 [`docs/dmad-false-positive-analysis.md`](docs/dmad-false-positive-analysis.md)에 실제 케이스와 함께 기록했습니다.

### sim (장기 실패 시뮬레이션)

완성된 코드를 "3개월 후 + 규모 10배" 상황에서 따라가며 터지는 곳을 찾습니다.

```
S1. 실패 시뮬레이션 - 코드 Read 후 시나리오 추적
S2. 근본 원인 탐색 - "이걸 고치려면 다른 걸 먼저 고쳐야 하는가?" 루프로 최하위 레이어까지
S3. 수정 + 확인 시뮬레이션 (루프 최대 2회)
```

**가장 큰 장점은 아직 발생하지 않은 버그를 배포 전에 잡는다는 점입니다.** 실제로 터진 후 원인을 찾는 게 아니라, 코드가 완성된 시점에 미리 시나리오를 돌립니다.

S2에서 표면 원인에서 멈추지 않고 수정 가능한 최하위 레이어까지 강제로 탐색합니다. 증상을 우회하는 fallback 수정은 차단하고, 실제 원인 위치만 수정합니다.

코드를 잘 모르는 상태에서 AI에게 맡길 때 특히 유용합니다. AI가 표면 증상만 고치고 더 깊은 원인을 남겨두는 패턴을 구조적으로 차단하기 때문에, 직접 근본 원인을 추적하기 어려운 상황에서도 올바른 수정이 이루어집니다.

`/sim` 명령으로 언제든 수동 실행도 가능합니다.

### smartLoop

실패 시 전체를 처음부터 재실행하지 않고 실패 유형별 최소 지점에서 재시작합니다.

| 실패 유형 | 재시작 지점 | 토큰 비용 |
|----------|------------|---------|
| lint 오류 | 직접 수정만 | ~50 |
| CI 실패 | 역추적만 재실행 | ~70 |
| sim FAIL | sim S2부터 | ~150 |
| 설계 결함 | 전체 재시작 | ~500 |

최대 3회. 3회 후에도 실패 시 사용자에게 보고하고 승인을 기다립니다.

---

## 파일 구조

```
claude-harness/
├── CLAUDE.md                            # 시작점 (역추적 + 파일 수별 검증 흐름)
├── state/
│   └── routing.json                     # DMAD 역할 정의 + smartLoop + 트리거
├── rules/
│   ├── eazycheck-v5.md                  # GATE + DMAD + sim + Check 상세
│   └── pattern-system.md               # 반복 실수 축적 시스템 (3Phase)
├── skills/
│   └── sim/SKILL.md                     # /sim 장기 실패 시뮬레이션 절차
└── docs/
    └── dmad-false-positive-analysis.md  # DMAD 오탐 유형, 차단 방법, 한계
```

---

## 설치

### 핵심만 (역추적 + 검증 흐름)

`CLAUDE.md`를 `~/.claude/CLAUDE.md`에 병합합니다.

### 전체

```bash
cp -r rules/ ~/.claude/rules/
cp -r skills/ ~/.claude/skills/
cp -r state/ ~/.claude/state/
# CLAUDE.md는 기존 파일에 병합 또는 교체
```

규칙 파일 내 경로는 본인 환경에 맞게 수정하세요.

---

## 한계

- 단일 프로젝트(Python/FastAPI)에서 운용하며 만든 설정입니다. 다른 스택에서는 조정이 필요할 수 있습니다.
- DMAD와 sim은 Claude가 실제 파일을 Read해야 동작합니다. 컨텍스트 창 부족 시 효과가 줄어듭니다.
- 규칙이 많을수록 토큰 비용이 증가합니다. 필요한 부분만 선택해서 쓰는 게 낫습니다.
- DMAD의 도메인 지식 부재 오탐은 여전히 발생합니다. ([상세](docs/dmad-false-positive-analysis.md))

---

## 참고

- [OpenAI Harness Engineering](https://openai.com/index/harness-engineering/)
- [Martin Fowler - Exploring Generative AI](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html)
- [Toss Tech - Harness for Team Productivity](https://toss.tech/article/harness-for-team-productivity)
- [CodeRabbit AI Code Review Stats 2026](https://www.coderabbit.ai/blog/ai-code-review-stats)
