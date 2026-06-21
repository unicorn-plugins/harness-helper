# 프롬프트: harness-helper 플러그인 제작

> 용도: 본 프롬프트를 Claude Code 실행 에이전트에 입력하면, harness-helper 팀 설계(`AGENTS.md`)와 `references/`
> 하네스 진단 자료를 **설치·실행 가능한 단일 Claude Code 플러그인 `harness-helper`**로 구현함.
> 작성 표준: `references/prompt-guide.md` 8종 섹션. 구조 참조: `~/workspace/corp-research`(동일 팀 자매 플러그인).
> 범위: harness-helper는 **점검(체크) 중심** 플러그인이며 **자체 hook은 생성하지 않음**. 점검 대상은
> `references/harness-checker.md`에서 **적용 가능(✅)·조건부(△)로 분류된 항목으로 한정**함.
> 사용법: 아래 `[목표]`~`[예시]` 전체를 실행 에이전트에 전달. 본 프롬프트는 *제작 지시서*이며 플러그인 본체가 아님.

---

[목표]
harness-helper 팀 설계(`AGENTS.md`)와 `references/` 하네스 진단 자료를, 다른 Claude Code 플러그인의 비용·성능·
보안 하네스 통제 중 **적용 가능한 항목을 점검·보완**하는 **단일 Claude Code 플러그인 `harness-helper`**
(marketplace.json·plugin.json·skill·agents·command)로 구현함. (hook 파일은 생성하지 않음)

[역할]
당신은 Claude Code 플러그인 아키텍처(plugin.json·skill·subagent)와 하네스 엔지니어링(MAS 9대 실행 리스크 통제)에
모두 정통한 플러그인 빌드 엔지니어입니다. 적용 가능성(✅/△/❌)을 정확히 판별하고, 동작 증거 없는 완료 보고를
거부하는 정직한 보고 원칙을 고수함.

[맥락]
- 내 상황: harness-helper는 현재 `AGENTS.md`(팀 설계 5인)·`CLAUDE.md`(빈 파일)·`references/`(가이드·검사기)만
  존재하고, `.claude-plugin/`·`skills/`·`agents/`·`commands/` 등 동작 구성요소가 없음. 문서 설계를 설치·실행
  가능한 플러그인으로 전환해야 함.
- 결과물 독자: ① 자신의 Claude Code 플러그인을 배포 전 점검·보완하려는 플러그인 개발자, ② 점검·보완을 수행하는
  harness-helper 팀 멤버(오케스트레이터·점검가 3인·구현 엔지니어).

[입력정보]
- 팀 설계: `AGENTS.md` — 멤버 5인(오케스트레이터 클로니·비용 코스티·성능 스피디·보안 가디·구현 빌디) 프로파일·
  위임 규칙·대화 가이드·정직한 보고 규칙·마크다운 규칙
- 점검 워크플로우·적용 가능 항목: `references/harness-checker.md` — **0-2장 적용 가능성 재판정 규칙(✅/△/❌)**,
  6단계 절차, 비용/성능/보안 체크리스트(2~4장, ✅/△ 32항목), 6장 레시피, 5장 기록부, 7장 구조적 불가(❌), 8장 로드맵
- 리스크 이론: `references/harness-engineering-guide.md` — MAS 9대 리스크, 비용→성능→보안 적용 순서, 심각도 SLA
- 보완 구현 참조(빌디 전용): `references/harness-hooks.md`·`references/hook-guide.md` — 대상 플러그인에 hook형
  보완을 구현할 때 참조하는 hook 이벤트·`.mjs` 규격(harness-helper 자체 hook 생성에는 사용하지 않음)
- 프롬프트 표준: `references/prompt-guide.md` — agents·skill 본문 작성 시 8종 섹션 준용
- 구조 템플릿: `~/workspace/corp-research/` — 동일 팀 스타일의 기존 플러그인. 아래 파일을 규격 기준으로 1:1 참조:
  - `.claude-plugin/{marketplace.json, plugin.json}`, `skills/research/SKILL.md`(Phase·역할분담), `agents/*.md`
    (frontmatter+페르소나 본문), `commands/research.md`(SKILL 호출 1줄), `README.md`,
    `docs/harness-adoption-plan.md`(점검 기록·로드맵 산출 형식)

[작업방법]
1. 소스 인벤토리: `AGENTS.md`·`references/` 전 항목을 읽어 (a) 멤버 5인 → 에이전트 매핑, (b) harness-checker
   6단계 → skill Phase, (c) **적용 가능 항목 집합(✅/△)**을 확정. corp-research 파일별 규격을 대조 기준으로 사용.
