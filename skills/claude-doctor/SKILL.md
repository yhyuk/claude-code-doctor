---
name: claude-doctor
description: Claude Code 설정 진단 및 맞춤 추천 도구. 설정 파일을 분석하여 토큰 효율성, 보안, 베스트 프랙티스를 진단하고 HTML 리포트를 생성합니다.
allowed-tools: Read, Write, Glob, Grep, Bash, AskUserQuestion
---

# Claude Code Doctor

> "당신은 Claude Code를 제대로 사용하고 있는가?"

사용자의 Claude Code 설정을 진단하고, 프로젝트 타입에 맞는 추천을 제공하는 도구.

## When to Use

- 사용자가 `/diagnose`, "진단", "설정 점검", "checkup" 등을 요청할 때
- Claude Code 설정 최적화가 필요할 때
- 새 프로젝트에서 CLAUDE.md 가이드가 필요할 때

## Workflow

### Step 1: 진단 레벨 선택

AskUserQuestion으로 진단 레벨을 선택받는다:

- **Level 1: CLAUDE.md 진단** — 글로벌/프로젝트 CLAUDE.md 품질 + 프로젝트 타입별 추천
- **Level 2: 스킬/에이전트 진단** — Level 1 + 커스텀 스킬, 에이전트, 커맨드 구성 분석
- **Level 3: 종합 진단** — Level 1 + 2 + settings, permissions, memory, 플러그인 전체

### Step 2: 프로젝트 타입 감지

현재 작업 디렉토리에서 파일을 확인하여 프로젝트 타입을 자동 감지한다:

| 파일 | 타입 |
|------|------|
| next.config.* | Next.js |
| nuxt.config.* | Nuxt.js |
| vite.config.* | Vite/React |
| angular.json | Angular |
| package.json (기본) | Node.js |
| build.gradle / build.gradle.kts | Java/Kotlin (Spring) |
| pom.xml | Java (Maven) |
| Cargo.toml | Rust |
| go.mod | Go |
| requirements.txt / pyproject.toml | Python |
| Gemfile | Ruby |
| *.sln / *.csproj | .NET/C# |
| 감지 불가 | Generic |

Spring 프로젝트의 경우 build.gradle/pom.xml 내 `org.springframework` 문자열도 확인한다.

### Step 3: 설정 파일 수집 및 진단

선택된 Level에 따라 `rules/` 디렉토리의 룰셋 JSON을 읽고 진단을 수행한다.

**룰셋 파일 경로** (이 SKILL.md와 같은 디렉토리 기준):
- `rules/claude-md.json` — Level 1 룰셋
- `rules/skills-agents.json` — Level 2 룰셋
- `rules/settings.json` — Level 3 룰셋
- `rules/recommendations.json` — 프로젝트 타입별 추천 룰셋

**진단 절차**:

