# Claude Code 훅(Hook) 작성 가이드

> Claude Code 플러그인·프로젝트의 훅을 작성할 때 참조하는 범용 가이드.  
> 등록 방법 · 출력 형식 · 크로스 플랫폼 주의사항 · 테스트 절차를 포함.

---

## 1. 훅(Hook)이란?

Claude Code가 특정 동작을 수행하는 순간에 끼어들어 **허용 · 차단 · 수정 · 컨텍스트 주입** 등을 자동으로 실행하는 규칙.  
훅은 shell 명령 또는 HTTP 요청으로 구현하며, JSON을 stdout으로 출력하거나 exit code로 동작을 지시함.

---

## 2. 대표 이벤트

| 이벤트 | 트리거 시점 | 차단 가능 | 주요 용도 |
|--------|------------|-----------|---------|
| `UserPromptSubmit` | 사용자가 프롬프트 전송 시 | ✅ | PII 감지·차단, 보안 정책 주입 |
| `PreToolUse` | 도구(Write·Bash 등) 실행 직전 | ✅ | 위험 명령 차단, 입력값 수정, 권한 제어 |
| `PostToolUse` | 도구 실행 직후 | ✗ (이미 실행됨) | 결과 교체·마스킹, 보충 컨텍스트 주입 |
| `SessionStart` | 새 세션 시작 시 | ✗ | 환경변수 설정, 컨텍스트 사전 주입 |
| `Stop` | Claude가 응답을 종료하려 할 때 | ✅ | 빌드 실패 시 중단 방지, 후속 작업 주입 |
| `SubagentStop` | 서브에이전트 종료 시 | ✅ | 서브에이전트 결과 검증 |
| `CwdChanged` | 작업 디렉토리 변경 시 | ✗ | 환경변수 재설정 |
| `ConfigChange` | Claude 설정 변경 시 | ✅ | 설정 변경 감사·차단 |

---

## 3. 훅 등록 위치

### 3-1. 플러그인 훅 (권장)

`.claude-plugin/plugin.json`의 `hooks` 필드에 직접 정의.  
- user scope에서도 정상 로딩됨 (Desktop `--setting-sources user` 모드 포함)  
- `${CLAUDE_PLUGIN_ROOT}` 변수로 버전 업 시 경로 자동 추적  
- `hooks/hooks.json`은 Desktop `--setting-sources user` 모드에서 **미로딩** → 사용 금지

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node",
            "args": ["${CLAUDE_PLUGIN_ROOT}/hooks/my-hook.mjs"],
            "timeout": 15,
            "statusMessage": "정책 검사 중..."
          }
        ]
      }
    ]
  }
}
```

### 3-2. 프로젝트 훅

`.claude/settings.json`의 `hooks` 필드에 정의. 해당 프로젝트 디렉토리에서만 동작.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node",
            "args": ["${CLAUDE_PROJECT_DIR}/.claude/hooks/check-bash.mjs"]
          }
        ]
      }
    ]
  }
}
```

### 3-3. 글로벌 훅

`~/.claude/settings.json`에 정의. 모든 세션·디렉토리에 적용.

### 3-4. 매처(matcher) 옵션

`PreToolUse` / `PostToolUse`는 `matcher` 필드로 특정 도구만 필터링 가능.

```json
{ "matcher": "Write|Edit",  "hooks": [...] }   // Write 또는 Edit 도구에만 적용
{ "matcher": "Bash",        "hooks": [...] }   // Bash 도구에만 적용
{ "matcher": "",            "hooks": [...] }   // 모든 도구에 적용 (matcher 생략과 동일)
```

---

## 4. 훅 출력 방법

훅 프로그램은 JSON을 stdout으로 출력하여 Claude에게 지시.  
아무것도 출력하지 않고 exit 0이면 통과(pass-through).

### 4-1. `decision: "block"` — 차단

```json
{
  "decision": "block",
  "reason": "사용자에게 표시할 차단 사유",
  "systemMessage": "Claude에게 전달할 메시지 (선택)"
}
```

**적용 이벤트**: `UserPromptSubmit`, `PreToolUse`, `Stop`, `SubagentStop`, `ConfigChange` 등  
**주의**: Claude Code Desktop 새 세션에서 `reason`이 UI에 표시되지 않을 수 있음  
→ OS 알림으로 보완 필요 (아래 6절 참조)

### 4-2. `additionalContext` — 컨텍스트 주입

Claude의 시스템 프롬프트에 텍스트를 자동 주입. 보안 정책·환경 정보 등에 활용.

```json
{
  "hookSpecificOutput": {
    "hookEventName": "UserPromptSubmit",
    "additionalContext": "여기에 주입할 텍스트"
  }
}
```

