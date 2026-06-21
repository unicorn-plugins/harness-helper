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

## 0-3. harness.yaml 설정 파일

훅 스크립트에 임계값을 하드코딩하지 않고 단일 설정 파일에서 관리함.  
훅 스크립트는 `yq` 또는 `jq`로 이 파일을 읽어 한도를 참조함.

**파일 위치**: 플러그인 루트 (`harness.yaml`)

```yaml
# harness.yaml — 비용 하네스 설정 (성능·보안 섹션은 추후 추가)

budget:
  max_tokens: 100000          # 세션 전체 토큰 한도
  max_tokens_per_call: 10000  # 호출당 최대 토큰
  alert_at: 0.8               # 80% 도달 시 경고 로그
  state_file: ".harness/budget.json"   # PostToolUse 누적 적재 파일

loops:
  targets:                    # 카운트할 도구 (Bash·Read 등 기본 도구는 제외)
    - "Agent"
    - "WebSearch"
    - "WebFetch"
  max_iterations: 10          # targets 도구의 최대 호출 횟수
  state_file: ".harness/loop.json"

agents:
  max_calls: 20               # Agent 도구 최대 호출 수
  state_file: ".harness/agents.json"

# 성능 하네스 설정
performance:
  circuit_breaker:
    targets:                  # 감시할 도구 (외부 호출 도구만 지정)
      - "WebFetch"
      - "WebSearch"
    failure_threshold: 3      # 연속 실패 횟수 임계값 (초과 시 open)
    recovery_seconds: 60      # open → half-open 전환 대기 시간(초)
                              # (bash: date +%s 로 경과 시간 비교, native 아님)
    half_open: true           # half-open 활성화 시 프로브 1회 허용 후 결과로 분기
                              #   성공 → closed (failures 리셋)
                              #   실패 → open (opened_at 리셋)
    fallback:
      strategy: "cache"       # cache | guide | tool | escalate
                              #   cache: 마지막 성공 결과를 additionalContext로 전달
                              #   guide: Claude에게 대체 방법 지시 (soft)
                              #   tool: updatedInput으로 폴백 도구로 교체
                              #   escalate: decision:ask로 사용자에게 판단 위임
      cache_file: ".harness/cb-cache.json"   # strategy=cache 일 때 캐시 저장 파일
      guide_message: "외부 서비스 차단 중. 이전 결과 또는 학습 지식으로 답변하세요."
    state_file: ".harness/circuit.json"
  cache:
    targets:                  # 캐시 적용 도구 (외부 조회 도구만 지정)
      - "WebFetch"
      - "WebSearch"
    ttl: 3600                 # 캐시 유효 시간(초), 초과 시 미스로 처리
    cache_file: ".harness/cache.json"
    # cache.json 구조:
    # { "<input_hash>": { "result": "...", "cached_at": <unix_ts> } }
    # circuit.json 구조:
    # { "state": "closed",      ← closed | open | half-open
    #   "failures": 0,          ← 연속 실패 카운트
    #   "opened_at": null,      ← open 전환 시각 (Unix timestamp)
    #   "probe_sent": false }   ← half-open 프로브 발송 여부
```

**훅에서 읽는 예시**

```bash
MAX=$(yq '.budget.max_tokens' harness.yaml)
CURRENT=$(jq '.total // 0' "$(yq '.budget.state_file' harness.yaml)")
[ "$CURRENT" -gt "$MAX" ] && echo "한도 초과" && exit 2
```