2. 매니페스트 생성 `.claude-plugin/`:
   - `marketplace.json`: 마켓플레이스 `name`과 `plugins[0].name`을 **모두 `"harness-helper"`**(사용자 지시),
     `plugins[0].source` `"./"`, `description`·`category:"developer-tools"`·`keywords` 기재
   - `plugin.json`: `$schema`·`name:"harness-helper"`·`displayName`·`version:"0.1.0"`·`description`·`author`·
     `keywords` 기재. **`hooks` 필드는 두지 않음**(harness-helper 자체 hook 미생성)
3. 에이전트 생성 `agents/` (멤버 4인을 1:1 매핑, 오케스트레이터 클로니는 메인 스레드이므로 파일 없음):
   - `cost-checker.md`(코스티)·`performance-checker.md`(스피디)·`security-checker.md`(가디)·
     `implementation-engineer.md`(빌디)
   - frontmatter: `name`·`description`·`model`·`tools`(최소권한). 본문: prompt-guide 8종 섹션으로 작성하되
     `[역할]`에 AGENTS.md 프로파일(닉네임·성향·경력) 반영
   - **점검가 3인 `[작업방법]` 필수 절차**: ⓪ harness-checker 0-2장 규칙으로 적용 가능성 재판정(✅/△/❌) → ✅·△
     항목만 Y/N 판정 → 파일:라인 근거 기재 → N 항목은 6장 레시피로 보완방안 도출. **❌ 구조적 불가(7장)는 Y/N
     판정 대상에서 제외하고 7장 대안만 안내**(담당 장: 코스티=2장, 스피디=3장, 가디=4장)
   - 빌디 `[작업방법]`: 사용자가 선택한 **적용 가능(✅/△) 미적용 항목만** 6장 레시피로 대상 플러그인에 구현.  
     구현 산출물·설명은 **한국어로 작성**
   - tools 최소권한: 점검가 3인 `Read`·`Grep`·`Glob`(읽기 전용), 구현 빌디 `Read`·`Write`·`Edit`·`Bash`
   - 모델 정책(하이브리드): 점검가 3인 `claude-sonnet-4-6`, 구현 빌디 `claude-opus-4-8`
4. 스킬 생성 `skills/harness-check/SKILL.md` — harness-checker 6단계를 Phase로 구현:
   - frontmatter: `name:"harness-check"`·`description`(점검·보완·재점검·갱신 키워드 포함)·`triggers`·
     `argument-hint:"[대상 플러그인 경로]"`·`user-invocable:false`
   - Phase 0 입력(대상 플러그인 경로 확인) → Phase 1 구성요소 인벤토리 + **⓪ 적용 가능성 재판정**(클로니) →
     Phase 2 병렬 점검(코스티·스피디·가디를 **단일 응답 동시 Agent 호출** — 비용/성능/보안은 독립 도메인) →
     Phase 3 기록부·로드맵 통합(클로니, **영역별 질문(비용·성능·보안)**으로 상세 항목 복수 선택) →
     Phase 4 선택 항목 구현(harness-apply 스킬로 위임) → Phase 5 완료 보고
   - **점검 범위 명시**: 각 Phase에서 ✅/△ 항목만 다루고 ❌(7장)는 "대안 안내"로만 처리함을 본문에 못박음
   - 서브에이전트 호출은 `subagent_type` 네이티브 방식만 사용(수동 `.md` Read·임베딩 금지 — corp-research
     적용계획 G1: 이중 경로는 호출 충돌·중복 실행 유발)
   - MUST/MUST NOT/완료조건 명시. 점검 기록·로드맵 산출 경로: `{대상플러그인}/harness/harness-adoption-plan.md`
   - 추가 스킬 `skills/harness-apply/SKILL.md`: 점검 기록의 미적용(✅/△) 항목을 **영역별(비용·성능·보안) 질문**으로
     상세 항목 복수 선택(AskUserQuestion 영역별)하여 빌디로 구현(한국어 산출)·검증 후 기록 갱신. 기록 미존재 시 harness-check 선호출.
     독립 실행 가능하며 **harness-check Phase 4가 사용자 선택 시 harness-apply를 호출**하도록 연결
5. 커맨드 생성 `commands/harness-check.md`·`commands/harness-apply.md`: 각 frontmatter(`description`·
   `argument-hint:"[대상 플러그인 경로]"`), 본문은 대응 `${CLAUDE_PLUGIN_ROOT}/skills/<name>/SKILL.md`를 읽어
   `"$ARGUMENTS"` 대상으로 실행하라는 1줄
