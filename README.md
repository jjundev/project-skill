# project-skill — Claude Code 스킬 파이프라인

Claude Code용 스킬 세트를 큐레이션한 레포입니다. 스킬들은 `.claude/skills/`에 있으며,
사용자의 전역(`~/.claude/skills/`)과 **양쪽 동기화**(dual-location)로 관리됩니다.

핵심은 **하나의 경계**입니다:

> **grill 시리즈 = 원하는 goal을 구체화·검증하는 작업 (아이디어/설계).**
> **writing-plans 이하 = 그 goal을 실제로 구현하기 위한 계획을 세우고, 짓고, 검증하는 작업.**

---

## 핵심 철학: goal ↔ 구현 분리

| | **grill 시리즈** | **구현 파이프라인 (writing-plans ~)** |
|---|---|---|
| 하는 일 | 원하는 goal을 질문으로 파고들어 **설계로 구체화**(+검증) | 확정된 설계를 **구현 계획으로 번역**하고 실행·검증 |
| 다루는 것 | 아이디어 / 설계 — *무엇을, 왜* | 구현 — *어떻게, 단계별로* |
| 대표 산출물 | `# Design` + 결정표 + `## Design spec` | 태스크별 실행 대본 (실패테스트 → 실제코드 → 커밋) |
| 코드 | 건드리지 않음 (read-only) | 실제로 작성·수정 |

"계획(plan)"이라는 단어의 혼동을 피하려고, **grill-yourself의 출력은 `Design`**(설계),
**writing-plans의 출력은 `Plan`**(실행 계획)으로 이름부터 구분합니다.

---

## 전체 파이프라인

```
 ┌─────────────── grill 시리즈 (goal 구체화) ───────────────┐   ┌──────────── 구현 파이프라인 ────────────┐

  grill-yourself  ──▶  grill-review  ──▶  writing-plans  ──▶  reviewing-plans  ──▶  subagent-driven  ──▶  verifying-implementation
  (goal→설계)          (설계 검증)         (구현 계획 수립)      (계획 검토)            (실제 구현)            (구현 검증)
```

- **grill-yourself / grilling** — goal을 파고들어 설계로 구체화 (자율 / 대화형)
- **grill-review** — 그 설계가 타당한지 fresh 컨텍스트로 적대 검증
- **writing-plans** — 확정된 설계를 태스크별 실행 대본으로 번역
- **reviewing-plans** — 그 실행 계획을 구현 전 검토 *(구 grill-plan)*
- **subagent-driven-development** — 태스크별 서브에이전트로 실제 구현
- **verifying-implementation** — 구현된 코드를 계획 대비 검증 *(구 grill-verify)*

> `reviewing-plans`와 `verifying-implementation`은 **구현 산출물**(계획·코드)을 검토하므로
> grill 계열이 아니며, 이번 구조 정리에서 `grill-` 접두어를 뗐습니다. grill-\*는 이제
> **아이디어 전용**입니다.

---

## 스킬 카탈로그

발동방식: **자동** = 상황에 맞으면 Claude가 스스로 호출 · **명시** = `/스킬명`으로 직접 호출

### 1. goal 구체화 — grill 시리즈 (아이디어 전용)
| 스킬 | 발동 | 설명 |
|---|---|---|
| `grilling` | 자동 | 계획/설계를 사용자와 한 질문씩 집요하게 인터뷰 |
| `grill-me` | 명시 | `/grilling` 세션 실행 (별칭) |
| `grill-yourself` | 명시 | 자율 self-인터뷰 → 결정표 + `Design spec` 산출 |
| `grill-review` | 명시 | 설계(결정표)를 fresh 컨텍스트로 적대 검증 |
| `grill-with-docs` | 명시 | grilling + ADR/용어집 문서화 |
| `grill-yourself-with-docs` | 명시 | grill-yourself + ADR/용어집 문서화 |