> 앞으로 성능(circuit_breaker·타임아웃)·보안(dlp·허용 도메인) 섹션을 이 파일에 추가함.

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
| 최대 반복 횟수 | `PreToolUse` 카운터로 한도 초과 시 deny, 또는 서브에이전트 `maxTurns` 지정, 또는 `Stop` hook 완료 게이트 존재 | ☐ Y ☐ N | | High | →6.2 |
| 반복 패턴 감지 | 동일 `tool_input` 반복을 상태파일로 비교·차단하는 hook 존재 | ☐ Y ☐ N | | High | →6.2 |
| 입력 절삭 | `PreToolUse updatedInput`으로 과대 입력을 절삭함 | ☐ Y ☐ N | | Medium | →6.2 |
| 요청별 사용량 측정 | `PostToolUse(Agent)`로 `totalTokens`·`usage`를 기록함 | ☐ Y ☐ N | | Medium | →6.2 |
| 호출 수 제한 | 고비용 도구 호출 카운터+상한 초과 시 `deny` hook 존재 | ☐ Y ☐ N | | Critical | →6.2 |
| Budget 한도 | `PostToolUse`에서 누적 토큰을 상태파일에 적재, 다음 `PreToolUse`에서 한도 비교 후 초과 시 Agent 도구 `deny` | ☐ Y ☐ N | | Critical | →6.2 |

### 2.B 조건부 — △ (한계 인지 후 적용)

| 체크 항목 | 판정 기준 | 적용 | 근거(파일:라인) | 심각도 | 한계(가능/불가) |
|-----------|-----------|------|------------------|--------|------------------|
| 핸드오프 요약 전달 | 서브에이전트 시스템 프롬프트에 "최종 요약만 반환" 지시 | ☐ Y ☐ N | | Medium | 가능: 요약 형식 지시 / 불가: 반환 내용 강제(soft) |
| 부분 결과 반환 | 한도 초과 시 `PreToolUse` deny + `additionalContext`로 부분 결과 반환 안내 포함 | ☐ Y ☐ N | | High | 가능: 루프 강제 종료(deny) / 불가: Claude 실제 반환 내용 강제(soft) |

> **비용 설계 포인트 — 스킬·에이전트 지침** (훅 강제가 어려운 경우 soft 보완)
>
> ① 무한 루프 방지
> - 동일 작업은 최대 N회 시도 후 현재까지 결과를 반환하세요
> - 이전과 동일한 검색어·쿼리를 반복하지 마세요
> - 매 반복 후 목표 달성 여부를 자체 평가하고, 달성 시 즉시 종료하세요
>
> ② 토큰 누수 방지
> - 서브에이전트에는 전체 이력 대신 핵심 요약(5줄 이내)만 전달하세요
> - 도구 출력이 길면 핵심 내용만 추출하여 컨텍스트에 포함하세요
> - 최종 결과는 핵심만 반환하세요 (중간 과정 포함 금지)
>
> ③ Agent 폭주 방지
> - 서브에이전트는 최대 N개까지만 호출하세요
> - 직접 처리 가능한 작업은 서브에이전트 없이 수행하세요
> - 총 호출이 N에 도달하면 현재 결과로 마무리하세요

---

## 3. 성능(Performance) 체크리스트

### 3.A 필수 — ✅ 적용 가능

| 체크 항목 | 판정 기준 | 적용 | 근거(파일:라인) | 심각도 | 미적용 조치 |
|-----------|-----------|------|------------------|--------|-------------|
| Circuit Breaker | `PostToolUseFailure` 실패 카운트+임계 초과 시 `deny`, deny 시 `additionalContext`로 폴백 응답 지침 주입(Graceful Degradation 포함) | ☐ Y ☐ N | | High | →6.2 |
| 병렬 실행 | 서브에이전트를 호출하는 스킬 지침에 독립 작업은 병렬 실행하도록 명시 | ☐ Y ☐ N | | Low | →6.2 |
| 스키마 검증 | `PostToolUse`에서 코드로 결과 형식·길이·필수 필드·패턴 검증 후 실패 시 `block` | ☐ Y ☐ N | | Medium | →6.2 |
| 품질 게이트 | 스킬 작업 파이프라인에 품질 체크 에이전트 단계 추가, 미달 시 재생성·중단 | ☐ Y ☐ N | | Medium | →6.2 |
| 교차 검증 | 복수 모델로 결과 교차 확인: 동일 모델 독립 2개 / 티어 조합(Sonnet+Opus) / 외부 모델(CLI·MCP로 타 모델 호출) | ☐ Y ☐ N | | Medium | →6.2 |
| 캐싱 | `PreToolUse`에서 `tool_input` 해시로 cache.json 조회, 히트 시 `deny`+캐시 결과 `additionalContext` 반환; `PostToolUse`에서 결과 저장(TTL 검사 포함) | ☐ Y ☐ N | | Low | →6.2 |