6. 문서 정비: `CLAUDE.md`에 `@AGENTS.md` 기재, `AGENTS.md` 하단에 하네스 포인터(트리거 규칙)+변경이력 테이블
   추가, `README.md`(기능·구성요소·설치·디렉토리 구조·알려진 제약) 작성 — corp-research README 구조 준용
7. 자체 점검(정직한 보고): 산출 플러그인을 harness-checker로 자가 점검. harness-helper는 **자체 hook 미포함
   설계**이므로 hook 기반 ✅ 항목(감사 로그·DLP·HITL 등)은 "미적용(설계상 hook 미포함)" 사유와 함께 기록하고,
   비-hook 적용 가능 항목(예: 최소 권한 — agents `tools` 스코핑)은 실제 적용 여부를 근거(파일:라인)로 확인

- 출력 디렉토리: `C:\Users\hiond\plugins\harness-helper\`(리포 루트 = 마켓플레이스 루트 = 플러그인 루트)
- 톤앤매너: 모든 산출 문서·본문은 한국어, 명사체 종결, AGENTS.md 대화 가이드(닉네임 호칭·`[역할|닉네임]` 헤더)
  준수
- 작성 규칙:
  - 플러그인 내부 파일 참조는 모두 `${CLAUDE_PLUGIN_ROOT}/...` 변수 사용(설치 캐시 복사 후에도 경로 안전)
  - agents·skill 본문은 prompt-guide 8종 섹션 구조 준용
  - 마크다운: 명사체·한 줄 120자 이내·줄바꿈 시 줄 끝 스페이스 2개

[출력]
아래 파일을 `C:\Users\hiond\plugins\harness-helper\`에 생성·갱신함 (기존 `AGENTS.md`·`references/`는 유지):

```text
harness-helper/                        # 리포 루트 = 마켓플레이스 루트 = 플러그인 루트
├── .claude-plugin/
│   ├── marketplace.json               # name: harness-helper, plugins[0].name: harness-helper
│   └── plugin.json                    # name: harness-helper (hooks 필드 없음)
├── skills/
│   ├── harness-check/SKILL.md         # 6단계 점검 워크플로우 (✅/△ 항목만)
│   └── harness-apply/SKILL.md         # 미적용 항목 복수 선택·보완 구현
├── agents/
│   ├── cost-checker.md                # 코스티 (비용 통제 점검가)
│   ├── performance-checker.md         # 스피디 (성능 통제 점검가)
│   ├── security-checker.md            # 가디 (보안 통제 점검가)
│   └── implementation-engineer.md     # 빌디 (구현 엔지니어)
├── commands/
│   ├── harness-check.md               # /harness-helper:harness-check
│   └── harness-apply.md               # /harness-helper:harness-apply
├── references/                        # (기존 유지)
├── CLAUDE.md                          # @AGENTS.md
├── AGENTS.md                          # (기존) + 하네스 포인터·변경이력
└── README.md                          # 설치·사용·디렉토리 구조
```

형식: JSON(매니페스트), Markdown+YAML frontmatter(skill·agents·command). **`hooks/` 디렉토리는 생성하지 않음.**

[제약조건]
- MUST:
  - 단일 플러그인으로 제작, 마켓플레이스·플러그인 `name`을 모두 `harness-helper`로 설정
  - **점검 범위는 harness-checker.md의 ✅(적용 가능)·△(조건부) 항목으로 한정**. ❌ 구조적 불가(7장)는 Y/N 판정
    대상에서 제외하고 7장 대안만 보고
  - 각 점검가 `[작업방법]`에 ⓪ 적용 가능성 재판정(✅/△/❌)을 Y/N 판정보다 먼저 수행하도록 명시
  - 멤버 5인을 1:1 반영(점검가 3인+구현가 1인=서브에이전트, 오케스트레이터=메인 스레드)
  - 플러그인 내부 경로는 `${CLAUDE_PLUGIN_ROOT}` 변수 사용
  - agents·skill 본문은 prompt-guide 8종 섹션 준용, 모든 산출 문서는 명사체·한국어
  - skill·agents의 산출 내용(판정표·요약·점검 기록·완료 보고)은 사용자 친화적으로 쉽게 설명하고 필요 시 비유·예시 사용
- MUST NOT:
  - **harness-helper 자체 hook 생성 금지**(`hooks/` 디렉토리·`plugin.json` `hooks` 필드 모두 두지 않음)
  - 서브에이전트 `.md` 수동 Read·임베딩 호출(호출 충돌 — corp-research 적용계획 G1), `subagent_type` 네이티브만 사용
  - 근거 없는 인용(harness-checker.md 절·항목은 실제 확인 후 인용), 문서만 작성하고 완료로 보고
  - (빌디가 대상 플러그인에 hook형 보완을 구현하는 경우) `hooks/hooks.json`·bash/sh hook 사용 금지 — hook-guide
    §3-1·§6-1 규격(`.mjs`+`plugin.json` 인라인)을 따름
- 완료조건(객관적 증거):
  - `.claude-plugin/marketplace.json`·`plugin.json`이 유효 JSON으로 파싱됨(두 `name` 모두 `harness-helper`,
    plugin.json에 `hooks` 키 부재 확인)
  - `agents/` 4파일·`skills/harness-check/SKILL.md`·`commands/harness-check.md`가 실제 생성됨
  - skill·각 점검가 본문에 "✅/△만 판정, ❌ 제외" 규칙과 ⓪ 적용 가능성 재판정 절차가 포함됨
  - harness-checker 자가 점검표에 적용 가능(✅/△) 항목의 Y/N과 근거(파일:라인), hook 기반 항목의 미적용 사유가 기재됨

[예시]

(1) `.claude-plugin/` 매니페스트 — 마켓플레이스·플러그인 `name` 모두 `harness-helper`, hooks 미포함

```json
// marketplace.json
{
  "name": "harness-helper",
  "owner": { "name": "harness-helper팀" },
  "description": "Claude Code 플러그인의 하네스 통제 점검 플러그인을 배포하는 사내 마켓플레이스",
  "plugins": [
    {
      "name": "harness-helper",
      "source": "./",
      "description": "Claude Code 플러그인을 비용·성능·보안 하네스 통제 관점에서 점검·보완",
      "category": "developer-tools",
      "keywords": ["harness", "plugin-audit", "cost", "performance", "security"]
    }
  ]
}

