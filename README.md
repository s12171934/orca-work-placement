---
title: orca-work-placement
created: 2026-10-16
updated: 2026-10-16
tags: [domain/인공지능, area/ai에이전트]
source: ai
---

# orca-work-placement

> 격리할 작업을 Orca 워크트리에 배치하고, 그 작업을 하는 에이전트 탭의 이름·모델·목표를 정하는 규칙 스킬이다.

## 하는 일

- 격리는 Orca 가 만든 워크트리로만 한다 (`git worktree add`·작업 체크아웃의 브랜치 금지).
- 탭 이름은 맡은 일로, 모델은 지정이 없으면 Opus, 복잡하거나 조건이 미묘한 목표는 띄우기 전에 사용자와 확정한다.

## 설치

에이전트 도구의 스킬 경로에 clone 하거나 링크한다.

```bash
git clone https://github.com/s12171934/orca-work-placement.git ~/.claude/skills/orca-work-placement
# 다른 도구는 그 도구의 스킬 경로에 링크한다. 예: ln -s ~/.claude/skills/orca-work-placement ~/.codex/skills/orca-work-placement
```

- Orca CLI 와 그 `orca-cli` 스킬이 설치되어 있어야 한다. 감독형 오케스트레이션은 `orca-orchestration-rules` 스킬이 맡는다.

## 구성과 공개 면

- [[20_Agents/skills/orca-work-placement/SKILL|SKILL.md]] — 스킬 본문. 형식은 에이전트 도구가 정한다.
- 공개 면은 스킬 이름(`orca-work-placement`)뿐이다. 다른 스킬은 이 이름으로만 이 스킬을 가리킨다.