> **품질 검증 기준 예시** (스키마 검증·품질 게이트 공통 체크리스트)  
> - 형식·필수 필드 존재 여부 (JSON 파싱, 키 존재 확인)  
> - 근거·출처 포함 여부 (Grounding — 인용·출처 URL 존재)  
> - 기대 길이·범위 충족 여부  
> - 금지 패턴 없음 (PII, 부적절 표현 등)  
> - (교차 검증) 복수 모델 결과 일치 여부 — Sonnet 생성 + Opus 검증 등

> **외부 도구 호출 설계 포인트** (웹 검색·MCP 등 외부 도구 호출 시 적용 검토)  
> - 연속 실패 위험 → Circuit Breaker + fallback 처리 (cache·guide·tool·escalate 중 선택)  
> - 동일 쿼리 반복 가능성 → 캐싱 적용 (TTL 기반, 비용·속도 절감)  
> - 결과가 다음 단계 입력인 경우 → 결과 품질 게이트 필수 (형식·필수 필드·근거 확인 후 block)

### 3.B 조건부 — △

| 체크 항목 | 판정 기준 | 적용 | 근거(파일:라인) | 심각도 | 한계(가능/불가) |
|-----------|-----------|------|------------------|--------|------------------|

> **성능 설계 포인트 — 스킬·에이전트 지침** (훅 강제가 어려운 경우 soft 보완)
>
> ① 장애 전파 방지
> - 외부 도구 호출 실패 시 즉시 전체 중단하지 말고 대체 방법을 먼저 시도하세요
> - 한 단계 실패가 다음 단계에 영향을 주지 않도록 각 단계를 독립적으로 처리하세요
> - 연속 실패 발생 시 현재까지 성공한 결과를 반환하고 실패 원인을 명시하세요
>
> ② 품질 저하 방지
> - 도구 실행 결과가 예상 형식·필수 필드를 충족하지 않으면 다음 단계로 전달하지 마세요
> - 결과에 근거·출처가 없으면 반환하지 말고 재시도하되 최대 N회로 제한하세요
> - 검증 실패 내용을 다음 단계에 반드시 포함하여 오류가 묻히지 않도록 하세요
>
> ③ 지연·낭비 방지
> - 서로 의존성이 없는 작업은 병렬로 실행하세요
> - 이전 단계에서 조회한 정보를 다시 조회하지 마세요 (중복 조회 금지)
> - 장시간 작업의 중간 결과는 메모에 저장하고 이후 단계에서 재사용하세요

---

## 4. 보안(Security) 체크리스트

### 4.A 필수 — ✅ 적용 가능

