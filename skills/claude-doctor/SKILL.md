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

사용자가 `--quick` 또는 "빠른 진단"을 요청하면 Quick 모드로 동작한다:
- HTML 리포트를 생성하지 않고 터미널에 요약만 출력
- JSON 리포트는 동일하게 저장
- Auto-fix 단계를 건너뜀
- 트렌드 비교는 터미널에 한 줄로 표시 (예: "이전 대비 +5점 개선 (B → A)")

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
   - `cross_skill_tools_check`: 여러 스킬의 SKILL.md를 읽어 allowed-tools 간 중복/충돌 비교
   - `settings_conflict_check`: 4단계 settings 파일을 모두 읽어 동일 키 중복 감지
   - `mcp_server_check`: settings.json의 mcpServers 키 존재 및 구성 유효성 확인
3. 각 항목에 pass/warning/fail 판정을 내린다
4. 추천 룰셋을 읽고 프로젝트 타입에 맞는 누락 섹션을 식별한다

### Step 4: 등급 산출

가중치 기반 채점 방식:

1. 해당 Level에 포함된 모든 룰의 weight 합계를 구한다 (= 만점)
2. fail 항목의 weight를 합산한다 (= 감점)
3. 점수 = ((만점 - 감점) / 만점) * 100, 최저 0점 클램핑
4. 카테고리별 점수를 각각 산출한다:
   - CLAUDE.md 카테고리: CMD-* 룰들의 가중치 기반 점수
   - 스킬/에이전트 카테고리: SKA-* 룰들의 가중치 기반 점수
   - Settings 카테고리: SET-* 룰들의 가중치 기반 점수
5. 종합 점수 = 각 카테고리 점수의 가중 평균 (해당 Level에 포함된 카테고리만)

가중 평균 비율:
- Level 1: CLAUDE.md 100%
- Level 2: CLAUDE.md 60%, 스킬/에이전트 40%
- Level 3: CLAUDE.md 40%, 스킬/에이전트 30%, Settings 30%

| 등급 | 점수 | 의미 |
|------|------|------|
| A | 90-100 | 최적화된 설정 |
| B | 75-89 | 양호 |
| C | 60-74 | 보통, 개선 권장 |
| D | 40-59 | 미흡 |
| F | 0-39 | 기본 설정 누락 다수 |

### Step 5: HTML 리포트 생성

`templates/report.html` 템플릿을 Read로 읽고, 플레이스홀더를 진단 결과로 치환하여 HTML 파일을 생성한다.

**사전 준비**:
1. 이 SKILL.md와 같은 디렉토리 기준으로 `../../package.json`을 Read로 읽어 `version` 값을 추출한다 ({{VERSION}} 치환에 사용)
2. `~/.claude/reports/` 디렉토리에서 동일 프로젝트의 이전 JSON 리포트를 Glob(`~/.claude/reports/diagnostic-*.json`)으로 검색한다 ({{TREND_SECTION}} 치환에 사용)

**치환 절차**:
모든 `{{플레이스홀더}}`를 아래 목록의 실제 값으로 빠짐없이 치환한다. 치환되지 않은 플레이스홀더가 남아있으면 안 된다.

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
- `{{VERSION}}` — package.json의 version 값
- `{{TREND_SECTION}}` — 이전 리포트와의 비교 트렌드 HTML (이전 리포트 없으면 빈 문자열)

**트렌드 HTML 생성 형식**:
```html
<div class="section">
  <div class="section-title">
    <span class="section-title-icon">&#128200;</span> 점수 변화 추이
  </div>
  <div class="trend-card">
    <div class="trend-compare">
      <span class="trend-prev">이전: {이전점수}점 ({이전등급})</span>
      <span class="trend-arrow">{화살표}</span>
      <span class="trend-current">현재: {현재점수}점 ({현재등급})</span>
    </div>
    <div class="trend-delta trend-{up|down|same}">
      {+n점 개선 | -n점 하락 | 변화 없음}
    </div>
    <div class="trend-changes">
      <div class="trend-improved">개선: {새로 pass된 항목 목록}</div>
      <div class="trend-regressed">악화: {새로 fail된 항목 목록}</div>
    </div>
  </div>
</div>
```

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

**JSON 리포트 저장**:

HTML과 동일한 타임스탬프로 JSON 리포트도 생성하여 `~/.claude/reports/diagnostic-{YYYYMMDD-HHmmss}.json`에 Write로 저장한다.
version 값은 사전 준비 단계에서 추출한 package.json의 version을 사용한다.

JSON 구조:
```json
{
  "version": "2.0.0",
  "timestamp": "2026-04-10 11:37:20",
  "projectPath": "/path/to/project",
  "projectType": "detected-type",
  "diagnosisLevel": 1,
  "score": 85,
  "grade": "B",
  "categories": {
    "claude-md": { "score": 90, "pass": 8, "total": 12 },
    "skills-agents": { "score": 80, "pass": 6, "total": 8 },
    "settings": { "score": 75, "pass": 9, "total": 12 }
  },
  "results": [
    { "id": "CMD-001", "name": "글로벌 CLAUDE.md 존재", "severity": "critical", "result": "pass", "detail": "파일 존재 확인됨" }
  ],
  "recommendations": [
    { "id": "REC-NEXTJS-001", "section": "컴포넌트 규칙", "missing": true }
  ]
}
```

**히스토리 비교**:

