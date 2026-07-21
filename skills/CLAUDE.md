# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A collection of Claude Code skills. Each skill lives in its own subdirectory and contains a single `SKILL.md` file. Skills are invoked via the `Skill` tool in Claude Code conversations.

## Skill File Format

Every `SKILL.md` must start with YAML frontmatter followed by markdown content:

```markdown
---
name: skill-name
description: One-line description — used to decide when to trigger the skill.
version: 1.0.0
---

# Skill Title

Prose and step-by-step instructions Claude should follow when the skill is invoked.
```

The `description` field is critical — it's what Claude reads to decide whether to invoke the skill.

## Adding a New Skill

1. Create a new directory named after the skill (kebab-case).
2. Add a `SKILL.md` with the frontmatter above.
3. Register it in the system prompt or settings so Claude Code knows it exists.

## Skill Content Guidelines

- Write steps as numbered `###` subheadings so Claude can follow them procedurally.
- Be explicit about edge cases and fallback behavior within the skill content.
- Skills reference other Claude tools (Bash, Read, Edit, etc.) and can invoke sub-agents.