| 체크 항목 | 판정 기준 | 적용 | 근거(파일:라인) | 심각도 | 미적용 조치 |
|-----------|-----------|------|------------------|--------|-------------|
| 입력 필터링 | `UserPromptSubmit`에서 악성 패턴 `block` | ☐ Y ☐ N | | High | →6.2 |
| DLP 필터 | 전송 입력에 주민·카드번호 정규식 탐지 시 `deny` | ☐ Y ☐ N | | Critical | →6.2 |
| 민감정보 마스킹 | `updatedInput`(전송 전)·`updatedToolOutput`(표시 전) 치환 | ☐ Y ☐ N | | Critical | →6.2 |
| 감사 로그 | **Managed Hook 권장** (개별 플러그인이 아닌 조직 정책으로 강제): UserPromptSubmit(프롬프트 원문)·Stop(최종 응답) 필수; PostToolUse는 민감 도구(Bash·WebFetch·파일쓰기)·실패 건만 선택 기록 | ☐ Y ☐ N | | Critical | →6.2 |
| 최소 권한 | `tools`·`disallowedTools`로 필요 도구만 허용 | ☐ Y ☐ N | | Critical | →6.2 |
| 도구 화이트리스트 | 레벨별 적용: ① 에이전트 frontmatter `allowed-tools`/`disallowed-tools`(에이전트 단위) ② `settings.json` `disallowedTools`(세션·프로젝트) ③ `managed-settings.json`(조직 강제) ④ `PreToolUse` `deny`(런타임 조건부); 스킬 자체 tool 통제 없음 — 세션 설정 또는 호출 에이전트 frontmatter 적용 | ☐ Y ☐ N | | Critical | →6.2 |
| HITL 승인 | 고위험 작업에 `PreToolUse` `decision: ask` 게이트; 시간·작업 기반 권한이 트리거 조건을 동적으로 결정 | ☐ Y ☐ N | | Critical | →6.2 |
| 시간·작업 기반 권한 | PreToolUse에서 시간(`date`)·작업 종류 분류 후 조건부 `deny`/`ask` 분기; 읽기→allow, 쓰기→ask, 삭제·실행→deny 또는 ask, 업무시간 외 고위험→deny; HITL의 트리거 조건 결정 역할 | ☐ Y ☐ N | | Critical | →6.2 |

### 4.B 조건부 — △

| 체크 항목 | 판정 기준 | 적용 | 근거(파일:라인) | 심각도 | 한계(가능/불가) |
|-----------|-----------|------|------------------|--------|------------------|
| 불신 데이터 표시 | PostToolUse `updatedToolOutput`으로 외부 콘텐츠를 `<untrusted_content>` 태그로 감쌈; 처리 규칙은 CLAUDE.md·SubagentStart `additionalContext`로 주입 | ☐ Y ☐ N | | High | 가능: 태그 삽입·지침 주입 / 불가: Claude의 태그 준수 강제(CLAUDE.md 희석 시 soft) |
| 역할 분리 | 신뢰/불신 컨텍스트 분리, 사실형 컨텍스트 주입 | ☐ Y ☐ N | | High | 가능: 분리·표시 / 불가: 주입 저항 강제(soft) |
| 네트워크 화이트리스트 | WebFetch·Bash(curl) URL 화이트리스트 외 `deny` | ☐ Y ☐ N | | Critical | 가능: 도구 deny / 불가: 패킷 레벨·Bash 난독화 차단 |

---

## 5. 평가 결과 기록부

② Y/N 판정 후 집계함. **근거 없는 Y는 0으로 계산**함.

| 영역 | 대상(✅+△) | 적용(Y) | 미적용(N) | 적용률 | 미적용 Critical/High |
|------|-----------|---------|-----------|--------|----------------------|
| 비용 | 8 | | | | |
| 성능 | 6 | | | | |
| 보안 | 11 | | | | |
| **합계** | **25** | | | | |

**미적용 항목 정리** — 표가 아닌 **문단(계층)** 형식으로 기재함.

- **계층**: ① 영역(비용·성능·보안) → ② 관련 리스크(`리스크명(약어)` — 돌·토·폭/멈·느·할/침·유·권) →  
  ③ 미적용 항목(`{ID}. [등급] 항목명` + 현재 문제 1~2줄 + 개선 방안 1~2줄)
- **항목 ID**: 영역 접두사(비용 `C`·성능 `P`·보안 `S`) + 영역별 일련번호(C1·C2·P1·S1…). **한 번 부여한 ID는 고정·영구 보존**.  
  구현 완료 시 항목을 삭제하지 않고 `~~취소선~~ [적용됨]`으로 잔류시켜 재점검·보완(apply) 시 번호가 어긋나지 않게 함.  
  **취소선 항목의 ID는 재사용하지 않음** — 신규 미적용 항목은 해당 영역의 최대 일련번호+1부터 증분(중간 간격 허용)
- **정렬**: 영역(비용→성능→보안) → 리스크(약어 표준순) → 항목(등급 Critical→High→Medium→Low → 항목명순 →  
  완전 동일 시 2~4장 체크리스트 등장순). **취소선(`[적용됨]`) 항목은 정렬·증분 채번 모집단에서 제외**.  
  최초 채번 시 ID는 이 정렬 순서대로 영역별 1부터 부여