**적용 이벤트**: 거의 모든 이벤트 (`UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `SessionStart` 등)

### 4-3. `permissionDecision` — 권한 제어 (PreToolUse 전용)

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "거부 사유",
    "additionalContext": "Claude에 전달할 보충 설명"
  }
}
```

| 값 | 동작 |
|----|------|
| `"allow"` | 권한 프롬프트 없이 즉시 허용 |
| `"deny"` | 도구 실행 거부 |
| `"ask"` | 사용자에게 확인 요청 |
| `"defer"` | 다음 훅에 위임 |

### 4-4. `updatedInput` — 입력값 수정 (PreToolUse 전용)

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "updatedInput": {
      "command": "수정된 명령어"
    }
  }
}
```

### 4-5. `updatedToolOutput` — 결과 교체 (PostToolUse 전용)

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "updatedToolOutput": "마스킹·교체된 결과 텍스트"
  }
}
```

### 4-6. `continue: false` — 세션 중단

Claude 세션 전체를 즉시 중단. 모든 이벤트에 적용 가능.

```json
{
  "continue": false,
  "stopReason": "사용자에게 표시할 중단 사유"
}
```

### 4-7. `CLAUDE_ENV_FILE` — 환경변수 설정 (SessionStart 전용)

`$CLAUDE_ENV_FILE` 환경변수가 가리키는 파일에 `export KEY=VALUE` 형식으로 기록.  
기록된 환경변수는 이후 Claude 세션 전체에 적용됨.

```javascript
import { appendFileSync } from "node:fs";
const envFile = process.env.CLAUDE_ENV_FILE;
if (envFile) {
  appendFileSync(envFile, "NODE_ENV=production\n");
  appendFileSync(envFile, "DEBUG=false\n");
}
```

### 4-8. exit code 동작

| exit code | 동작 |
|-----------|------|
| `0` | 정상 통과 |
| `2` | 차단 (JSON 출력 없이 차단 시 사용) |
| 기타 | 비차단 오류 (stderr이 사용자에게 표시) |

---

## 5. 훅 입력 형식

훅 프로그램은 stdin으로 JSON을 받음. 이벤트에 따라 구조가 다름.

### UserPromptSubmit 입력

```json
{
  "session_id": "...",
  "prompt": "사용자가 입력한 프롬프트 전문",
  "cwd": "/current/working/directory"
}
```

### PreToolUse / PostToolUse 입력

```json
{
  "session_id": "...",
  "tool_name": "Write",
  "tool_input": {
    "file_path": "/path/to/file",
    "content": "..."
  },
  "tool_response": { ... }
}
```

---

## 6. 훅 프로그램 작성 가이드

### 6-1. 파일 형식: `.mjs` (ES Module Node.js) 권장

- **Windows 환경에서 `bash`/`sh` 무음 실패** → 반드시 `node`로 실행
- `.mjs` 확장자 = ES Module 강제 적용 (`package.json` `type` 설정 무관)
- `import`/`export` 문법 및 `import.meta.dirname` 사용 가능

```json
// plugin.json hooks 등록 예시
{ "type": "command", "command": "node", "args": ["${CLAUDE_PLUGIN_ROOT}/hooks/my-hook.mjs"] }
```

### 6-2. stdin 처리

훅은 항상 stdin을 읽어야 함. 읽지 않으면 파이프가 막힐 수 있음.

```javascript
import { readFileSync } from "node:fs";
let raw;
try { raw = readFileSync(0, "utf8"); } catch { process.exit(0); }
let input;
try { input = JSON.parse(raw); } catch { process.exit(0); }
```

### 6-3. fail-open 설계

훅 내부 오류로 정상 워크플로우를 차단하면 안 됨. 예외 발생 시 exit 0으로 통과.

```javascript
try {
  main();
} catch {
  process.exit(0); // 내부 오류 → fail-open (통과)
}
```

### 6-4. OS 알림 (block 차단 시 사용자 통지)

Claude Code Desktop 새 세션에서 `decision: block`의 `reason`이 UI에 표시되지 않을 수 있음.  
OS 알림으로 보완 권장.

```javascript
import { execFileSync } from "node:child_process";

function notifyOS(title, body) {
  try {
    if (process.platform === "win32") {
      const safe = body.replace(/'/g, "`");
      execFileSync("powershell", [
        "-NonInteractive", "-Command",
        `$wsh = New-Object -ComObject WScript.Shell; $wsh.Popup('${safe}', 8, '${title}', 48)`
      ], { stdio: "ignore", timeout: 10000 });
    } else if (process.platform === "darwin") {
      const esc = body.replace(/\\/g, "\\\\").replace(/"/g, '\\"');
      execFileSync("osascript", [
        "-e", `display notification "${esc}" with title "${title}" sound name "Basso"`
      ], { stdio: "ignore", timeout: 5000 });
    }
  } catch { /* 알림 실패해도 차단은 유지 */ }
}
```

| 플랫폼 | 방법 | 특징 |
|--------|------|------|
| Windows | `WScript.Shell.Popup` (PowerShell) | 8초 자동 닫힘, 아이콘 48=경고 |
| macOS | `osascript display notification` | 토스트, 사운드 지정 가능 |
| Linux | `notify-send` (libnotify) | 데스크톱 환경에 따라 다름 |

### 6-5. 전체 훅 템플릿

```javascript
#!/usr/bin/env node
import { readFileSync } from "node:fs";
import { execFileSync } from "node:child_process";

