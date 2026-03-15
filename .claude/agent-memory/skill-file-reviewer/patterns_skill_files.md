---
name: Skill file review patterns
description: Common patterns, issues, and effective techniques discovered across skill file reviews in ClaudeCodeSkills repo
type: reference
---

## Effective Techniques Found
- Detailed API mapping tables (REST to GraphQL) with gotcha notes per endpoint are highly effective for domain-specific migrations
- Parallel agent strategy with explicit prompt templates helps orchestrate multi-service work
- Comprehensive checklists (Section 8 style) serve as both agent guidance and human verification gates
- Concrete code examples (Go patterns) anchored to specific domain types make instructions unambiguous
- Numbered gotchas with rationale (not just rules) help agents understand *why* and handle edge cases
- Error preservation checklists that specify `errors.Is` parity between old and new implementations

## Common Issues to Watch For
- Mixed-language content (Vietnamese + English) reduces accessibility and creates inconsistent tone
- Overly long skill files (1000+ lines) risk exceeding context limits and diluting priority instructions — inline code (HTML templates, full implementations) is the usual culprit
- Hardcoded paths (~/Desktop/shopify/...) make skills non-portable across users/machines
- Section cross-references by number (e.g., "Section 1.2") that don't match actual section numbering — this recurs across revisions
- Missing frontmatter fields or incomplete description keywords for skill matching
- Report templates with example data can confuse agents into using the example data literally
- Go raw string literals containing backticks cause compile errors — watch for ` + "`" patterns in inline Go code
- Duplicate checklist items across subsections (e.g., two "Testing" checklists with overlapping items)
- Step number references in later sections that are off-by-one from the actual workflow numbering

## Structural Anti-patterns
- Embedding 100+ lines of HTML/CSS/JS templates inline when a reference to an external file would suffice
- Including complete helper implementations (test infrastructure, reporters) instead of describing the pattern and letting the agent generate
- No persona/identity statement — skill files that jump straight into instructions without framing agent behavior

## Project Conventions
- Skill files use YAML frontmatter with `name` and `description` fields
- Skills are versioned in directories (v1.md, v2.md) within topic folders (shopify/)
- Vietnamese is used in some skill files — appears to be the author's primary working language
- External test directories are preferred over in-repo test files for live API testing patterns

## Quality Benchmarks
- Good: Specific API mapping tables, concrete code examples, numbered gotchas with rationale
- Good: Checklists that enforce workflow discipline and scope boundaries
- Good: Error preservation requirements with specific verification methods
- Needs improvement: Length management (extract boilerplate code), section numbering accuracy, path portability, persona statements