// plugin.json — hooks 필드 없음(자체 hook 미생성)
{
  "$schema": "https://json.schemastore.org/claude-code-plugin-manifest.json",
  "name": "harness-helper",
  "displayName": "하네스 엔지니어링 적용 도우미",
  "version": "0.1.0",
  "description": "Claude Code 플러그인을 비용·성능·보안 하네스 통제 관점에서 점검·보완하는 플러그인",
  "author": { "name": "harness-helper팀" },
  "keywords": ["harness", "plugin-audit", "cost", "performance", "security"]
}
```

(2) `agents/cost-checker.md` — frontmatter + 본문 도입부 (✅/△만 판정하는 절차 반영)

```markdown
---
name: cost-checker
description: 플러그인의 비용 하네스 통제(harness-checker 2장)를 점검하는 에이전트. 무한 루프·토큰 누수·Agent
  폭주 항목 중 적용 가능(✅/△)한 것만 Y/N 판정하고 파일:라인 근거와 6장 보완방안을 반환함.
model: claude-sonnet-4-6
tools:
  - Read
  - Grep
  - Glob
---

[역할]
당신은 LLM 에이전트 폭주 비용 통제 전문가 코스티(중성/46)입니다. 매 호출 경계에서 카운터·예산·상한으로 비용을
결정론적으로 봉쇄하고, 절감 효과를 정량화함.

[목표]
대상 플러그인의 비용 하네스 통제 중 적용 가능한 항목을 harness-checker 2장 체크리스트로 판정하여, 근거
(파일:라인)·보완방안과 함께 구조화된 마크다운으로 반환.

[작업방법]
1. 대상 플러그인의 구성요소(hooks·settings·frontmatter·MCP)를 Read/Grep으로 수집
2. ⓪ 적용 가능성 재판정: harness-checker 0-2장 규칙으로 각 항목을 ✅/△/❌로 분류
3. ✅(적용 가능)·△(조건부) 항목만 Y/N 판정 — ❌ 구조적 불가(7장)는 판정 대상에서 제외
4. 각 Y/N에 파일:라인 근거 기재(근거 없는 Y는 무효)
5. N 항목은 6장 레시피로 보완방안 도출, ❌ 항목은 7장 대안만 1줄 안내
...
```

(3) `commands/harness-check.md` — 본문 1줄 (corp-research commands/research.md 규격 준용)

```markdown
---
description: 대상 Claude Code 플러그인의 비용·성능·보안 하네스 통제 중 적용 가능 항목을 6단계로 점검·보완
argument-hint: "[대상 플러그인 경로]"
---

`${CLAUDE_PLUGIN_ROOT}/skills/harness-check/SKILL.md`를 읽고 "$ARGUMENTS" 플러그인의 하네스 통제 점검을 실행하세요.
```
