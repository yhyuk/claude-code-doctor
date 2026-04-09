# Claude Code Doctor

> 당신은 Claude Code를 제대로 사용하고 있는가?

Claude Code 설정 파일(CLAUDE.md, settings, skills, agents 등)을 분석하여 개선점을 진단하고, 시각적 HTML 리포트로 결과를 제공하는 진단 스킬입니다.

[English](./README.md)

## 주요 기능

- **3단계 진단** — 원하는 깊이를 선택하여 분석
- **프로젝트 타입 감지** — 기술 스택을 자동 감지하여 맞춤 추천 제공
- **시각적 HTML 리포트** — 등급, 차트, 상세 결과를 브라우저에서 확인
- **다크/라이트 모드** — 시스템 설정에 따라 자동 전환

## 진단 레벨

| Level | 범위 | 항목 수 |
|-------|------|---------|
| 1 | CLAUDE.md | 파일 품질, 토큰 효율성, 구조, 중복 (10개 룰) |
| 2 | 스킬/에이전트 | Level 1 + 커스텀 스킬, 에이전트, 커맨드 검증 (+7개 룰) |
| 3 | 종합 | Level 2 + settings.json, 권한, 메모리, 플러그인 (+10개 룰) |

## 등급 기준

| 등급 | 점수 | 의미 |
|------|------|------|
| A | 90-100 | 최적화된 설정 |
| B | 75-89 | 양호, 소소한 개선 가능 |
| C | 60-74 | 보통, 개선 권장 |
| D | 40-59 | 미흡, 개선 필요 |
| F | 0-39 | 기본 설정 누락 다수 |

## 지원 프로젝트 타입

Next.js, React (Vite), Node.js, Spring Boot, Python, Rust, Go, Generic

## 설치

### 스킬로 등록 (심볼릭 링크)

```bash
ln -s /path/to/claude-code-doctor/skills/claude-doctor ~/.claude/skills/claude-doctor
```

### 플러그인으로 설치 (준비 중)

```bash
# Claude Code 플러그인 마켓플레이스를 통해 제공 예정
```

## 사용법

Claude Code 세션에서:

```
/claude-doctor
```

1. 진단 레벨 선택 (1, 2, 3)
2. 분석 완료 대기
3. HTML 리포트가 브라우저에서 자동 오픈
4. 터미널에 요약 출력

## 진단 항목

### Level 1: CLAUDE.md

| ID | 항목 | 심각도 |
|----|------|--------|
| CMD-001 | 글로벌 CLAUDE.md 존재 | Critical |
| CMD-002 | 프로젝트 CLAUDE.md 존재 | Warning |
| CMD-003 | CLAUDE.md 크기 (500줄 이하) | Warning |
| CMD-004 | 섹션 구조화 여부 | Warning |
| CMD-005 | 글로벌-프로젝트 중복 | Warning |
| CMD-006 | 모호한 지시사항 | Info |
| CMD-007 | 토큰 효율성 추정 | Info |
| CMD-008 | 코드 블록 비중 (50% 이하) | Info |
| CMD-009 | 외부 파일 참조 활용 | Info |
| CMD-010 | .claudeignore 존재 | Warning |

### Level 2: 스킬/에이전트

| ID | 항목 | 심각도 |
|----|------|--------|
| SKA-001 | 커스텀 스킬 존재 | Info |
| SKA-002 | 커스텀 에이전트 존재 | Info |
| SKA-003 | 커스텀 커맨드 존재 | Info |
| SKA-004 | SKILL.md frontmatter 유효성 | Warning |
| SKA-005 | 에이전트 프롬프트 크기 (500줄 이하) | Warning |
| SKA-006 | allowed-tools 과다 허용 | Warning |
| SKA-007 | 학습된 스킬 활용 | Info |

### Level 3: Settings

| ID | 항목 | 심각도 |
|----|------|--------|
| SET-001 | settings.json JSON 유효성 | Critical |
| SET-002 | 모델 설정 확인 | Info |
| SET-003 | permissions.allow 과다 | Warning |
| SET-004 | 위험 명령어 허용 여부 | Critical |
| SET-005 | deny 목록 설정 | Info |
| SET-006 | 플러그인 설치 상태 | Info |
| SET-007 | memory 활용도 | Info |
| SET-008 | 프로젝트별 설정 분리 | Info |
| SET-009 | statusLine 설정 | Info |
| SET-010 | hooks 설정 여부 | Info |

## 라이선스

MIT
