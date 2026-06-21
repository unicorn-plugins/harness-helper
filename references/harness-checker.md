# 플러그인 하네스 적용 검사기 (harness-checker)

> 목적: 개발된 Claude Code 플러그인을 **비용·성능·보안 하네스 통제** 관점에서 점검하고, 미적용 항목의  
> 구현 방안을 도출하는 **재사용 진단 도구**임. 플러그인을 만들 때마다 본 문서로 반복 점검함.  
> 기준선 일자: 2026-06-21 · 근거 문서: [harness-engineering-guide.md](harness-engineering-guide.md),
> [harness-hooks.md](harness-hooks.md)  
> 대상: 플러그인 개발자(점검 수행자)

**두 개의 축**  
- **적용 가능성(✅/△/❌)**: 플랫폼·플러그인 구조가 결정하는 *구현 가능 여부*. 0-2장 규칙으로 재판정·갱신함.  
- **적용 여부(Y/N)**: *이 플러그인의 현재 상태*. 매 점검 시 근거(파일:라인)와 함께 평가함.

---

## 0. 검사기 사용 방법

플러그인 개발 완료 후 아래 6단계로 적용함.

| 단계 | 행위 | 산출 |
|------|------|------|
| ⓪ 적용 가능성 재판정 | 1장 인벤토리를 토대로 각 항목의 ✅/△/❌를 [0-2장 규칙](#0-2-적용-가능성-재판정-규칙)으로 확인 | 갱신된 체크 항목 집합 |
| ① 구성요소 인벤토리 | 플러그인의 skills·agents·commands·hooks·MCP·tools·permissions를 [1장](#1-플러그인-구성요소-인벤토리) 표에 기재 | 점검 범위 확정 |
| ② Y/N 판정 | 2~4장 체크리스트의 "판정 기준"과 플러그인을 대조해 적용 여부 표기 | 1차 평가 |
| ③ 근거 기재 | 각 Y/N에 **파일:라인** 증거 기재 (근거 없는 Y는 무효) | 검증 가능 평가 |
| ④ 미적용 구현 | N 항목을 [6장](#6-미적용-하네스-구현-방법) 레시피로 보완 | hook·설정 추가 |
| ⑤ 우선 보완·기록 | 심각도순으로 보완, [5장 기록부](#5-평가-결과-기록부) 갱신, [8장 로드맵](#8-적용-로드맵) 반영 | 적용 방안 |

---

## 0-2. 적용 가능성 재판정 규칙

기준선(아래 2~4장·7장의 ✅/△/❌)은 **고정 박제가 아님**. 자기 플러그인 구조·Claude Code 버전에 맞게  
다음 3분류 결정 규칙으로 각 항목을 다시 분류함.

| 판정 | 결정 규칙 |
|------|-----------|
| ✅ 적용 가능 | hook·설정으로 **매 호출 경계에서 결정론적 강제** 가능(deny 지점·해당 이벤트 존재) |
| △ 조건부 | 호출·도구 경계로만 가능, **외부 상태 조립** 필요, 또는 **모델 준수에 의존**(soft) |
| ❌ 구조적 불가 | **생성 도중 제어·연속 wall-clock·OS·네트워크·인프라·전송 계층** 필요(플러그인 범위 밖) |

**승격 규칙(△→✅)**: 대상 플러그인 구조가 단일 게이트를 강제하면 승격 가능함.  
- 예시: 네트워크 화이트리스트 = 기준선 △(도구 deny만) → 플러그인이 **모든 외부 전송을 단일 MCP 도구로 경유**하면  
  그 도구 내부에서 도메인 화이트리스트를 강제 가능 → **✅로 승격**.

**기준선 갱신**: Claude Code가 새 hook 이벤트·필드를 추가하면 [harness-hooks.md](harness-hooks.md)를 재확인해  
본 문서의 기준선과 일자를 갱신함. ❌였던 항목이 플랫폼 변화로 ✅/△가 될 수 있음.

---

## 1. 플러그인 구성요소 인벤토리

점검 전 대상 플러그인의 구성요소를 기재함. (없으면 "없음")

| 구성요소 | 위치(경로) | 내용 요약 |
|----------|------------|-----------|
| plugin.json / marketplace.json | `.claude-plugin/` | (기재) |
| skills | `skills/` | (기재) |
| agents | `agents/` | (기재) |
| commands | `commands/` | (기재) |
| hooks | `hooks/hooks.json` | (이벤트·matcher 목록 기재) |
| MCP 서버 | `.mcp.json` 등 | (기재) |
| 노출 tools | (skill/agent frontmatter) | (도구 목록 기재) |
| permissions | settings / frontmatter | (allow/deny 규칙 기재) |
| 고위험 작업 | (식별) | 삭제·결제·외부전송·대량처리 해당 도구 (기재) |
| 외부 전송 경로 | (식별) | WebFetch·이메일·API 등 egress 지점 (기재) |

---

## 2. 비용(Cost) 체크리스트

판정: `☐ Y ☐ N` · 근거 칸에 파일:라인 기재 · 미적용 시 6장 동일 항목 레시피 적용.

### 2.A 필수 — ✅ 적용 가능

| 체크 항목 | 판정 기준(무엇을 보면 적용) | 적용 | 근거(파일:라인) | 심각도 | 미적용 조치 |
|-----------|------------------------------|------|------------------|--------|-------------|
| 최대 반복 횟수 | 서브에이전트 `maxTurns` 지정 또는 `Stop` hook 완료 게이트 존재 | ☐ Y ☐ N | | High | →6.2 |
| 반복 패턴 감지 | 동일 `tool_input` 반복을 상태파일로 비교·차단하는 hook 존재 | ☐ Y ☐ N | | High | →6.2 |
| 부분 결과 반환 | 미완 시에도 현재까지 산출·로그를 반환하도록 설계됨 | ☐ Y ☐ N | | High | →6.2 |
| 입력 절삭 | `PreToolUse updatedInput`으로 과대 입력을 절삭함 | ☐ Y ☐ N | | Medium | →6.2 |
| 핸드오프 요약 전달 | 서브에이전트가 전체 이력이 아닌 최종 요약만 반환함 | ☐ Y ☐ N | | Medium | →6.2 |
| 요청별 사용량 측정 | `PostToolUse(Agent)`로 `totalTokens`·`usage`를 기록함 | ☐ Y ☐ N | | Medium | →6.2 |
| 호출 수 제한 | 고비용 도구 호출 카운터+상한 초과 시 `deny` hook 존재 | ☐ Y ☐ N | | Critical | →6.2 |

### 2.B 조건부 — △ (한계 인지 후 적용)

| 체크 항목 | 판정 기준 | 적용 | 근거(파일:라인) | 심각도 | 한계(가능/불가) |
|-----------|-----------|------|------------------|--------|------------------|
| 이력 요약 | `PreCompact` hook·입력 절삭으로 이력 축소 | ☐ Y ☐ N | | Medium | 가능: 축소·아카이브 / 불가: 요약 품질 강제(soft) |
| Budget 한도 | 누적 비용 적재 후 초과 시 후속 호출 `deny` | ☐ Y ☐ N | | Critical | 가능: 호출 경계 게이트 / 불가: 생성 중단 |
| Kill-switch | 폭주 플래그 시 고비용 도구 일괄 `deny` | ☐ Y ☐ N | | Critical | 가능: 추가 작업 차단 / 불가: 진행 중 토큰 절단 |
| 동시 실행 제한 | `flock` in-flight 카운터로 동시성 상한 근사 | ☐ Y ☐ N | | Critical | 가능: 경계 근사 / 불가: 완전 강제 |
| 깊이 제한 | 서브에이전트에 `Agent` 도구 미부여 또는 깊이 검사 | ☐ Y ☐ N | | Critical | 가능: 추가 스폰 차단 / 불가: 임의 숫자 깊이(런타임 고정) |

---

## 3. 성능(Performance) 체크리스트

### 3.A 필수 — ✅ 적용 가능

| 체크 항목 | 판정 기준 | 적용 | 근거(파일:라인) | 심각도 | 미적용 조치 |
|-----------|-----------|------|------------------|--------|-------------|
| Circuit Breaker | `PostToolUseFailure` 실패 카운트+임계 초과 시 `deny` hook | ☐ Y ☐ N | | High | →6.2 |
| 병렬 실행 | 독립 조회를 병렬 호출(읽기전용 도구 `readOnlyHint`) | ☐ Y ☐ N | | Low | →6.2 |
| 스키마 검증 | 출력 JSON 스키마 검증 실패 시 `block` 게이트 | ☐ Y ☐ N | | Medium | →6.2 |
| 교차 검증 | 다중 검증자 비교 후 불일치 시 차단 구조 | ☐ Y ☐ N | | Medium | →6.2 |
| 검증 실패 차단 | 근거·테스트 검증 실패 시 다음 단계 `block` | ☐ Y ☐ N | | Medium | →6.2 |

### 3.B 조건부 — △

| 체크 항목 | 판정 기준 | 적용 | 근거(파일:라인) | 심각도 | 한계(가능/불가) |
|-----------|-----------|------|------------------|--------|------------------|
| Graceful Degradation | `PostToolUseFailure`+`additionalContext`로 대체 응답 지침 | ☐ Y ☐ N | | High | 가능: 복구 지침 주입 / 불가: 폴백 내용 강제(soft) |
| 리소스 격리 | 위험 작업을 서브에이전트로 컨텍스트 격리 | ☐ Y ☐ N | | High | 가능: 컨텍스트 격리 / 불가: 인프라(CPU·메모리) 격리 |
| 캐싱 | MCP·스크립트로 조회 결과 캐시·재사용 | ☐ Y ☐ N | | Low | 가능: 직접 구축 / 불가: 프롬프트 캐시 제어(런타임 자동) |
| Grounding | 근거(출처·조항) 존재 검증 후 누락 시 `block` | ☐ Y ☐ N | | Medium | 가능: 존재 검증 / 불가: 정확성 강제(soft) |

---

## 4. 보안(Security) 체크리스트

### 4.A 필수 — ✅ 적용 가능

| 체크 항목 | 판정 기준 | 적용 | 근거(파일:라인) | 심각도 | 미적용 조치 |
|-----------|-----------|------|------------------|--------|-------------|
| 입력 필터링 | `UserPromptSubmit`에서 악성 패턴 `block` | ☐ Y ☐ N | | High | →6.2 |
| 불신 데이터 표시 | 외부 본문을 신뢰/불신 컨텍스트로 분리·태그 | ☐ Y ☐ N | | High | →6.2 |
| DLP 필터 | 전송 입력에 주민·카드번호 정규식 탐지 시 `deny` | ☐ Y ☐ N | | Critical | →6.2 |
| 민감정보 마스킹 | `updatedInput`(전송 전)·`updatedToolOutput`(표시 전) 치환 | ☐ Y ☐ N | | Critical | →6.2 |
| 감사 로그 | `PostToolUse`로 모든 도구 호출 전수 기록 | ☐ Y ☐ N | | Critical | →6.2 |
| 최소 권한 | `tools`·`disallowedTools`로 필요 도구만 허용 | ☐ Y ☐ N | | Critical | →6.2 |
| 도구 화이트리스트 | 허용 외 도구·인자 `deny`(권한 모드 우회 불가) | ☐ Y ☐ N | | Critical | →6.2 |
| HITL 승인 | 고위험 작업에 `PreToolUse` `ask` 게이트 | ☐ Y ☐ N | | Critical | →6.2 |
| 시간·작업 기반 권한 | `date`·작업 분류 후 조건부 `deny`/`ask` | ☐ Y ☐ N | | Critical | →6.2 |

### 4.B 조건부 — △

| 체크 항목 | 판정 기준 | 적용 | 근거(파일:라인) | 심각도 | 한계(가능/불가) |
|-----------|-----------|------|------------------|--------|------------------|
| 역할 분리 | 신뢰/불신 컨텍스트 분리, 사실형 컨텍스트 주입 | ☐ Y ☐ N | | High | 가능: 분리·표시 / 불가: 주입 저항 강제(soft) |
| 네트워크 화이트리스트 | WebFetch·Bash(curl) URL 화이트리스트 외 `deny` | ☐ Y ☐ N | | Critical | 가능: 도구 deny / 불가: 패킷 레벨·Bash 난독화 차단 |

---

## 5. 평가 결과 기록부

② Y/N 판정 후 집계함. **근거 없는 Y는 0으로 계산**함.

| 영역 | 대상(✅+△) | 적용(Y) | 미적용(N) | 적용률 | 미적용 Critical/High |
|------|-----------|---------|-----------|--------|----------------------|
| 비용 | 12 | | | | |
| 성능 | 9 | | | | |
| 보안 | 11 | | | | |
| **합계** | **32** | | | | |

**미적용 항목 목록(심각도순)**: (기재 — Critical → High → Medium → Low)

**작성 예시(1행)**

| 체크 항목 | 적용 | 근거(파일:라인) | 심각도 | 비고 |
|-----------|------|------------------|--------|------|
| 감사 로그 | ✅ Y | `hooks/audit-log.sh:1`, `.claude-plugin/hooks/hooks.json:8` | Critical | PostToolUse matcher `*` 전수 기록 확인 |
| HITL 승인 | ❌ N | (없음) | Critical | 삭제 도구에 `ask` 게이트 부재 → 6.2 적용 |

---

## 6. 미적용 하네스 구현 방법

### 6.1 공통 복붙 스니펫

차단은 `exit 2` 또는 JSON `permissionDecision`/`decision`으로 함. `exit 1`은 비차단임([harness-hooks.md §8](harness-hooks.md)).

**A. PreToolUse `deny` — 차단·화이트리스트·카운터 게이트** (`harness-hooks.md` §3.1)

```bash
#!/bin/bash
INPUT=$(cat); CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
if echo "$CMD" | grep -qE 'rm -rf|curl .*(?!allowed\.com)'; then
  jq -n '{hookSpecificOutput:{hookEventName:"PreToolUse",
    permissionDecision:"deny", permissionDecisionReason:"hook 정책으로 차단됨"}}'
else exit 0; fi
```

**B. PostToolUse 감사 로그·사용량 적재** (`harness-hooks.md` §2.3, §3.6)

```bash
#!/bin/bash
INPUT=$(cat); LOG="${CLAUDE_PLUGIN_DATA:-.}/audit.log"
echo "$INPUT" | jq -c '{ts:.session_id, tool:.tool_name, input:.tool_input,
  usage:(.tool_response.usage // null), tokens:(.tool_response.totalTokens // null)}' >> "$LOG"
exit 0
```

**C. Stop 완료 게이트(무한 루프 방지 포함)** (`harness-hooks.md` §2.1)

```bash
#!/bin/bash
INPUT=$(cat)
[ "$(echo "$INPUT" | jq -r '.stop_hook_active')" = "true" ] && exit 0   # 이미 계속 진행 중이면 종료
# ... 완료 조건(산출물 존재·스키마 통과) 검사 후, 미충족 시에만 아래 ...
echo "완료 조건 미충족: 산출물 누락" >&2; exit 2
```

**D. PostToolUse `updatedToolOutput` — 표시 전 마스킹** (`harness-hooks.md` §3.2)

```json
{ "hookSpecificOutput": { "hookEventName": "PostToolUse",
  "updatedToolOutput": { "stdout": "[redacted]", "stderr": "", "interrupted": false, "isImage": false } } }
```

**E. PreToolUse `updatedInput` — 전송 전 절삭·치환** (`harness-hooks.md` §2.2): 입력 객체 **전체를 교체**하므로  
바꾸지 않을 필드도 함께 넣음.

### 6.2 항목별 레시피

| 항목 | 권장 이벤트 | 타입 | 핵심 조치 | 근거 절 | 참조 |
|------|-------------|------|-----------|---------|------|
| 최대 반복 횟수 | Stop/SubagentStop | command | `stop_hook_active` 확인 후 미완 시 block, 8회 cap 신뢰 | §2.1, §2.5 | C |
| 반복 패턴 감지 | PreToolUse | command | 최근 `tool_input` 해시를 상태파일과 비교, 반복 시 deny | §1.5, §2.5 | A |
| 부분 결과 반환 | Stop | command | 산출물 존재 미충족 시에만 block | §2.1 | C |
| 입력 절삭 | PreToolUse | command | `updatedInput`으로 과대 필드 절삭 | §2.2 | E |
| 핸드오프 요약 전달 | (Agent 설계) | — | 서브에이전트가 최종 요약 블록만 반환 | §2.3 | — |
| 요청별 사용량 측정 | PostToolUse(Agent) | command | `totalTokens`·`usage` 로그 적재 | §2.3 | B |
| 호출 수 제한 | PostToolUse+PreToolUse | command | 카운터 증가, 상한 초과 시 deny | §2.5, §3.1 | A,B |
| 이력 요약 | PreCompact | command | 압축 직전 핵심 이력 보존·아카이브 | §1.1 | — |
| Budget 한도 | PostToolUse(Agent)+PreToolUse | command | 누적 토큰 적재, 예산 초과 시 후속 deny | §2.3, §3.1 | A,B |
| Kill-switch | PreToolUse | command | 폭주 플래그 존재 시 고비용 도구 deny | §3.1 | A |
| 동시 실행 제한 | PreToolUse+PostToolUse | command | `flock` in-flight 카운터, ≥N이면 deny | §3.1, §8 | A |
| 깊이 제한 | PreToolUse(Agent) | command | 서브에이전트 `tools`에서 Agent 제거 또는 깊이 검사 후 deny | §3.1, §4.2 | A |
| Circuit Breaker | PostToolUseFailure+PreToolUse | command | 연속 실패 카운트, 임계 초과 시 일정시간 deny | §5(4행) | A |
| 병렬 실행 | (도구 설계) | — | 읽기전용 도구 `readOnlyHint`, 독립 작업 병렬 호출 | engine §4.2 | — |
| 스키마 검증 | PostToolUse/Stop | command | 출력 스키마 검증 실패 시 block | §5(6행) | C |
| 교차 검증 | (Agent 설계) | agent | 다중 검증자 비교 후 불일치 block | §5(6행) | — |
| 검증 실패 차단 | Stop/PostToolUse | command | 근거·테스트 검증 실패 시 다음 단계 block | §5(6행) | C |
| Graceful Degradation | PostToolUseFailure | command | `additionalContext`로 캐시·대체 응답 지침 주입 | §5(4행) | — |
| 리소스 격리 | (Agent 설계) | — | 위험 작업 서브에이전트 컨텍스트 격리 | §4.2 | — |
| 캐싱 | (MCP/스크립트) | mcp_tool | 조회 결과 캐시 저장·재사용 | §1.4 | — |
| Grounding | PostToolUse/Stop | command | 근거 존재 검증, 누락 시 block | §5(6행) | C |
| 입력 필터링 | UserPromptSubmit | command | 악성 패턴 매칭 시 decision block | §3.3, §3.6 | — |
| 불신 데이터 표시 | SubagentStart/(설계) | command | 외부 본문 불신 컨텍스트 분리·태그 | §3.3 | — |
| 역할 분리 | UserPromptSubmit/(설계) | command | 신뢰/불신 분리, 사실형 주입 | §3.3 | — |
| DLP 필터 | PreToolUse | command | 전송 입력 주민·카드번호 정규식, 탐지 시 deny | §3.2, §3.6 | A |
| 민감정보 마스킹 | PreToolUse/PostToolUse | command | `updatedInput`/`updatedToolOutput` 치환 | §3.2 | D,E |
| 감사 로그 | PostToolUse | command | 모든 도구 호출 tool·input·결과 적재 | §3.2, §3.6 | B |
| 최소 권한 | (설정) | — | `tools`·`disallowedTools`로 필요 도구만 | §3.6, §4.2 | — |
| 도구 화이트리스트 | PreToolUse/(설정) | command | 허용 외 도구·인자 deny | §3.1, §3.6 | A |
| HITL 승인 | PreToolUse | command | 고위험 작업 `permissionDecision: ask` | §3.1, §3.6 | A |
| 시간·작업 기반 권한 | PreToolUse | command | `date`·작업 분류 후 조건부 deny/ask | §3.1 | A |
| 네트워크 화이트리스트 | PreToolUse | command | URL 화이트리스트 외 deny(앱 레벨) | §3.1 | A |

**상세 예시(1건) — 감사 로그**  
- 이벤트·타입: `PostToolUse` · `command`(전수 기록은 결정적이므로 토큰 0의 command 사용)  
- matcher: `*`(전 도구) — 고위험 도구만 좁히려면 `Bash|Write|Edit` 등으로 한정  
- 핵심 조치: 모든 호출의 `tool_name`·`tool_input`·결과 사용량을 `${CLAUDE_PLUGIN_DATA}/audit.log`에 append  
- 근거: [harness-hooks.md §3.2, §3.6](harness-hooks.md) · 스니펫: 6.1-B

---

## 7. 구조적 불가(❌) 항목과 대안

아래는 **플러그인(hook 포함) 범위 밖**임. 체크리스트에 넣지 않으며, 필요 시 런타임/인프라로 해결함.

| 항목 | 리스크 | 불가 사유 | 대안(런타임/SDK/OS/인프라) |
|------|--------|-----------|-----------------------------|
| 단계별 타임아웃 | 무한 루프 | 생성·도구 wall-clock 타임아웃 hook 없음 | Agent SDK 타임아웃, 외부 워치독 프로세스 |
| 컨텍스트 하드 윈도 캡 | 토큰 누수 | 런타임 자동 압축 전속 | SDK 컨텍스트 설정, 더 큰/작은 모델 선택 |
| 메인 에이전트 실시간 토큰 측정 | 토큰 누수 | hook에 메인 대화 토큰 실시간 미제공 | SDK usage 스트림, 게이트웨이 프록시 계측 |
| Backpressure | 시스템 마비 | 인바운드 큐·부하 차단은 세션 밖 | API 게이트웨이·로드밸런서·요청 큐 |
| 계층적 타임아웃 | 응답 지연 | 에이전트·다계층 wall-clock 타임아웃 없음 | SDK `maxTurns`+외부 타임아웃, 오케스트레이션 계층 |
| 스트리밍 응답 | 응답 지연 | 전송 계층, 서브에이전트 핸드오프는 단일 블록 | SDK `include_partial_messages`(메인), 클라이언트 |
| OS 샌드박스 | Prompt Injection | `deny`는 접근 차단일 뿐 프로세스·리소스 격리가 아님 | 컨테이너·격리 VM 배포, 코드 실행 도구 화이트리스트 축소 |

> 주의: `updatedToolOutput`은 Claude가 보는 내용만 바꿈 — **실제 전송 차단은 `PreToolUse`로 사전 처리**함.  
> `if` 필터는 best-effort(fail-open) — **하드 강제는 permission system 병행**함([harness-hooks.md §1.3, §3.2](harness-hooks.md)).

---

## 8. 적용 로드맵

[harness-engineering-guide.md §1·§8](harness-engineering-guide.md) 원칙에 따라 **비용 → 성능 → 보안** 순,  
심각도 **Critical → High → Medium → Low** 우선으로 적용함.

| 단계 | 영역 | 우선 착수(심각도) | 대표 항목 |
|------|------|-------------------|-----------|
| 1 | 비용(즉효·단순) | Critical: Agent 폭주 | 호출 수 제한, Budget·Kill-switch, 최대 반복(Stop 게이트), 요청별 사용량 측정 |
| 2 | 성능 | High: 시스템 마비 | Circuit Breaker, 검증 실패 차단, 병렬화·캐싱 |
| 3 | 보안(구조 변경) | Critical: 데이터 유출·권한 오남용 | 감사 로그, DLP·마스킹, 최소 권한·도구 화이트리스트, HITL, 네트워크 화이트리스트 |

**배포 게이트**: [harness-engineering-guide.md §7](harness-engineering-guide.md) 체크리스트와 본 검사기에서  
**Critical·High 항목이 전부 Y**가 되기 전에는 배포하지 않음. 미적용은 [§8 심각도 SLA](harness-engineering-guide.md)에 따라 보완함.
