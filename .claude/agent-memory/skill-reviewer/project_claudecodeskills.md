---
name: ClaudeCodeSkills project context
description: Purpose and structure of the ClaudeCodeSkills repository
type: project
---

This repository is a collection of Claude Code skill configurations (agents). Skills are stored as Markdown files with YAML frontmatter containing `name` and `description` fields, followed by a system prompt body.

**Current skill inventory (as of 2026-03-15):**
- `shopify/v1.md` — `shopify-rest-to-graphql-go`: Converts Shopify REST Admin API code to GraphQL in Go codebases. ~1400 lines, written primarily in Vietnamese.
- `shopify/v2.md` — `skill-creator`: A meta-skill for creating, testing, and iterating on other skills. ~486 lines, written in English.

**Naming convention in use:** Files are versioned (`v1.md`, `v2.md`) inside domain subdirectories (`shopify/`). The `name` field in frontmatter is the actual identifier used for triggering.

**Why:** The user is building and refining skills for Claude Code, likely for use in production Shopify app development workflows.

**How to apply:** When reviewing or suggesting new skills, follow the existing structure — Markdown with YAML frontmatter, body as the system prompt, organized into domain subdirectories with versioned filenames.
