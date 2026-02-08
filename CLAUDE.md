# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Claude Code **plugin** containing a single skill (`go-plugin-release`) that guides users through setting up cross-platform binary compilation and distribution pipelines for Go-based Claude Code plugins. This is a template/guidance repo — it contains no Go code itself.

## Repository Structure

```
.claude-plugin/plugin.json          # Plugin manifest
skills/go-plugin-release/
├── SKILL.md                        # Skill definition with frontmatter + setup procedure
├── assets/*.tmpl                   # Template files users copy into their projects
└── references/                     # Deep-dive docs for the release workflow and troubleshooting
```

## Key Concepts

- **SKILL.md frontmatter**: The YAML frontmatter (`name`, `description`, `version`) controls skill discovery and triggering. The `description` field determines when Claude Code activates this skill — it must contain trigger phrases users would say.
- **Template placeholders**: All `.tmpl` files use `{{PLACEHOLDER}}` syntax (e.g., `{{BINARY_NAME}}`, `{{CMD_PATH}}`). When applying templates, all placeholders must be replaced with project-specific values derived from the target project's `go.mod`, `plugin.json`, and GitHub repo URL.
- **Two-branch strategy**: The skill teaches a `main` (source) + `releases` (orphan branch with binaries only) model. The releases branch is force-pushed on each release — this is intentional.
- **Relative paths**: SKILL.md references `assets/` and `references/` as relative paths. If files are reorganized, these references must stay valid.

## Editing Guidelines

- When modifying templates, keep `{{PLACEHOLDER}}` syntax consistent and update the placeholder table in SKILL.md if adding/removing any.
- When editing SKILL.md, preserve the YAML frontmatter block — it's required for skill registration.
- The `references/` docs explain *why* decisions were made (e.g., why orphan branches, why `exec` in the wrapper). Keep rationale alongside technical details.
