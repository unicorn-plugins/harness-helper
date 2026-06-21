# 하네스 엔지니어링 적용 도우미 — `harness-helper` 플러그인

대상 Claude Code 플러그인을 **비용·성능·보안 하네스 통제** 관점에서 점검하고,  
적용 가능(✅/△) 미적용 항목의 **보완방안·적용 로드맵**을 도출하는 점검 플러그인.

배포 전 플러그인의 MAS 9대 실행 리스크 통제 적용 여부를 진단하고, 사용자가 선택한 항목을 실제로 보완함.

---

## 주요 기능

- **단일 명령 5단계 점검**: `/harness-helper:check {대상경로}` 한 번으로 인벤토리→재판정→병렬 점검→
  기록·로드맵→보고 수행 (보완 구현은 점검 범위에서 분리 — `apply`로 별도 진행)
- **멀티 에이전트 분업**: 비용·성능·보안을 전담 점검가 3인이 **병렬** 점검 후, 오케스트레이터가 통합
- **적용 가능 항목만 점검**: `references/harness-checker.md`의 적용 가능성 재판정으로 ✅(적용 가능)·△(조건부)
  항목만 Y/N 판정, ❌ 구조적 불가(7장)는 대안만 안내
- **근거 기반 진단**: 모든 Y 판정에 파일:라인 근거 병기 (근거 없는 Y는 무효)
- **선택 보완 구현(언제든)**: `apply`가 미적용 항목을 **영역→리스크→항목 ID(C1·P2·S3) 문단**으로 제시하면  
  **ID로 복수 선택**해 빌디가 6장 레시피로 구현·검증 후 기록 갱신(완료 항목은 취소선+`[적용됨]`으로 ID 보존).  
  점검 직후뿐 아니라 추후 세션에서도 독립 실행 가능

---

## 구성 요소

| 유형 | 이름 | 네임스페이스 호출 | 역할 |
|------|------|------------------|------|
| 스킬 | `check` | `/harness-helper:check` | 오케스트레이션·5단계 점검 워크플로우 |
| 스킬 | `apply` | `/harness-helper:apply` | 미적용(✅/△) 항목 복수 선택·보완 구현 |
| 에이전트 | `cost-checker` | `harness-helper:cost-checker` | 비용 통제 점검 (harness-checker 2장) |
| 에이전트 | `performance-checker` | `harness-helper:performance-checker` | 성능 통제 점검 (3장) |
| 에이전트 | `security-checker` | `harness-helper:security-checker` | 보안 통제 점검 (4장) |
| 에이전트 | `implementation-engineer` | `harness-helper:implementation-engineer` | 선택 항목 보완 구현 (6장) |
| 참조 | `references/*.md` | — | 검사기·엔지니어링·hook·프롬프트 가이드 |

> 오케스트레이터(클로니)는 별도 에이전트 파일 없이 스킬을 구동하는 **메인 스레드**임.

---

## 사전 요구사항

| 의존성 | 용도 | 준비 방법 |
|--------|------|----------|
| **Node.js** | 빌디의 hook형 보완 구현 시 `.mjs` 검증(`node`) | 작업 환경에 Node.js 설치 |
| **대상 플러그인 경로** | 점검·보완 대상 | 로컬에 접근 가능한 Claude Code 플러그인 디렉토리 |

---

## 설치

```bash
# 로컬 개발/테스트
claude plugin marketplace add .
claude plugin install harness-helper@harness-helper
```

> 일회성 로드: 리포 루트에서 `claude --plugin-dir .`

## 사용

```text
# 점검·기록·로드맵 도출 (보완 구현은 apply로 별도 진행)
/harness-helper:check C:\path\to\target-plugin

# 미적용 항목을 언제든 복수 선택하여 보완 구현
/harness-helper:apply C:\path\to\target-plugin
```

또는 자연어로 `이 플러그인 하네스 점검 해줘`·`미적용 항목 구현해줘` 입력 시 모델이 해당 스킬을 자동 호출함.

---

## 디렉토리 구조

```text
harness-helper/                        # 리포 루트 = 마켓플레이스 루트 = 플러그인 루트
├── .claude-plugin/
│   ├── marketplace.json               # name: harness-helper
│   └── plugin.json                    # name: harness-helper (hooks 필드 없음)
├── skills/
│   ├── check/SKILL.md         # 5단계 점검 워크플로우
│   └── apply/SKILL.md         # 미적용 항목 복수 선택·보완 구현
├── agents/
│   ├── cost-checker.md                # 코스티
│   ├── performance-checker.md         # 스피디
│   ├── security-checker.md            # 가디
│   └── implementation-engineer.md     # 빌디
├── commands/
│   ├── check.md               # /harness-helper:check
│   └── apply.md               # /harness-helper:apply
├── references/                        # 검사기·엔지니어링·hook·프롬프트 가이드
├── prompts/                           # 플러그인 제작 프롬프트(제작 지시서)
├── CLAUDE.md                          # @AGENTS.md
├── AGENTS.md                          # 팀 설계 + 하네스 포인터
└── README.md
```

> 플러그인 내부 파일 참조는 모두 `${CLAUDE_PLUGIN_ROOT}/...` 변수를 사용함.

---

## 하네스 자가 점검

harness-helper는 **점검 중심·자체 hook 미포함** 설계임. 자기 적용 결과는 아래와 같음.

| 항목 | 적용 | 근거·사유 |
|------|:---:|------|
| 최소 권한 | ✅ Y | 점검가 3인 `tools`=Read/Grep/Glob(읽기 전용), 빌디=Read/Write/Edit/Bash로 한정 |
| 깊이 제한 | ✅ Y | 서브에이전트에 `Agent` 도구 미부여 → 추가 스폰 차단 |
| 외부 전송(egress) | 해당 없음 | 네트워크 전송 도구 미보유 → 데이터 유출 표면 없음 |
| hook 기반 항목(감사 로그·DLP·HITL·호출 카운터 등) | 미적용 | 설계상 자체 hook 미포함 — 필요 시 별도 도입 |

> 점검가는 읽기 전용 도구만 보유하여 위험이 낮음. 파일 변경 권한은 구현 단계의 빌디에만 한정됨.

---

## 라이선스

© harness-helper팀
