# Claude Code Doctor

> Are you using Claude Code properly?

A diagnostic skill for Claude Code that analyzes your configuration files (CLAUDE.md, settings, skills, agents, etc.) and provides actionable recommendations with a visual HTML report.

[한국어](./README.ko.md)

## Features

- **3-Level Diagnosis** — Choose the depth of analysis
- **Project Type Detection** — Auto-detects your tech stack and provides tailored recommendations
- **Visual HTML Report** — Opens automatically in your browser with grades, charts, and detailed results
- **Dark/Light Mode** — Follows your system preference

## Diagnosis Levels

| Level | Scope | Items |
|-------|-------|-------|
| 1 | CLAUDE.md | File quality, token efficiency, structure, duplicates (10 rules) |
| 2 | Skills & Agents | Level 1 + custom skills, agents, commands validation (+ 7 rules) |
| 3 | Comprehensive | Level 2 + settings.json, permissions, memory, plugins (+ 10 rules) |

## Grading

| Grade | Score | Meaning |
|-------|-------|---------|
| A | 90-100 | Optimized configuration |
| B | 75-89 | Good, minor improvements possible |
| C | 60-74 | Average, improvements recommended |
| D | 40-59 | Below average, needs attention |
| F | 0-39 | Missing basic configurations |

## Supported Project Types

Next.js, React (Vite), Node.js, Spring Boot, Python, Rust, Go, and Generic fallback.

## Installation

### As a Skill (symlink)

```bash
ln -s /path/to/claude-code-doctor/skills/claude-doctor ~/.claude/skills/claude-doctor
```

### As a Plugin (coming soon)

```bash
# Will be available via Claude Code plugin marketplace
```

## Usage

In a Claude Code session:

```
/claude-doctor
```

1. Select diagnosis level (1, 2, or 3)
2. Wait for analysis to complete
3. HTML report opens automatically in your browser
4. Terminal shows a brief summary

## Diagnosis Rules

### Level 1: CLAUDE.md

| ID | Rule | Severity |
|----|------|----------|
| CMD-001 | Global CLAUDE.md exists | Critical |
| CMD-002 | Project CLAUDE.md exists | Warning |
| CMD-003 | CLAUDE.md size (< 500 lines) | Warning |
| CMD-004 | Section structure (headers) | Warning |
| CMD-005 | Global-project duplication | Warning |
| CMD-006 | Vague instructions | Info |
| CMD-007 | Token efficiency estimate | Info |
| CMD-008 | Code block ratio (< 50%) | Info |
| CMD-009 | External file references | Info |
| CMD-010 | .claudeignore exists | Warning |

### Level 2: Skills & Agents

| ID | Rule | Severity |
|----|------|----------|
| SKA-001 | Custom skills exist | Info |
| SKA-002 | Custom agents exist | Info |
| SKA-003 | Custom commands exist | Info |
| SKA-004 | SKILL.md frontmatter validity | Warning |
| SKA-005 | Agent prompt size (< 500 lines) | Warning |
| SKA-006 | Excessive allowed-tools | Warning |
| SKA-007 | Learned skills usage | Info |

### Level 3: Settings

| ID | Rule | Severity |
|----|------|----------|
| SET-001 | settings.json validity | Critical |
| SET-002 | Model configuration | Info |
| SET-003 | Excessive permissions.allow | Warning |
| SET-004 | Dangerous command patterns | Critical |
| SET-005 | Deny list configuration | Info |
| SET-006 | Plugin installation status | Info |
| SET-007 | Memory utilization | Info |
| SET-008 | Per-project settings | Info |
| SET-009 | StatusLine configuration | Info |
| SET-010 | Hooks configuration | Info |

## Project Structure

```
claude-code-doctor/
  package.json
  .claude-plugin/
    plugin.json
    marketplace.json
  skills/
    claude-doctor/
      SKILL.md
      rules/
        claude-md.json
        skills-agents.json
        settings.json
        recommendations.json
      templates/
        report.html
```

## License

MIT
