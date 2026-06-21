---
name: harness-check
description: Claude Code 플러그인을 비용·성능·보안 하네스 통제 관점에서 점검하고, 적용 가능(✅/△) 미적용 항목의 보완방안·로드맵을 도출하는 스킬. 플러그인 점검·하네스 진단·재점검·갱신·보완 요청 시 사용.
triggers:
  - harness-check
  - 하네스 점검
  - 하네스 체크
  - 하네스 진단
  - 플러그인 점검
  - 하네스 보완
argument-hint: "[대상 플러그인 경로]"
user-invocable: false
---

# harness-check — 플러그인 하네스 통제 점검 스킬

대상 Claude Code 플러그인의 비용·성능·보안 하네스 통제 중 **적용 가능(✅/△) 항목**을 6단계로 점검하고,  
미적용 항목의 보완방안·적용 로드맵을 도출함. 근거 문서: `${CLAUDE_PLUGIN_ROOT}/references/harness-checker.md`.

## 팀 역할 분담

| 역할 | 닉네임 | 담당 Phase | subagent_type |
|------|--------|-----------|---------------|
| 오케스트레이터 | 클로니 | Phase 0·1·3·4·5 | 메인 (직접 수행) |
| 비용 통제 점검가 | 코스티 | Phase 2 (비용 2장) | `cost-checker` |
| 성능 통제 점검가 | 스피디 | Phase 2 (성능 3장) | `performance-checker` |
| 보안 통제 점검가 | 가디 | Phase 2 (보안 4장) | `security-checker` |
| 구현 엔지니어 | 빌디 | Phase 4 (harness-apply 스킬 경유) | `implementation-engineer` |

> 서브에이전트는 `subagent_type` 네이티브 호출만 사용함 (수동 `.md` Read·임베딩 금지 — 이중 경로는 호출 충돌 유발).

---

## Phase 0. 컨텍스트 확인 `[클로니 — 직접 수행]`

- `"$ARGUMENTS"`(또는 입력)에서 `{대상경로}` 확보. 미입력 시 AskUserQuestion으로 대상 플러그인 경로 요청
- `{대상경로}/harness/harness-adoption-plan.md` 존재 여부 확인
  - 미존재 → 초기 점검 (Phase 1부터)
  - 존재 + 재점검/갱신 요청 → 기존 기록을 기준선으로 재점검

## Phase 1. 구성요소 인벤토리 + 적용 가능성 재판정 `[클로니 — 직접 수행]`

1. `{대상경로}`의 구성요소를 harness-checker 1장 표로 기재:  
   plugin.json·marketplace.json·skills·agents·commands·hooks·MCP·노출 tools·permissions·고위험 작업·외부 전송 경로
2. harness-checker 0-2장 규칙으로 각 항목의 적용 가능성(✅/△/❌)을 재판정하여 점검 범위 확정
   - ❌ 구조적 불가(7장) 항목은 점검 범위에서 제외 (대안만 Phase 3에서 안내)

## Phase 2. 병렬 점검 `[코스티 + 스피디 + 가디 — 단일 응답 동시 호출]`

> **핵심 규칙**: 세 점검가를 **단일 응답에서 동시에** Agent 호출함 (비용·성능·보안은 독립 도메인).

Agent 도구 호출 (subagent_type 네이티브, 3건 동시 실행):

```
subagent_type: "cost-checker"   · model: "sonnet"
prompt: |
  [목표] {대상경로}의 비용(2장) 적용 가능(✅/△) 항목 Y/N 판정·근거·보완방안 반환
  [입력정보] 대상경로={대상경로} · 인벤토리·적용가능성 재판정 결과(Phase 1)

subagent_type: "performance-checker" · model: "sonnet"
prompt: |
  [목표] {대상경로}의 성능(3장) 적용 가능(✅/△) 항목 Y/N 판정·근거·보완방안 반환
  [입력정보] 대상경로={대상경로} · 인벤토리·적용가능성 재판정 결과(Phase 1)

subagent_type: "security-checker" · model: "sonnet"
prompt: |
  [목표] {대상경로}의 보안(4장) 적용 가능(✅/△) 항목 Y/N 판정·근거·보완방안 반환
  [입력정보] 대상경로={대상경로} · 인벤토리·적용가능성 재판정 결과(Phase 1)
```

반환값 → `{cost_result}`·`{perf_result}`·`{security_result}`

