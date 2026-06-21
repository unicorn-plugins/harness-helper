# 하네스 엔지니어링 Hooks 적용 가이드 (harness-hooks)

> 작성일: 2026-06-18  
> 대상: Skill · Agent · 플러그인 개발자  
> 목적: 하네스 엔지니어링의 "구조적 통제"를 Claude Code **Hooks**로 강제하는 방법 정리. 특히 **비용·보안** 측면 활용법 제시  
> 관계: [하네스 엔지니어링 적용 가이드](harness-engineering-guide.md)의 9대 리스크를 hooks로 구현하는 **실행편**  
> 출처: Claude 공식 문서(docs.claude.com). 본문 모든 Fact는 9절 출처 링크에 근거함

---

## 0. 개요 — Hooks가 하네스에서 하는 역할

하네스 엔지니어링의 핵심은 품질·비용·안전을 **프롬프트(부탁)** 가 아니라 **구조(강제)** 로 달성하는 것임.  
Hooks는 그 "강제"를 담당하는 결정적 코드 게이트임.

- **정의**: Hooks는 Claude Code 생명주기의 특정 지점에서 **자동 실행**되는 사용자 정의 셸 명령·HTTP 엔드포인트·  
  LLM 프롬프트·서브에이전트임. 모델의 판단과 무관하게 항상 실행됨
- **프롬프트와의 결정적 차이**: `PreToolUse` hook이 `deny`를 반환하면 `bypassPermissions` 모드나  
  `--dangerously-skip-permissions`에서도 도구 호출이 차단됨. 즉 사용자가 권한 모드를 바꿔도 우회 불가함
- **단, 단방향 강화만 가능**: hook의 `allow`는 settings의 deny 규칙을 무력화하지 못함.  
  hook은 제약을 **조일 수는 있어도 풀 수는 없음**

| 구분 | 프롬프트/CLAUDE.md | Hooks |
|------|--------------------|-------|
| 실행 보장 | 모델이 따를 수도, 안 따를 수도 있음(확률적) | 조건 충족 시 항상 실행(결정적) |
| 차단 능력 | 없음(권고) | `exit 2` 또는 JSON `deny`로 실제 차단 |
| 비용 성격 | 컨텍스트 토큰 차지 | command/http/mcp는 Claude 토큰 비소비, prompt/agent는 소비 |
| 적합 용도 | 정적 규칙·스타일 | 측정 가능한 한도·권한·전송 게이트 |

---

## 1. Hooks 기본 개념 (Fact)

### 1.1 생명주기와 주요 이벤트

이벤트는 실행 빈도(cadence)에 따라 세 종류로 나뉨. **빈도가 비용·지연에 직결**되므로 반드시 인지함.

| Cadence | 이벤트(예) | 발생 시점 |
|---------|-----------|-----------|
| 세션당 1회 | `SessionStart`, `SessionEnd`, `Setup` | 세션 시작·종료, 명시적 초기화 |
| 턴당 1회 | `UserPromptSubmit`, `Stop`, `StopFailure`, `PostToolBatch` | 프롬프트 제출·턴 종료·배치 종료 |
| 도구 호출마다 | `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest` | 에이전트 루프 내 매 도구 호출 |

그 외 `SubagentStart`/`SubagentStop`(서브에이전트), `PreCompact`/`PostCompact`(컨텍스트 압축),  
`TaskCreated`/`TaskCompleted`, `FileChanged`, `CwdChanged`, `Notification`, `InstructionsLoaded` 등이 있음.

> **비용 관점 핵심**: `PreToolUse`/`PostToolUse`는 **도구를 호출할 때마다** 발생함. 여기에 무거운 hook을 달면  
> 호출 수에 비례해 비용·지연이 누적됨. `Stop`/`SubagentStop`은 턴당 1회라 상대적으로 안전함.

### 1.2 설정 위치와 스코프

hook을 어디에 정의하느냐가 적용 범위를 결정함.

| 위치 | 스코프 | 공유 |
|------|--------|------|
| `~/.claude/settings.json` | 내 모든 프로젝트 | 안 됨(로컬 머신) |
| `.claude/settings.json` | 단일 프로젝트 | 됨(repo 커밋 가능) |
| `.claude/settings.local.json` | 단일 프로젝트 | 안 됨(gitignore됨) |
| 관리형 정책 설정(managed) | 조직 전체 | 됨(관리자 통제) |
| 플러그인 `hooks/hooks.json` | 플러그인 활성화 시 | 됨(플러그인에 번들) |
| Skill·Agent frontmatter | 해당 컴포넌트 활성 동안 | 됨(컴포넌트 파일에 정의) |