1. 룰셋 JSON 파일을 Read로 읽는다
2. 각 룰의 `check` 타입에 따라 검사를 수행한다:
   - `file_exists`: 대상 파일 존재 여부 (Glob 사용)
   - `line_count`: 파일 줄 수 확인 (Bash: `wc -l`)
   - `has_headers`: 마크다운 헤더(`#`, `##`) 존재 여부 (Grep 사용)
   - `duplicate_check`: 글로벌-프로젝트 CLAUDE.md 간 유사 블록 탐지 (Read로 양쪽 읽고 비교)
   - `vague_terms`: 모호한 표현 탐지 (Grep: "적절히|필요시|상황에 따라|가능하면|경우에 따라")
   - `token_estimate`: 토큰 수 추정 (Bash: `wc -w` 후 x1.3)
   - `code_block_ratio`: 코드 블록 비율 (Grep으로 ``` 블록 카운트)
   - `json_valid`: JSON 파싱 유효성 (Bash: `python3 -c "import json; json.load(open('...'))"`)
   - `pattern_match`: 특정 패턴 존재 여부 (Grep 사용)
   - `count_items`: 디렉토리 내 항목 수 (Glob 사용)
   - `dangerous_patterns`: 위험 패턴 탐지 (Grep: "rm -rf|sudo|curl.*\\|.*sh")
3. 각 항목에 pass/warning/fail 판정을 내린다
4. 추천 룰셋을 읽고 프로젝트 타입에 맞는 누락 섹션을 식별한다

### Step 4: 등급 산출

100점 기준에서 감점 방식:
- Critical fail: -15점
- Warning fail: -5점
- Info fail: -2점
- 추천 항목은 감점하지 않음 (별도 표시)

| 등급 | 점수 | 의미 |
|------|------|------|
| A | 90-100 | 최적화된 설정 |
| B | 75-89 | 양호 |
| C | 60-74 | 보통, 개선 권장 |
| D | 40-59 | 미흡 |
| F | 0-39 | 기본 설정 누락 다수 |

### Step 5: HTML 리포트 생성

`templates/report.html` 템플릿을 Read로 읽고, 플레이스홀더를 진단 결과로 치환하여 HTML 파일을 생성한다.

**플레이스홀더 목록**:
- `{{TIMESTAMP}}` — 진단 일시
- `{{PROJECT_PATH}}` — 프로젝트 경로
- `{{PROJECT_TYPE}}` — 감지된 프로젝트 타입
- `{{GRADE}}` — 종합 등급 (A~F)
- `{{SCORE}}` — 종합 점수
- `{{GRADE_COLOR}}` — 등급별 색상 (A:#22c55e, B:#84cc16, C:#eab308, D:#f97316, F:#ef4444)
- `{{CRITICAL_COUNT}}` — Critical 개수
- `{{WARNING_COUNT}}` — Warning 개수
- `{{INFO_COUNT}}` — Info 개수
- `{{PASS_COUNT}}` — Pass 개수
- `{{TOTAL_COUNT}}` — 전체 항목 수
- `{{CATEGORY_CARDS}}` — 카테고리별 카드 HTML
- `{{TOP_ISSUES}}` — 주요 개선사항 HTML
- `{{RECOMMENDATIONS}}` — 추천사항 HTML
- `{{DETAIL_SECTIONS}}` — 상세 진단 결과 HTML
- `{{DIAGNOSIS_LEVEL}}` — 진단 레벨
- `{{SCORE_DEG}}` — 점수를 각도로 변환한 값 (SCORE * 3.6, 예: 73점 → 262.8)

**카테고리 카드 HTML 생성 형식**:
```html
<div class="card">
  <div class="card-title">CLAUDE.md</div>
  <div class="card-score">{pass}/{total}</div>
  <div class="card-bar"><div class="bar-fill" style="width:{percent}%"></div></div>
</div>
```

**주요 개선사항 HTML 생성 형식** (Critical과 Warning만):
```html
<div class="issue issue-{severity}">
  <span class="badge badge-{severity}">{SEVERITY}</span>
  <span class="issue-text">{항목명} — {조언}</span>
</div>
```

**추천사항 HTML 생성 형식**:
```html
<details class="recommendation">
  <summary>{추천 섹션명} — {설명}</summary>
  <pre><code>{예시 코드}</code></pre>
</details>
```

**상세 결과 HTML 생성 형식**:
```html
<details class="detail-section">
  <summary>{카테고리명} ({pass}/{total} Pass)</summary>
  <div class="detail-item detail-{result}">
    <span class="detail-id">{ID}</span>
    <span class="detail-name">{항목명}</span>
    <span class="detail-result">{PASS|WARNING|FAIL}</span>
  </div>
  <!-- WARNING/FAIL 항목에는 추가 정보 표시 -->
  <div class="detail-info">
    현재: {현재값} | 권장: {권장값}<br>
    조언: {advice}
  </div>
</details>
```

치환 완료된 HTML을 `~/.claude/reports/diagnostic-{YYYYMMDD-HHmmss}.html`에 Write로 저장한다.

### Step 6: 결과 출력

1. Bash로 `open {HTML파일경로}` 실행하여 브라우저 오픈 (macOS)
   - Linux의 경우 `xdg-open` 사용
2. 터미널에 10줄 이내 요약 출력:

```
=== Claude Code Doctor v1.0.0 ===
프로젝트: {프로젝트타입} (감지됨)
종합 등급: {등급} ({점수}점)
Critical: {n} | Warning: {n} | Info: {n}
추천: {n}개 ({추천목록})

리포트: ~/.claude/reports/diagnostic-{timestamp}.html
```

## Important

- 민감 정보 마스킹: API 키, 토큰, 패스워드 패턴은 `***MASKED***`로 치환
  - 패턴: `(sk-|api[_-]?key|token|password|secret)[=:]\s*\S+`
- 파일 미존재 시 에러 대신 "파일 없음" 판정으로 처리
- 1000줄 초과 CLAUDE.md는 처음 500줄 + 마지막 100줄만 샘플링 분석
- HTML은 외부 CDN 없이 단일 파일로 동작 (CSS/JS 인라인)
- 모든 출력은 한국어로