**작성 예시**

```markdown
## 비용 (Cost)

### Agent 폭주(폭)
**C1. [Critical] 호출 수 제한**
- 현재 문제: 고비용 도구 호출 카운터·상한이 없어 폭주 시 비용이 급증함
- 개선 방안: PreToolUse에 호출 카운터+상한 추가, 초과 시 `deny` (→6.2)

## 보안 (Security)

### 권한 오남용(권)
**S1. [Critical] HITL 승인 게이트**
- 현재 문제: 삭제 도구에 `ask` 게이트가 없어 고위험 작업이 무승인 실행됨
- 개선 방안: PreToolUse `decision: ask` 게이트 추가 (→6.2)

~~**S2. [High] 감사 로그**~~ **[적용됨]** — `hooks/audit-log.mjs:1` (2026-06-21 보완)
```

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
| 최대 반복 횟수 | PreToolUse 또는 Stop/SubagentStop | command | `PreToolUse` 카운터 한도 초과 시 deny(권장), 또는 `Stop` 완료 게이트 block | §2.1, §2.5, §3.1 | A,C |
| 반복 패턴 감지 | PreToolUse | command | 최근 `tool_input` 해시를 상태파일과 비교, 반복 시 deny | §1.5, §2.5 | A |
| 부분 결과 반환 | PreToolUse | command | 한도 초과 시 deny + `additionalContext`로 부분 결과 반환 안내(Claude soft) | §3.1 | A |
| 입력 절삭 | PreToolUse | command | `updatedInput`으로 과대 필드 절삭 | §2.2 | E |
| 핸드오프 요약 전달 | (Agent 설계) | — | 서브에이전트 시스템 프롬프트에 요약 형식 지시(soft) | §2.3 | — |
| 요청별 사용량 측정 | PostToolUse(Agent) | command | `totalTokens`·`usage` 로그 적재 | §2.3 | B |
| 호출 수 제한 | PreToolUse | command | Agent 도구 호출 직전 카운터 증가·한도 비교, 초과 시 deny (matcher: Agent) | §3.1 | A |
| Budget 한도 | PostToolUse+PreToolUse | command | `PostToolUse`에서 `totalTokens` 읽어 상태파일 누적 → 다음 `PreToolUse`에서 한도 비교·초과 시 deny | §2.3, §3.1 | A,B |
| Circuit Breaker | PostToolUseFailure+PreToolUse | command | 연속 실패 카운트, 임계 초과 시 deny + `additionalContext`로 폴백 응답 지침 주입 | §5(4행) | A |
| 병렬 실행 | (스킬 지침) | — | 서브에이전트를 호출하는 스킬 지침에 독립 작업은 병렬 실행하도록 명시 | engine §4.2 | — |
| 스키마 검증 | PostToolUse | command | 결과 형식·길이·필수 필드·패턴을 코드로 검증, 실패 시 block | §5(6행) | C |
| 품질 게이트 | (스킬 에이전트) | — | 스킬 작업 단계에 품질 체크 에이전트 추가, 의미·완성도 미달 시 재생성·중단(△) | §5(6행) | — |
| 교차 검증 | (스킬 에이전트) | — | 복수 모델로 교차 확인: 동일 모델 독립 2개 / 티어 조합(Sonnet+Opus) / 외부 모델(CLI·MCP) | §5(6행) | — |
| 캐싱 | PreToolUse+PostToolUse | command | PreToolUse에서 입력 해시로 캐시 히트 시 deny+결과 반환, PostToolUse에서 결과 저장(TTL 검사) | §3.1 | A,B |
| 입력 필터링 | UserPromptSubmit | command | 악성 패턴 매칭 시 decision block | §3.3, §3.6 | — |
| 불신 데이터 표시 | PostToolUse + SubagentStart | command | PostToolUse `updatedToolOutput`으로 외부 콘텐츠 `<untrusted_content>` 태그 감쌈; SubagentStart `additionalContext`로 처리 규칙 재주입; CLAUDE.md에 태그 처리 규칙 필수 명시 | §3.3 | D |
| 역할 분리 | UserPromptSubmit/(설계) | command | 신뢰/불신 분리, 사실형 주입 | §3.3 | — |
| DLP 필터 | PreToolUse | command | 전송 입력 주민·카드번호 정규식, 탐지 시 deny | §3.2, §3.6 | A |
| 민감정보 마스킹 | PreToolUse/PostToolUse | command | `updatedInput`/`updatedToolOutput` 치환 | §3.2 | D,E |
| 감사 로그 | UserPromptSubmit + Stop + PostToolUse(선택) | command | **managed-settings.json 적용 권장** — UserPromptSubmit(프롬프트 원문)·Stop(최종 응답) 필수 기록; PostToolUse는 민감 도구·실패 건만 선택; 개별 플러그인 settings.json으로 비활성화 불가 | §3.2, §3.6 | B |
| 최소 권한 | (설정) | — | `tools`·`disallowedTools`로 필요 도구만 | §3.6, §4.2 | — |
| 도구 화이트리스트 | frontmatter + settings.json + PreToolUse | command | ① 에이전트 frontmatter `allowed-tools`/`disallowed-tools` ② `settings.json` `disallowedTools` ③ `managed-settings.json`(조직 강제) ④ PreToolUse `deny`(런타임 조건부); 스킬은 자체 통제 없음 | §3.1, §3.6 | A |
| HITL 승인 | PreToolUse | command | 고위험 작업 `decision: ask`; 시간·작업 기반 권한이 트리거 조건 결정 | §3.1, §3.6 | A |
| 시간·작업 기반 권한 | PreToolUse | command | 시간(`date`)·작업 종류 분류 후 deny/ask 분기: 읽기→allow, 쓰기→ask, 삭제·실행→deny 또는 ask, 업무시간 외 고위험→deny | §3.1 | A |
| 네트워크 화이트리스트 | PreToolUse | command | URL 화이트리스트 외 deny(앱 레벨) | §3.1 | A |