### 1.3 hook 해석 흐름: event → matcher → if → handler

설정은 3단 중첩 구조임.

1. **이벤트** 선택(`PreToolUse` 등)
2. **matcher 그룹**으로 발생 조건 필터(예: `"Bash"`, `"Edit|Write"`, 정규식 `"mcp__memory__.*"`)
3. **handler** 정의(실행할 명령/프롬프트/URL 등)

추가로 도구 이벤트에는 핸들러별 **`if` 필드**로 더 좁힐 수 있음(권한 규칙 문법, 예: `"Bash(rm *)"`, `"Edit(*.ts)"`).  
matcher와 `if`가 좁을수록 불필요한 프로세스 spawn이 줄어 **비용·지연이 감소**함.

> **주의**: `if` 필터는 best-effort임. Bash 명령 파싱이 안 되면 fail-open(그냥 실행)함.  
> **하드 차단은 hook의 `if`가 아니라 [permission system](https://docs.claude.com/en/docs/claude-code/iam)으로 강제**해야 함.

### 1.4 5가지 handler 타입 — 비용·차단능력 비교

| 타입 | 동작 | Claude 토큰 비용 | 차단 가능 | 권장 용도 |
|------|------|------------------|-----------|-----------|
| `command` | 로컬 셸 명령 실행(stdin JSON) | **없음**(로컬 실행) | 됨 | 결정적 게이트·검증·로깅(기본 선택) |
| `http` | 외부 URL로 POST | 없음(외부 서비스 비용 별도) | 됨 | 사내 검증 서비스 연동 |
| `mcp_tool` | 연결된 MCP 서버 도구 호출 | 없음(MCP 서버 비용 별도) | 됨 | MCP 기반 스캔·검증 |
| `prompt` | Claude 모델 1회 호출(기본 Haiku) | **있음** | 됨 | 단순 LLM 판단이 필요한 게이트 |
| `agent` | 서브에이전트 spawn(최대 50턴, 도구 사용) | **있음(가장 큼)** | 됨 | 파일·테스트 검사가 필요한 검증(실험적) |

> **비용 1원칙**: 결정적으로 판정 가능한 것은 `command`로 처리함. `prompt`/`agent`는 LLM을 호출하므로  
> 토큰 비용이 발생함. 특히 `agent`는 최대 50턴까지 도는 별도 에이전트라 **가장 비쌈**(실험적 기능, 프로덕션은 command 권장).

### 1.5 입력·출력과 차단 신호

- **입력**: command hook은 이벤트 JSON을 **stdin**으로 받음(http는 POST 본문). 공통 필드로  
  `session_id`, `cwd`, `permission_mode`, `tool_name`, `tool_input` 등이 옴
- **Exit code 신호** (command hook)
  - `exit 0`: 성공. stdout의 JSON을 파싱해 처리
  - `exit 2`: **차단 에러**. stdout/JSON은 무시되고 **stderr가 Claude에 피드백**됨. 효과는 이벤트별로 다름  
    (`PreToolUse`=도구 차단, `UserPromptSubmit`=프롬프트 거부, `Stop`=종료 방지 등)
  - 그 외 코드: **비차단 에러**(작업은 계속됨). 단 `WorktreeCreate`는 예외로 0이 아니면 생성 중단
  - 함정: `exit 1`은 비차단임. 정책 강제는 반드시 **`exit 2`** 를 사용함
- **JSON 출력(decision control)**: `exit 0` + stdout JSON으로 정교하게 제어함
  - 공통: `continue:false`(전체 중단)+`stopReason`, `suppressOutput`, `systemMessage`
  - `PreToolUse`: `hookSpecificOutput.permissionDecision`(allow/deny/ask/defer)
  - `PostToolUse`/`UserPromptSubmit`/`Stop` 등: 최상위 `decision:"block"`+`reason`
  - 컨텍스트 주입: `additionalContext`(**문자열 상한 10,000자**, 초과 시 파일로 대체)

---

## 2. 비용(Cost) 측면 Hooks 활용

> **핵심 관점**: 비용은 사후 정산이 아니라 **실행 중 통제**의 문제임.  
> Hooks로 ① 무한 루프 차단 ② 토큰 누수 절삭 ③ Agent 폭주·고비용 도구 게이트 ④ hook 자체 비용 최소화를 수행함.

### 2.1 무한 루프 차단 — `Stop` hook과 block cap

`Stop` hook이 `decision:"block"`을 반환하면 Claude가 종료하지 않고 계속 작업함. 이를 완료 강제 게이트로 쓸 수 있으나,  
**무한 반복 위험**이 있음. 안전장치는 다음과 같음.

- Claude Code는 `Stop` hook이 **연속 8회** block하면 자동으로 무시하고 턴을 종료함(폭주 backstop)
- hook은 입력의 `stop_hook_active` 필드를 확인해 **이미 계속 진행 중이면 조기 종료**해야 함

```bash
#!/bin/bash
INPUT=$(cat)
# 이미 stop hook이 한 번 계속을 유발했다면 재차단하지 않음 (무한 루프 방지)
if [ "$(echo "$INPUT" | jq -r '.stop_hook_active')" = "true" ]; then
  exit 0
fi
# ... 완료 조건 검사 후, 미충족 시에만 exit 2 또는 decision:block ...
```

- **안티패턴 → 권장패턴**
  - ❌ `stop_hook_active` 무시하고 조건 미충족 시 무조건 block → ✅ 활성 플래그 확인 후 조기 종료
  - ❌ LLM 판단(`prompt` hook)을 매 `Stop`마다 호출 → ✅ 결정적 검사는 `command`로, 8회 cap 신뢰

### 2.2 토큰 누수 차단 — 입력 절삭과 컨텍스트 상한

- **PreToolUse `updatedInput`**: 도구 입력을 실행 전 **치환**함. 과도한 입력을 잘라 토큰을 절감할 수 있음  
  (입력 객체 **전체를 교체**하므로 바꾸지 않을 필드도 함께 넣어야 함)
- **`additionalContext` 상한 활용**: hook이 주입하는 컨텍스트는 10,000자로 제한됨. 큰 데이터는 요약·발췌만 주입함
- **정적 규칙은 hook 대신 CLAUDE.md**: 변하지 않는 지시는 스크립트 실행 없이 로드되는 CLAUDE.md에 둠  
  (hook의 `additionalContext`는 매번 실행·주입되어 토큰·지연을 유발)

### 2.3 Agent 폭주·고비용 도구 게이트 + 비용 측정

- **고비용 도구 사전 차단**: `PreToolUse` + `if`로 특정 도구·인자를 `deny`/`ask` 처리함
- **서브에이전트 비용 계측**: `Agent` 도구가 완료되면 `PostToolUse`의 `tool_response`에 **사용량 텔레메트리**가 실림.  
  이를 hook으로 기록해 서브에이전트별 비용을 추적함

| 필드 | 의미 |
|------|------|
| `totalTokens` | 서브에이전트 전 턴에 청구된 총 토큰 |
| `usage` | `input_tokens`/`output_tokens`/`cache_creation_input_tokens`/`cache_read_input_tokens` 분해 |
| `totalDurationMs` | 실행 wall-clock 시간 |
| `totalToolUseCount` | 서브에이전트가 호출한 도구 수 |
| `resolvedModel` | 실제 실행 모델(요청 모델과 다를 수 있음) |

```json
// PostToolUse, matcher "Agent": 서브에이전트 토큰·시간을 로그로 적재 (command hook)
{ "hooks": { "PostToolUse": [ { "matcher": "Agent", "hooks": [
  { "type": "command",
    "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/log-subagent-cost.sh", "args": [] } ] } ] } }
```

- **SubagentStart 컨텍스트 주입**: 서브에이전트 생성 시 `additionalContext`로 "예산·범위 제한" 지침을 주입함  
  (단, SubagentStart는 생성 자체를 차단하지는 못함)

### 2.4 Hook 자체 비용 최소화 (가장 중요)

Hooks는 비용을 통제하는 도구이지만, **잘못 쓰면 그 자체가 비용원**이 됨. 다음 규칙을 지킴.

- **타입 선택**: 결정적 판정은 `command`(토큰 0). LLM 판단이 꼭 필요할 때만 `prompt`, 파일·테스트 검사가  
  필요할 때만 `agent`를 사용함. `agent`는 최대 50턴 도는 별도 에이전트라 가장 비쌈
- **빈도 통제**: `prompt`/`agent` hook을 **도구 호출마다 발생하는 이벤트**(`PreToolUse`/`PostToolUse`)에 달면  
  매 호출이 LLM 호출이 되어 비용이 폭증함. LLM 판단형 hook은 **턴당 1회 이벤트**(`Stop`/`SubagentStop`)에 둠
- **범위 축소**: matcher와 `if`로 발생 대상을 최대한 좁혀 불필요한 실행·spawn을 제거함
- **지연 = 비용**: 동기 hook은 모델 진행을 **블로킹**함. `UserPromptSubmit`은 기본 timeout이 30초로 짧으니  
  무거운 작업은 피함. 차단이 필요 없는 후처리(테스트·배포 등)는 `async:true`로 비차단 실행함

```json
// 비차단(async) 후처리: 파일 수정 후 테스트를 백그라운드 실행, 결과는 다음 턴에 보고
{ "hooks": { "PostToolUse": [ { "matcher": "Write|Edit", "hooks": [
  { "type": "command",
    "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/run-tests-async.sh",
    "args": [], "async": true, "timeout": 300 } ] } ] } }
```

> `async`는 `command` 타입만 지원함. async hook은 차단·결정을 할 수 없음(작업은 이미 진행됨).  
> 결과의 `additionalContext`만 다음 턴에 전달됨.

### 2.5 비용 통제 hook 요약표

| 리스크 | 이벤트 | hook 타입 | 조치 |
|--------|--------|-----------|------|
| 무한 루프 | `Stop`/`SubagentStop` | command | `stop_hook_active` 확인 + 완료 조건 검사, 8회 cap 신뢰 |
| 토큰 누수 | `PreToolUse` | command | `updatedInput`으로 입력 절삭, 컨텍스트는 요약만 주입 |
| Agent 폭주 | `PreToolUse`/`PostToolUse(Agent)` | command | 고비용 도구 `deny`, `totalTokens`·`usage` 적재로 추적 |
| hook 비용 | (전 이벤트) | command 우선 | LLM형은 저빈도 이벤트·좁은 matcher로 제한 |
| 지연 | `PostToolUse` 등 | command+async | 비차단 후처리, 짧은 timeout 준수 |

---

## 3. 보안(Security) 측면 Hooks 활용

> **핵심 관점**: 보안은 프롬프트가 아니라 **구조적 통제**로 달성함.  
> Hooks로 ① 권한 오남용 차단 ② 데이터 유출(마스킹·전송 통제) ③ Prompt Injection 방어 ④ hook 자체 보안 ⑤ 조직 통제를 수행함.

### 3.1 권한 오남용 차단 — `PreToolUse` deny (우회 불가)

`PreToolUse` hook은 **권한 모드 검사보다 먼저** 실행됨. `permissionDecision:"deny"`를 반환하면  
`bypassPermissions` 모드에서도 도구가 차단됨. 사용자가 우회할 수 없는 정책 게이트를 만들 수 있음.

```bash
#!/bin/bash
# .claude/hooks/block-rm.sh — 파괴적 명령 차단 (PreToolUse, matcher "Bash")
COMMAND=$(jq -r '.tool_input.command')
if echo "$COMMAND" | grep -q 'rm -rf'; then
  jq -n '{ hookSpecificOutput: {
    hookEventName: "PreToolUse",
    permissionDecision: "deny",
    permissionDecisionReason: "파괴적 명령은 hook 정책으로 차단됨" } }'
else
  exit 0   # 결정 없음 → 정상 권한 흐름 진행
fi
```

- `if`로 대상을 좁힘: `"Bash(rm *)"`, `"Bash(git push *)"` 등. 단 `if`는 best-effort이므로  
  **확정적 차단은 permission system 병행** 권장
- 다중 `PreToolUse` hook 결정 우선순위: `deny > defer > ask > allow`
- **HITL 게이트**: `permissionDecision:"ask"`로 고위험 작업(삭제·결제·외부전송)에 사람 승인을 강제함

### 3.2 데이터 유출 방지 — 입출력 정제

- **PostToolUse `updatedToolOutput`**: 도구 결과를 Claude에 전달하기 **전에 치환**함. 민감정보 마스킹/redaction에 사용  
  (값은 해당 도구의 출력 **형태(shape)와 일치**해야 함. 예: `Bash`는 `{stdout, stderr, interrupted, isImage}`)

```json
{ "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "updatedToolOutput": { "stdout": "[redacted]", "stderr": "", "interrupted": false, "isImage": false } } }
```

- **방향별 차단 지점**: 외부로 나가는 입력은 **`PreToolUse`**, 들어오는 결과는 **`PostToolUse`**에서 정제함
- **중요 한계**: `updatedToolOutput`은 **Claude가 보는 내용만** 바꿈. 도구는 이미 실행되어 파일 쓰기·네트워크 전송은  
  **이미 발생**했고 텔레메트리도 원본을 기록함. 실제 전송을 막으려면 반드시 **`PreToolUse`로 사전 차단**해야 함

### 3.3 Prompt Injection 방어 — `UserPromptSubmit` 검증

- `UserPromptSubmit` hook으로 악성 패턴을 탐지해 `decision:"block"`으로 프롬프트를 거부함
- **주입 컨텍스트는 사실형으로**: `additionalContext`는 명령형("~하라")이 아닌 사실 진술("배포 대상은 production임")로  
  작성함. 명령형 텍스트는 Claude의 prompt-injection 방어를 자극해 컨텍스트가 아니라 경고로 노출될 수 있음
- 외부 문서 본문은 신뢰 컨텍스트와 분리해 불신 데이터로 다룸(개념은 [하네스 가이드 5.1](harness-engineering-guide.md) 참조)

### 3.4 Hook 자체 보안 — 풀 권한 실행 주의

> **공식 경고**: command hook은 **사용자 계정의 전체 권한**으로 셸을 실행함.  
> 사용자가 접근 가능한 모든 파일을 수정·삭제·열람할 수 있음. 추가 전 반드시 검토·테스트함.

공식 보안 best practice는 다음과 같음.

- **입력 검증·정제**: stdin 입력을 절대 맹신하지 않음
- **셸 변수 항상 큰따옴표**: `$VAR`가 아니라 `"$VAR"` 사용
- **경로 탐색 차단**: 파일 경로에 `..` 포함 여부 검사
- **절대 경로 사용**: 스크립트는 `${CLAUDE_PROJECT_DIR}`/`${CLAUDE_PLUGIN_ROOT}` 등 절대 경로로 참조  
  (exec form은 인자별 1개로 전달되어 따옴표 불필요, shell form은 placeholder를 큰따옴표로 감쌈)
- **민감 파일 회피**: `.env`, `.git/`, 키 파일 등은 건드리지 않음

### 3.5 조직 통제 (관리자/플러그인)

엔터프라이즈 배포 시 관리형 설정(managed settings)으로 hook 공급망을 통제함.

| 설정 | 위치 | 효과 |
|------|------|------|
| `allowManagedHooksOnly: true` | managed only | 관리형·SDK hook과 **관리형에 force-enable된 플러그인** hook만 허용. user/project/기타 플러그인 hook 차단 |
| `allowedHttpHookUrls` | 모든 레벨 | http hook이 보낼 수 있는 URL을 화이트리스트로 제한(`*` 와일드카드, 미일치는 무시) |
| `httpHookAllowedEnvVars` | 모든 레벨 | http hook 헤더에 끼워넣을 수 있는 환경변수명을 제한(교집합으로 적용) |
| `disableAllHooks: true` | settings | 전체 hook 일시 비활성(관리형 hook은 managed 레벨에서만 비활성 가능) |
| `strictPluginOnlyCustomization` | managed only | skills/hooks 등을 플러그인·관리형 출처로만 한정 |

- http hook의 인증 토큰은 `headers` + `allowedEnvVars`로만 주입되며, **목록에 없는 변수는 빈 문자열로 대체**됨
- 플러그인 hook은 `plugin@marketplace` 전체 ID로 신뢰가 부여되어, 동일 이름이라도 다른 마켓플레이스면 차단됨

### 3.6 보안 통제 hook 요약표

| 리스크 | 이벤트 | 조치 |
|--------|--------|------|
| 권한 오남용 | `PreToolUse` | `deny`/`ask`(권한 모드 우선·우회 불가), `if`로 고위험 작업 분류 |
| 데이터 유출 | `PreToolUse`(전송 전)·`PostToolUse`(표시 전) | 입력 차단·결과 마스킹(`updatedToolOutput`) |
| Prompt Injection | `UserPromptSubmit` | 악성 패턴 `block`, 컨텍스트는 사실형 주입 |
| hook 자체 | (작성 시) | 입력검증·변수 quoting·경로탐색 차단·절대경로·민감파일 회피 |
| 공급망 | managed settings | `allowManagedHooksOnly`·URL/env 화이트리스트·플러그인 ID 신뢰 |

---

## 4. 개발 대상별 적용 (Skill / Agent / 플러그인)

### 4.1 Skill

- Skill의 **frontmatter에 직접 hooks 정의** 가능. 해당 Skill이 활성인 동안에만 동작하고 종료 시 정리됨
- 모든 이벤트 지원. `once: true`는 **Skill frontmatter에서만** 유효함(세션당 1회 실행 후 제거)

```yaml
---
name: secure-operations
description: 보안 검사를 동반한 작업 수행
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/security-check.sh"
---
```

### 4.2 Agent (서브에이전트)

- Agent도 frontmatter에 동일 형식으로 hooks 정의함. **`Stop` hook은 자동으로 `SubagentStop`으로 변환**됨
- `SubagentStart`로 생성 시 지침 주입(차단 불가), `SubagentStop`으로 완료 검증·계속 지시 가능
- 부모 세션에 서브에이전트 결과를 주입하려면 `Agent` 도구에 대한 `PostToolUse` hook을 사용함

### 4.3 플러그인

- 플러그인은 `hooks/hooks.json`에 hook을 번들하며, 활성화 시 user/project hook과 병합됨
- 스크립트 경로는 `${CLAUDE_PLUGIN_ROOT}`(설치 디렉터리, 업데이트마다 변경),  
  영속 데이터는 `${CLAUDE_PLUGIN_DATA}`로 참조함
- 경로 placeholder를 쓰는 hook은 **exec form**(`args` 사용) 권장 — 공백·특수문자 따옴표 불필요  
  (Windows에서 `.cmd`/`.bat` 셋업은 exec form 불가 → `node`로 스크립트 직접 실행하거나 shell form 사용)

```json
{ "description": "자동 코드 포매팅",
  "hooks": { "PostToolUse": [ { "matcher": "Write|Edit", "hooks": [
    { "type": "command", "command": "${CLAUDE_PLUGIN_ROOT}/scripts/format.sh", "args": [], "timeout": 30 } ] } ] } }
```

---

## 5. 9대 리스크 ↔ Hook 매핑표

[하네스 엔지니어링 적용 가이드](harness-engineering-guide.md)의 9대 리스크를 hooks로 구현하는 대응표임.

| # | 리스크 | 권장 이벤트 | hook 타입 | 핵심 조치 |
|---|--------|-------------|-----------|-----------|
| 1 | 무한 루프 | `Stop`/`SubagentStop` | command | `stop_hook_active` 확인, 완료 검사, 8회 block cap |
| 2 | 토큰 누수 | `PreToolUse` | command | `updatedInput` 절삭, 컨텍스트 10,000자 상한 |
| 3 | Agent 폭주 | `PreToolUse`/`PostToolUse(Agent)` | command | 고비용 도구 `deny`, `totalTokens`/`usage` 추적 |
| 4 | 시스템 마비 | `PostToolUseFailure` | command | 실패 로깅·알림, `additionalContext`로 복구 지침 |
| 5 | 응답 지연 | `PostToolUse` | command+async | 비차단 후처리, 짧은 timeout |
| 6 | 할루시네이션 전파 | `Stop`/`PostToolUse` | command(필요 시 agent) | 스키마·근거·테스트 검증 후 `block` |
| 7 | Prompt Injection | `UserPromptSubmit` | command | 악성 패턴 `block`, 사실형 컨텍스트 |
| 8 | 데이터 유출 | `PreToolUse`/`PostToolUse` | command | 전송 전 차단·표시 전 마스킹 |
| 9 | 권한 오남용 | `PreToolUse` | command | `deny`/`ask` 게이트(권한 모드 우선) |

> 표의 거의 모든 행이 `command`임에 주목함 — **결정적 통제가 기본, LLM형(prompt/agent)은 예외**라는 비용 원칙을 반영함.

---

## 6. 복붙용 설정 예시 (검증된 JSON 형식)

### 6.1 비용 — Stop 완료 게이트(무한 루프 방지 포함)

```json
{ "hooks": { "Stop": [ { "hooks": [
  { "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/check-done.sh", "args": [] } ] } ] } }
```

### 6.2 보안 — 파괴적 Bash 명령 차단(권한 모드 우회 불가)

```json
{ "hooks": { "PreToolUse": [ { "matcher": "Bash", "hooks": [
  { "type": "command", "if": "Bash(rm *)",
    "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh", "args": [] } ] } ] } }
```

### 6.3 보안 — 외부 검증 서비스(HTTP hook, URL·토큰 통제)

```json
{ "hooks": { "PreToolUse": [ { "matcher": "Bash", "hooks": [
  { "type": "http", "url": "http://localhost:8080/hooks/pre-tool-use", "timeout": 30,
    "headers": { "Authorization": "Bearer $MY_TOKEN" }, "allowedEnvVars": ["MY_TOKEN"] } ] } ] } }
```

---

## 7. 배포 전 체크리스트

### 비용
- [ ] LLM형(prompt/agent) hook이 `PreToolUse`/`PostToolUse` 등 고빈도 이벤트에 달려 있지 않음
- [ ] 결정적 게이트는 `command`로 구현됨(불필요한 토큰 미소비)
- [ ] matcher·`if`로 발생 대상이 최대한 좁혀짐
- [ ] `Stop` hook은 `stop_hook_active`를 확인해 무한 루프를 방지함
- [ ] 차단 불필요한 후처리는 `async:true`로 비차단 실행됨, timeout이 설정됨
- [ ] 서브에이전트 비용(`totalTokens`/`usage`)이 측정·기록됨

### 보안
- [ ] 고위험 작업이 `PreToolUse` `deny`/`ask`로 게이트됨(권한 모드 우회 불가 확인)
- [ ] 실제 외부 전송 차단은 `PreToolUse`(사전)에서 처리됨(`PostToolUse`는 표시만 변경)
- [ ] 민감정보 마스킹이 `updatedToolOutput`으로 적용됨
- [ ] `UserPromptSubmit`에서 악성 패턴이 검증·차단됨, 주입 컨텍스트는 사실형임
- [ ] command hook이 입력검증·변수 quoting·경로탐색 차단·절대경로·민감파일 회피를 지킴
- [ ] (조직) `allowManagedHooksOnly`·`allowedHttpHookUrls`·`httpHookAllowedEnvVars`로 공급망이 통제됨

---

## 8. 주의사항·한계 (Fact)

- **차단 코드**: 정책 강제는 `exit 1`이 아니라 **`exit 2`**(예외: `WorktreeCreate`는 모든 비0 코드가 중단)
- **PostToolUse는 되돌릴 수 없음**: 도구가 이미 실행된 뒤 발생함. 사전 차단은 `PreToolUse`
- **PermissionRequest는 비대화 모드(`-p`)에서 미발생** → 자동 권한 결정은 `PreToolUse` 사용
- **`if` 필터는 best-effort**: 파싱 실패 시 fail-open. 하드 강제는 permission system 병행
- **다중 `updatedInput` 경합**: 병렬 실행이라 마지막 완료가 이김(비결정적). 같은 도구 입력을 둘 이상이 고치지 않게 함
- **timeout 기본값**: command/http/mcp_tool 600초(`UserPromptSubmit` 30초, `MessageDisplay` 10초), prompt 30초, agent 60초
- **`agent` hook은 실험적**: 동작·설정이 변경될 수 있음. 프로덕션은 `command` hook 권장
- **컨텍스트 stale**: `--resume`/`--continue` 시 과거 턴의 `additionalContext`는 재실행 없이 재생됨  
  (타임스탬프·커밋 SHA 등은 낡을 수 있음). `SessionStart`는 resume 시 다시 실행되어 갱신 가능
- **`/hooks` 메뉴**: 설정된 hook을 읽기 전용으로 확인 가능. 추가·수정은 settings JSON 직접 편집

---

## 9. 출처 (Claude 공식 문서)

- Hooks reference — <https://docs.claude.com/en/docs/claude-code/hooks>
- Automate actions with hooks (guide) — <https://docs.claude.com/en/docs/claude-code/hooks-guide>
- Settings(Hook configuration 포함) — <https://docs.claude.com/en/docs/claude-code/settings>
- Plugins reference(hooks 번들) — <https://docs.claude.com/en/docs/claude-code/plugins-reference>

> 본 문서의 모든 Fact는 위 공식 문서(2026-06-18 기준)에 근거함. 버전 의존 기능은 최신 Claude Code 기준이며,  
> 실제 적용 전 사용 중인 버전의 문서로 재확인할 것.
