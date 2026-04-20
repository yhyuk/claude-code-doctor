# Claude Code Doctor — 프로젝트 가이드

Claude Code 설정을 진단하는 플러그인/스킬이다. CLAUDE.md, skills, agents, settings 등을 분석하여 토큰 효율성·보안·베스트 프랙티스를 점검하고 HTML 리포트를 생성한다.

## 프로젝트 구조

```
claude-code-doctor/
  package.json                        — 단일 버전 소스 (version 필드)
  .claude-plugin/
    plugin.json                       — 플러그인 메타데이터
    marketplace.json                  — 마켓플레이스 등록 정보
  skills/
    claude-doctor/
      SKILL.md                        — 스킬 진입점 및 워크플로우 정의
      rules/
        claude-md.json                — Level 1 룰셋 (CLAUDE.md 품질 진단, 12개 룰)
        skills-agents.json            — Level 2 룰셋 (스킬/에이전트 구성 진단, 8개 룰)
        settings.json                 — Level 3 룰셋 (settings.json, permissions, memory 진단, 12개 룰)
        recommendations.json          — 프로젝트 타입별 추천 룰셋
      templates/
        report.html                   — HTML 리포트 템플릿 (CSS/JS 인라인, 외부 CDN 없음)
```

## 룰셋 JSON 구조

`rules/*.json` 파일은 아래 스키마를 따른다. 신규 룰 추가 시 모든 필드를 빠짐없이 작성한다.

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | string | 룰 식별자. 카테고리 접두사 + 3자리 숫자. 예: `CMD-001`, `SA-001`, `SET-001` |
| `name` | string | 한국어 룰 이름. 30자 이하 |
| `severity` | string | `"critical"` / `"warning"` / `"info"` 중 하나 |
| `check` | string | 검사 타입 (아래 허용 값 참조) |
| `target` | string | 검사 대상 파일 경로. `~` 또는 `{CWD}` 변수 사용 가능 |
| `source` | string | 근거 출처. `"official"` (Anthropic 공식 문서) / `"best-practice"` / `"derived"` |
| `rationale` | string | 이 룰이 필요한 이유. 공식 문서 인용 시 큰따옴표로 감쌈 |
| `advice` | string | 진단 실패 시 사용자에게 보여줄 구체적인 개선 지시. 모호한 표현 금지 |

**허용되는 `check` 값:**

- `file_exists` — 파일 존재 여부 (Glob)
- `line_count` — 파일 줄 수 (`wc -l`)
- `has_headers` — 마크다운 `#`/`##` 헤더 존재 여부 (Grep)
- `duplicate_check` — 글로벌-프로젝트 CLAUDE.md 간 유사 블록 탐지
- `vague_terms` — 모호한 표현 탐지 (Grep)
- `token_estimate` — 토큰 수 추정 (`wc -w` × 1.3)
- `code_block_ratio` — 코드 블록 비율 (Grep으로 ``` 카운트)
- `json_valid` — JSON 유효성 검사 (`python3 -c "import json; json.load(open(...))"`)
- `pattern_match` — 특정 패턴 존재 여부 (Grep)
- `count_items` — 디렉토리 내 항목 수 (Glob)
- `dangerous_patterns` — 위험 패턴 탐지 (`rm -rf`, `sudo`, `curl.*|.*sh`)
- `consistency_check` — 내부 충돌 규칙 탐지

## 코딩 컨벤션

### 출력 언어
- 모든 사용자 대면 출력(터미널 요약, HTML 리포트 내 텍스트, advice 문구)은 반드시 한국어로 작성한다.
- 영어는 파일 경로, 코드 식별자, 기술 용어에 한해 허용한다.

### JSON 룰셋 형식
- 들여쓰기: 공백 2칸
- 배열 내 룰 순서: `id` 오름차순 (CMD-001, CMD-002, ...)
- `advice` 필드: 모호한 표현(`적절히`, `필요시`, `상황에 따라`) 금지. 구체적인 행동 지시문으로 작성한다.
  - 나쁜 예: "적절히 설정을 최적화하세요."
  - 좋은 예: "글로벌 CLAUDE.md에 `allow_edit: true`를 추가하세요."
- 새 룰의 `severity` 기준:
  - `critical`: 없으면 Claude Code가 정상 동작하지 않는 설정 누락
  - `warning`: 성능·보안·팀 협업에 영향을 주는 비권장 설정
  - `info`: 개선하면 좋지만 즉각적인 영향은 없는 항목

### HTML 템플릿 플레이스홀더 규칙
- 플레이스홀더 형식: `{{UPPER_SNAKE_CASE}}`
- 신규 플레이스홀더 추가 시 반드시 `SKILL.md`의 "플레이스홀더 목록" 섹션에도 동시에 추가한다.
- HTML 생성 시 플레이스홀더를 빠짐없이 치환한다. 미치환 플레이스홀더가 HTML에 남아 있으면 안 된다.
- 현재 정의된 플레이스홀더:
  `{{TIMESTAMP}}`, `{{PROJECT_PATH}}`, `{{PROJECT_TYPE}}`, `{{GRADE}}`, `{{SCORE}}`,
  `{{GRADE_COLOR}}`, `{{CRITICAL_COUNT}}`, `{{WARNING_COUNT}}`, `{{INFO_COUNT}}`,
  `{{PASS_COUNT}}`, `{{TOTAL_COUNT}}`, `{{CATEGORY_CARDS}}`, `{{TOP_ISSUES}}`,
  `{{RECOMMENDATIONS}}`, `{{DETAIL_SECTIONS}}`, `{{DIAGNOSIS_LEVEL}}`, `{{SCORE_DEG}}`,
  `{{VERSION}}`, `{{TREND_SECTION}}`

## 빌드 및 검증

이 프로젝트는 빌드 단계가 없다 (순수 JSON + HTML + Markdown 구성).

**JSON 유효성 검사** (룰셋 파일 수정 후 반드시 실행):

```bash
# 모든 룰셋 파일 한 번에 검사
for f in skills/claude-doctor/rules/*.json .claude-plugin/plugin.json .claude-plugin/marketplace.json; do
  python3 -c "import json, sys; json.load(open('$f')); print('OK:', '$f')"
done
```

**수동 동작 확인**: 수정 후 `/claude-doctor` 스킬을 직접 실행하여 HTML 리포트가 정상 생성되는지 확인한다.

## 버전 관리

- `package.json`의 `version` 필드가 단일 버전 소스(Single Source of Truth)이다.
- `.claude-plugin/plugin.json`의 `version`은 `package.json`과 항상 동일해야 한다.
- 버전 업 시 두 파일을 동시에 수정하고 하나의 커밋으로 반영한다.
- 커밋 메시지 규칙 (Conventional Commits):
  - `feat:` 새 룰 추가, 새 스킬 기능
  - `fix:` 룰 판정 오류 수정, HTML 치환 버그
  - `docs:` SKILL.md, CLAUDE.md 등 문서 변경
  - `chore:` plugin.json, package.json 등 메타데이터 변경
- `version` 부 버전(minor) 업 기준: 룰셋 추가/제거. 패치(patch) 업 기준: 기존 룰 수정, 버그 수정.