### 2. 구현 계획 & 검토
| 스킬 | 발동 | 설명 |
|---|---|---|
| `writing-plans` | 자동 | 설계를 태스크별 TDD 실행 대본으로 작성 |
| `reviewing-plans` | 명시 | 실행 계획을 구현 전 검토 (추적성/분해/완결성/인터페이스/실행가능성) |

### 3. 구현 실행
| 스킬 | 발동 | 설명 |
|---|---|---|
| `subagent-driven-development` | 자동 | 태스크별 서브에이전트 구현 + 리뷰 루프 (기본 엔진) |
| `executing-plans` | 자동 | 계획을 현재 세션에서 직접 실행 (서브에이전트 대안) |
| `using-git-worktrees` | 자동 | 격리 워크스페이스 확보 |
| `finishing-a-development-branch` | 자동 | 구현 후 merge/PR/정리 |
| `dispatching-parallel-agents` | 자동 | 독립 작업 병렬 서브에이전트 |

### 4. 구현 검증
| 스킬 | 발동 | 설명 |
|---|---|---|
| `verifying-implementation` | 명시 | 구현 코드를 계획 대비 정적+동적 검증 |
| `requesting-code-review` | 자동 | 리뷰어 서브에이전트에게 코드 리뷰 요청 |
| `receiving-code-review` | 자동 | 리뷰 피드백 수용 규율 (맹목 수용 금지) |

### 5. 규율 & 기법
| 스킬 | 발동 | 설명 |
|---|---|---|
| `tdd` | 자동 | 테스트 우선 개발 (설계 품질 중심) |
| `test-driven-development` | 명시 | TDD 규율 (방탄) — subagent-driven 엔진 전용, 자동발동 끔 |
| `diagnosing-bugs` | 자동 | 하드 버그 진단 (피드백 루프 우선) |

### 6. 설계 & 도메인
| 스킬 | 발동 | 설명 |
|---|---|---|
| `codebase-design` | 자동 | 깊은 모듈 설계 어휘 |
| `domain-modeling` | 자동 | 도메인 용어(CONTEXT.md) · ADR 유지 |
| `improve-codebase-architecture` | 명시 | 아키텍처 심화 기회 스캔 → HTML 리포트 → grill |
| `prototype` | 명시 | 버릴 프로토타입으로 설계 질문 해소 |
| `writing-great-skills` | 명시 | 스킬 작성 레퍼런스(어휘·원칙) |

### 7. 제품 파이프라인 (이슈 트래커)
| 스킬 | 발동 | 설명 |
|---|---|---|
| `to-prd` | 명시 | 대화를 PRD로 합성 → 트래커 발행 |
| `to-issues` | 명시 | 계획/PRD를 tracer-bullet 슬라이스 이슈로 분해 |

---

## 사용법

- **명시 호출**: `/grill-yourself`, `/reviewing-plans`, `/verifying-implementation` 처럼 슬래시로 직접 실행.
- **자동 발동**: `writing-plans`·`subagent-driven-development`·`tdd`·`diagnosing-bugs` 등은
  상황이 맞으면 Claude가 알아서 호출.
- **시각화**: `grill-yourself` / `grill-review` / `reviewing-plans`는 `--viz` 플래그로 결과를
  HTML 아티팩트 대시보드로도 렌더링 (공유 절차는 `_shared/grill-viz.md`).

## 대표 흐름 예시

```
"결제 재시도 로직을 넣고 싶다"
  → /grill-yourself        # goal을 파고들어 설계 확정 (재시도 횟수·백오프·멱등성…)
  → /grill-review          # 설계 적대 검증
  → writing-plans          # 태스크별 실행 대본 (자동)
  → /reviewing-plans       # 계획 검토
  → subagent-driven-development   # 태스크별 구현 (자동)
  → /verifying-implementation     # 구현을 계획 대비 검증
```

## 구조 메모

- 스킬은 `.claude/skills/<name>/SKILL.md`. 공유 자산은 `_shared/`.
- 프로젝트 사본과 전역(`~/.claude/skills/`) 사본을 **동기화**해서 씁니다. 스킬을 수정하면 양쪽에 반영합니다.