Step 5 사전 준비에서 검색한 이전 JSON 리포트가 있을 경우:
1. 가장 최근 JSON 리포트를 Read로 읽는다
2. `projectPath`가 동일한 리포트만 비교 대상으로 사용한다
3. 점수 변화를 비교한다:
   - 점수 상승: "이전 대비 +{n}점 개선" 표시
   - 점수 하락: "이전 대비 -{n}점 하락" 경고
   - results 배열을 비교하여 새로 pass/fail된 항목 변화 목록 생성
4. 비교 결과로 트렌드 HTML을 생성하여 `{{TREND_SECTION}}`에 치환한다
5. 이전 리포트가 없으면 `{{TREND_SECTION}}`을 빈 문자열로 치환한다

### Step 6: 결과 출력

1. Bash로 `open {HTML파일경로}` 실행하여 브라우저 오픈 (macOS)
   - Linux의 경우 `xdg-open` 사용
2. 터미널에 10줄 이내 요약 출력:

```
=== Claude Code Doctor v{{VERSION}} ===
프로젝트: {프로젝트타입} (감지됨)
종합 등급: {등급} ({점수}점)
Critical: {n} | Warning: {n} | Info: {n}
추천: {n}개 ({추천목록})

리포트: ~/.claude/reports/diagnostic-{timestamp}.html
```

### Step 7: 자동 수정 (Auto-fix)

Step 6 결과 출력 후, Warning 또는 Critical 항목이 1개 이상이고 `--quick` 모드가 아닌 경우에만 진행한다.

**7-1. 자동 수정 가능 항목 확인**

진단 결과에서 아래 ID에 해당하는 Warning/Critical 항목만 추출한다:

| ID | 수정 내용 | 방법 |
|----|-----------|------|
| CMD-004 | CLAUDE.md 구조화 부족 | 내용 분석 후 적절한 헤더(`##`) 삽입 |
| CMD-006 | 모호한 표현 | 모호한 표현을 하이라이트하고 구체적 대안 제시 (사용자 확인 후 치환) |
| CMD-008 | 코드 블록 비중 초과 | 코드 블록을 `.claude/rules/` 파일로 분리 제안 |
| SKA-003 | 커맨드→스킬 마이그레이션 | `commands/` 내 `.md` 파일을 `skills/` 구조로 변환 |

자동 수정 불가 항목(CMD-001 파일 생성, SET-004 위험 명령 등)은 Step 7을 건너뛰고 수동 수정 안내만 출력한다.

**7-2. 사용자 확인**

자동 수정 가능 항목이 1개 이상이면 AskUserQuestion으로 묻는다:

```
자동 수정 가능한 항목이 {n}개 있습니다. 자동 수정을 진행하시겠습니까? (y/n)
- {ID}: {항목명}
- ...
```

사용자가 `n`을 선택하면 자동 수정을 건너뛴다.

**7-3. 자동 수정 수행**

사용자가 `y`를 선택하면 항목별로 아래 절차를 수행한다:

1. **수정 전 원본 백업**: Bash로 `cp {대상파일} {대상파일}.bak` 실행
2. **항목별 수정**:

   - **CMD-004 (구조화 부족)**:
     - Read로 CLAUDE.md 전체 내용을 읽는다
     - 헤더(`#`, `##`)가 없는 주요 섹션을 식별한다
     - Edit으로 적절한 위치에 `##` 헤더를 삽입한다

   - **CMD-006 (모호한 표현)**:
     - Grep으로 모호한 표현(`적절히|필요시|상황에 따라|가능하면|경우에 따라`) 위치를 찾는다
     - 각 표현에 대해 AskUserQuestion으로 구체적 대안을 제시하고 사용자 확인을 받는다
     - 확인된 항목만 Edit으로 치환한다

   - **CMD-008 (코드 블록 비중 초과)**:
     - Read로 CLAUDE.md를 읽고 코드 블록(` ``` `)을 추출한다
     - Write로 각 코드 블록을 `.claude/rules/{적절한이름}.md`로 분리한다
     - Edit으로 CLAUDE.md 내 해당 블록을 `rules/{파일명}` 참조 링크로 교체한다

   - **SKA-003 (커맨드→스킬 마이그레이션)**:
     - Glob으로 `commands/*.md` 파일 목록을 수집한다
     - 각 `.md` 파일을 Read로 읽고 `skills/{이름}/SKILL.md` 구조로 Write한다
     - 원본 파일 삭제는 수행하지 않고 마이그레이션 완료 안내만 출력한다

3. **수정 완료 후 요약 출력**:

```
=== Auto-fix 완료 ===
수정된 항목: {n}개
- CMD-004: CLAUDE.md 헤더 {n}개 삽입 완료
- CMD-006: 모호한 표현 {n}개 치환 완료
- CMD-008: 코드 블록 {n}개를 rules/ 로 분리 완료
- SKA-003: commands/ → skills/ 마이그레이션 {n}개 완료

백업 파일: {대상파일}.bak
```

**주의사항**:
- `--quick` 모드에서는 Step 7 전체를 건너뜀
- 자동 수정 불가 항목(CMD-001, SET-004 등)은 수동 수정 안내 메시지만 출력
- 수정 중 오류 발생 시 백업에서 복원: `cp {대상파일}.bak {대상파일}`

## Important

- 민감 정보 마스킹: API 키, 토큰, 패스워드 패턴은 `***MASKED***`로 치환
  - 패턴: `(sk-|api[_-]?key|token|password|secret)[=:]\s*\S+`
- 파일 미존재 시 에러 대신 "파일 없음" 판정으로 처리
- 1000줄 초과 CLAUDE.md는 처음 500줄 + 마지막 100줄만 샘플링 분석
- HTML은 외부 CDN 없이 단일 파일로 동작 (CSS/JS 인라인)
- 모든 출력은 한국어로