function notifyOS(title, body) {
  try {
    if (process.platform === "win32") {
      const safe = body.replace(/'/g, "`");
      execFileSync("powershell", ["-NonInteractive", "-Command",
        `$wsh = New-Object -ComObject WScript.Shell; $wsh.Popup('${safe}', 8, '${title}', 48)`],
        { stdio: "ignore", timeout: 10000 });
    } else if (process.platform === "darwin") {
      const esc = body.replace(/\\/g, "\\\\").replace(/"/g, '\\"');
      execFileSync("osascript", ["-e",
        `display notification "${esc}" with title "${title}" sound name "Basso"`],
        { stdio: "ignore", timeout: 5000 });
    }
  } catch {}
}

function main() {
  // 1. stdin 읽기
  let raw;
  try { raw = readFileSync(0, "utf8"); } catch { process.exit(0); }
  let input;
  try { input = JSON.parse(raw); } catch { process.exit(0); }

  // 2. 검사 로직 (이벤트에 따라 input.prompt 또는 input.tool_input 사용)
  const prompt = typeof input?.prompt === "string" ? input.prompt : "";
  if (!prompt) process.exit(0);

  // 3-A. 차단이 필요한 경우
  // const reason = "차단 사유";
  // notifyOS("Claude 보안 알림", reason);
  // process.stdout.write(JSON.stringify({ decision: "block", reason, systemMessage: reason }));
  // process.exit(0);

  // 3-B. 컨텍스트 주입만 필요한 경우
  // process.stdout.write(JSON.stringify({
  //   hookSpecificOutput: {
  //     hookEventName: "UserPromptSubmit",
  //     additionalContext: "주입할 정책·정보 텍스트",
  //   }
  // }));

  process.exit(0); // 통과
}

try { main(); } catch { process.exit(0); }
```

---

## 7. 테스트 방법

플러그인 훅 변경 후 반드시 아래 3단계를 수행해야 반영됨.

### STEP 1 — 버전 업 및 플러그인 업데이트

`plugin.json`의 `version` 필드를 올린 뒤 업데이트 명령 실행.

```
프롬프트 입력: 'patch 버전 올리고 플러그인 업데이트'
```

Claude가 자동으로 version bump + `claude plugin update {플러그인명}` 수행.

### STEP 2 — Claude Code 재시작

훅 변경사항은 **재시작 후에만 적용**됨. 새 대화(New Chat)만으로는 갱신 안 됨.

- Claude Desktop 앱: 종료 후 재시작
- 터미널 claude: 세션 종료 후 재시작

### STEP 3 — 훅 조건 텍스트 입력하여 테스트

| 훅 종류 | 테스트 방법 | 성공 판정 |
|--------|------------|---------|
| `additionalContext` | 새 세션 system-reminder 확인 | 주입 텍스트가 context에 포함됨 |
| `decision: block` | 차단 조건 텍스트 입력 | OS 팝업 표시 + 프롬프트 차단 |
| `permissionDecision: deny` | 해당 도구 호출 유발 프롬프트 입력 | 도구 실행 거부 메시지 표시 |

---

## 8. 자주 하는 실수

| 실수 | 원인 | 해결 |
|------|------|------|
| 훅이 동작하지 않음 | `hooks/hooks.json`에 등록 | `plugin.json` `hooks` 필드로 이전 |
| Windows에서 무음 실패 | `bash`/`sh` 사용 | `node` (.mjs)로 교체 |
| 새 세션에서 block 팝업 미표시 | Desktop UI 미표시 버그 | OS 알림 코드 추가 |
| stdin 파이프 블로킹 | stdin 미소비 | `readFileSync(0, "utf8")` 추가 |
| 훅 오류가 작업 차단 | 예외 미처리 | `try { main(); } catch { process.exit(0); }` |
| 재시작 전 테스트 | 변경 미반영 | 항상 재시작 후 테스트 |

---

## 9. 주요 경로 변수

| 변수 | 의미 | 사용 위치 |
|------|------|---------|
| `${CLAUDE_PLUGIN_ROOT}` | 플러그인 설치 루트 경로 | `plugin.json` hooks args |
| `${CLAUDE_PROJECT_DIR}` | 현재 프로젝트 디렉토리 | `settings.json` hooks args |
| `${CLAUDE_PLUGIN_DATA}` | 플러그인 데이터 저장 경로 | `plugin.json` hooks args |
| `$CLAUDE_ENV_FILE` | 환경변수 기록 파일 경로 | 훅 프로그램 내부 (SessionStart) |

---

## 10. 참고

- GitHub issue #27398: `hooks/hooks.json` Desktop user scope 미로딩 (수정 계획 없음)
- GitHub issue #18547: 동일 문제 (closed as "not planned")
- Lessons Learned: `AGENTS.md` → Lessons Learned 섹션 참조