**Circuit Breaker fallback 처리 유형**

| strategy | 처리 방식 | 강제 여부 | 구현 위치 |
|----------|-----------|----------|-----------|
| `cache` | 마지막 성공 결과를 `additionalContext`로 전달 | ✅ | PostToolUse(캐시 저장) + PreToolUse(캐시 반환) |
| `guide` | Claude에게 대체 방법 지시 (`additionalContext`) | △ soft | PreToolUse deny 시 메시지 주입 |
| `tool` | `updatedInput`으로 폴백 도구로 교체 | ✅ | PreToolUse |
| `escalate` | `decision:ask`로 사용자에게 판단 위임 | ✅ | PreToolUse |

> `cache` strategy 사용 시 PostToolUse(성공)에서 결과를 `cb-cache.json`에 저장해야 함.  
> 주기적 백그라운드 헬스체크(고전적 half-open 프로브)는 훅 범위 밖 → 외부 크론·워치독 프로세스 필요.

---

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
| 1 | 비용(즉효·단순) | Critical: Agent 폭주 | 호출 수 제한, Budget 한도, 최대 반복(PreToolUse 카운터), 요청별 사용량 측정 |
| 2 | 성능 | High: 시스템 마비 | Circuit Breaker, 스키마 검증·품질 게이트·교차 검증, 병렬화·캐싱 |
| 3 | 보안(구조 변경) | Critical: 데이터 유출·권한 오남용 | 감사 로그, DLP·마스킹, 최소 권한·도구 화이트리스트, HITL, 네트워크 화이트리스트 |

**배포 게이트**: [harness-engineering-guide.md §7](harness-engineering-guide.md) 체크리스트와 본 검사기에서  
**Critical·High 항목이 전부 Y**가 되기 전에는 배포하지 않음. 미적용은 [§8 심각도 SLA](harness-engineering-guide.md)에 따라 보완함.