## Phase 3. 기록부·로드맵 통합 + 보완 항목 선택 `[클로니 — 직접 수행]`

1. 세 결과를 harness-checker 5장 기록부 표로 통합 (영역별 대상·Y·N·적용률·미적용 Critical/High)
2. 미적용 항목을 심각도순(Critical→High→Medium→Low)·8장 로드맵 순(비용→성능→보안)으로 정렬
3. `{대상경로}/harness/harness-adoption-plan.md`에 점검 기록·Gap·로드맵 작성 (corp-research 형식 준용)
4. **영역별 도출**: 미적용(N)·✅/△ 항목을 비용/성능/보안으로 그룹화(각 영역 상세 항목·심각도·보완 요약)
5. AskUserQuestion을 **영역별 질문**(미적용 항목 있는 영역만, 최대 3개: 비용·성능·보안)으로 구성, 각 질문  
   `multiSelect: true`로 **구현할 상세 항목을 영역별 복수 선택**받음. 한 영역 항목 4개 초과 시 심각도순 4개씩  
   분할 질문(옵션 4개 한도). 전 영역 선택 0건이면 Phase 5로

## Phase 4. 선택 항목 구현 `[harness-apply 스킬로 위임]`

Phase 3에서 사용자가 선택한 항목이 있으면 `harness-apply` 스킬에 구현을 위임함:
- `${CLAUDE_PLUGIN_ROOT}/skills/harness-apply/SKILL.md`를 읽고 아래를 입력으로 실행
  - 대상경로={대상경로}
  - 선택 항목 목록={Phase 3에서 선택된 항목}
- harness-apply가 빌디(`implementation-engineer`)를 호출해 구현·검증하고, 결과를  
  harness-adoption-plan.md(N→Y·변경 이력)에 반영함

선택 항목이 없으면 Phase 5로 진행. 반환값 → `{impl_result}` (변경 파일·검증 결과 표)

## Phase 5. 완료 보고 `[클로니 — 직접 수행]`

```
[하네스 점검 완료]
■ 대상        : {대상경로}
■ 적용률      : 비용 NN% · 성능 NN% · 보안 NN%
■ 미적용 C/H  : (Critical·High 항목 목록)
■ 기록·로드맵 : {대상경로}/harness/harness-adoption-plan.md
■ 구현(빌디)  : (선택 항목·검증 결과) 또는 "구현 미선택"
```

---

## 입력 정보

| 항목 | 필수 | 기본값 | 설명 |
|------|------|--------|------|
| 대상 플러그인 경로 | 필수 | — | 점검할 Claude Code 플러그인 루트 경로 |

## 출력

| 산출물 | 경로 |
|--------|------|
| 점검 기록·적용 로드맵 | `{대상경로}/harness/harness-adoption-plan.md` |
| (구현 선택 시) 보완 적용 결과 | 대상 플러그인 내 변경 파일 |

## 제약조건

### MUST
- 점검 범위는 harness-checker.md의 ✅(적용 가능)·△(조건부) 항목으로 한정
- 각 점검가는 ⓪ 적용 가능성 재판정을 ② Y/N 판정보다 먼저 수행
- Phase 2 세 점검가는 단일 응답에서 동시 호출 (subagent_type 네이티브)
- 모든 Y 판정에 파일:라인 근거 병기 (근거 없는 Y는 무효)
- 점검 기록은 `{대상경로}/harness/harness-adoption-plan.md`에 저장
- 산출 문서(harness-adoption-plan.md)·완료 보고 내용은 사용자 친화적으로 쉽게 설명하고, 필요 시 비유·예시를 사용

### MUST NOT
- ❌ 구조적 불가(7장) 항목을 Y/N 판정 대상에 포함 (대안만 안내)
- 서브에이전트 `.md` 수동 Read·임베딩 호출 (subagent_type 네이티브만 사용)
- 근거 없는 완료 보고 (문서 작성 ≠ 점검 완료)

### 완료조건
- [ ] 세 점검가가 각 도메인 판정표를 반환함 (응답 텍스트 존재)
- [ ] `{대상경로}/harness/harness-adoption-plan.md`에 기록부·로드맵이 생성됨
- [ ] 미적용 Critical·High 항목이 심각도순으로 정렬·기재됨
- [ ] (구현 선택 시) 빌디의 변경 동작 검증 결과가 보고됨
