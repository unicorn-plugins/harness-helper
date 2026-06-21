---
name: harness-apply
description: 점검 기록(harness-adoption-plan.md)의 미적용(✅/△) 하네스 통제 항목을 사용자가 언제든 복수 선택하여 대상 Claude Code 플러그인에 보완 구현하는 스킬. 미적용 항목 구현·보완 적용·추가 적용 요청 시, 또는 harness-check가 선택 항목을 위임할 때 사용.
triggers:
  - harness-apply
  - 하네스 보완
  - 하네스 적용
  - 미적용 구현
  - 미구현 항목 구현
  - 보완 적용
argument-hint: "[대상 플러그인 경로]"
user-invocable: false
---

# harness-apply — 미적용 하네스 통제 보완 구현 스킬

점검 기록(`{대상경로}/harness/harness-adoption-plan.md`)에 남은 **미적용(N)·적용 가능(✅/△) 항목**을  
사용자가 **언제든 복수 선택**하여 대상 플러그인에 보완 구현함. harness-check 직후뿐 아니라 추후 세션에서도  
독립 실행 가능. 구현은 구현 엔지니어(빌디)가 수행하고, 결과를 점검 기록에 반영함.

## 팀 역할 분담

| 역할 | 닉네임 | 담당 Phase | subagent_type |
|------|--------|-----------|---------------|
| 오케스트레이터 | 클로니 | Phase 0·1·2·4·5 | 메인 (직접 수행) |
| 구현 엔지니어 | 빌디 | Phase 3 (선택 항목 구현·검증) | `implementation-engineer` |

> 서브에이전트는 `subagent_type` 네이티브 호출만 사용함 (수동 `.md` Read·임베딩 금지).

---

## Phase 0. 컨텍스트 확인 `[클로니 — 직접 수행]`

- `"$ARGUMENTS"`(또는 입력)에서 `{대상경로}` 확보. 미입력 시 AskUserQuestion으로 요청
- **harness-check가 위임한 선택 항목 목록**이 입력으로 전달됐으면 → Phase 3으로 직행(선택 단계 생략)
- `{대상경로}/harness/harness-adoption-plan.md` 존재 여부 확인
  - 존재 → Phase 1
  - 미존재 → `harness-check` 스킬을 먼저 호출해 점검 기록을 생성한 뒤 Phase 1
    (`${CLAUDE_PLUGIN_ROOT}/skills/harness-check/SKILL.md`를 읽어 {대상경로} 점검 실행)

## Phase 1. 미적용 항목 로드 `[클로니 — 직접 수행]`

- harness-adoption-plan.md에서 **미적용(N)·적용 가능(✅/△)** 항목을 추출하여 선택지 목록 구성
- 각 항목에 영역(비용/성능/보안)·심각도(Critical→High→Medium→Low)·보완방안 요약 병기
- 남은 미적용 항목이 없으면 → "보완할 미적용 항목 없음" 보고 후 종료

## Phase 2. 복수 선택 `[클로니 — 직접 수행]`

- AskUserQuestion(`multiSelect: true`)으로 구현할 항목을 **복수 선택**받음
  - 선택지는 심각도순 정렬, 각 옵션에 영역·심각도·보완 요약 표기
- 선택 0건이면 종료, 1건 이상이면 Phase 3

## Phase 3. 선택 항목 구현 `[빌디 서브에이전트]`

Agent 도구로 호출 (subagent_type 네이티브):

```
subagent_type: "implementation-engineer" · model: "opus"
prompt: |
  [목표] 선택된 적용 가능(✅/△) 항목을 6장 레시피로 {대상경로}에 구현 후 동작 검증
  [입력정보] 대상경로={대상경로} · 선택 항목 목록 · 각 항목 근거·보완방안(harness-adoption-plan.md)
```

반환값 → `{impl_result}` (변경 파일·이벤트/타입·검증 결과 표)

## Phase 4. 점검 기록 갱신 `[클로니 — 직접 수행]`

- 구현·검증에 성공한 항목의 상태를 harness-adoption-plan.md에서 N→Y로 갱신, 근거(변경 파일:라인) 기재
- 변경 이력 테이블에 적용 일자·항목·검증 결과 추가
- 검증 실패·보류 항목은 사유와 함께 미적용으로 유지

## Phase 5. 완료 보고 `[클로니 — 직접 수행]`

```
[보완 구현 완료]
■ 대상        : {대상경로}
■ 구현 항목   : (항목·검증 결과 목록)
■ 갱신 기록   : {대상경로}/harness/harness-adoption-plan.md
■ 남은 미적용 : (영역·심각도별 잔여 항목 수)
```

---

## 입력 정보

| 항목 | 필수 | 기본값 | 설명 |
|------|------|--------|------|
| 대상 플러그인 경로 | 필수 | — | 보완할 Claude Code 플러그인 루트 경로 |
| 선택 항목 목록 | 선택 | — | harness-check 위임 시 전달(있으면 선택 단계 생략) |

## 출력

| 산출물 | 경로 |
|--------|------|
| 보완 적용 결과(변경 파일) | 대상 플러그인 내 변경 파일 |
| 갱신된 점검 기록 | `{대상경로}/harness/harness-adoption-plan.md` |

## 제약조건

### MUST
- 적용 가능(✅/△) **미적용(N)** 항목만 구현 대상에 포함
- 사용자 복수 선택(AskUserQuestion `multiSelect: true`)을 거쳐 선택된 항목만 구현
- 빌디는 각 변경 후 동작 검증 증거 확보 (node 실행·JSON 파싱·로그 적재)
- 구현 결과를 harness-adoption-plan.md에 N→Y·변경 이력으로 반영
- 산출 내용·완료 보고는 사용자 친화적으로 쉽게 설명하고, 필요 시 비유·예시를 사용

### MUST NOT
- ❌ 구조적 불가(7장)·이미 적용(Y) 항목을 구현 대상에 포함
- 서브에이전트 `.md` 수동 Read·임베딩 호출 (subagent_type 네이티브만 사용)
- 검증 없는 완료 보고 (구현 ≠ 동작 확인)

### 완료조건
- [ ] 사용자가 복수 선택한 항목이 빌디에게 전달됨 (위임 시 전달된 목록 사용)
- [ ] 빌디의 변경 동작 검증 결과(통과/실패)가 항목별로 보고됨
- [ ] harness-adoption-plan.md의 해당 항목 상태·변경 이력이 갱신됨
