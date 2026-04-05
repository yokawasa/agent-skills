# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a repository of [Agent Skills](https://agentskills.io) — prompt-based skill definitions for Claude Code. Skills are loaded by Claude Code at runtime to provide domain-specific knowledge and behaviors.

## Skill Structure

Each skill lives in its own directory and follows this layout:

```
<skill-name>/
  SKILL.md              # Main skill definition (frontmatter + instructions)
  references/           # Optional reference documents loaded on demand
    *.md
```

### SKILL.md Format

Every skill must have a YAML frontmatter block followed by the skill body:

```markdown
---
name: <skill-name>
description: >
  <When to trigger this skill — written as trigger conditions, not a summary.
   This text is used by Claude Code to decide when to invoke the skill.>
---

# Skill Title

...skill instructions...
```

The `description` field is critical: it should describe **when to use** the skill (trigger conditions), not what the skill does. This is how Claude Code decides whether to load the skill for a given user request.

### Reference Files

For skills with large bodies of knowledge, split content into `references/*.md` files. The `SKILL.md` should include a table listing each reference file, its contents, and when to load it — so Claude loads only what's relevant rather than everything upfront.

## Adding a New Skill

1. Create a new directory: `<skill-name>/`
2. Write `SKILL.md` with frontmatter (`name`, `description`) and skill body
3. If the knowledge base is large, split into `references/*.md` and add a reference table in `SKILL.md`
4. Add an entry to the `README.md` Skills table

## Conventions

- Skill names use kebab-case (`digital-agency-api-guidebook`)
- Reference filenames are short, lowercase, underscore-separated (`uri_request.md`)
- Normative strength in skill instructions follows the pattern: **[MUST]** / **[SHOULD]** / **[MAY]**
- Skills are language-agnostic at the file level; the skill body may be in Japanese or English depending on the target audience
