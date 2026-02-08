# go-plugin-release

A Claude Code plugin that guides you through setting up cross-platform binary compilation and distribution pipelines for Go-based Claude Code plugins.

## What it does

This plugin provides a skill that walks you through creating:

- **Makefile** for local build/test/cover/clean
- **CI workflow** with multi-OS/Go-version matrix testing
- **Release workflow** that cross-compiles for 5 platforms (macOS Intel/ARM, Linux x64/ARM, Windows x64), generates a platform-detection wrapper script, pushes to an orphan `releases` branch, and creates a GitHub Release with checksums
- **Wrapper script** that auto-detects OS/architecture and dispatches to the correct binary

## Install

```bash
claude plugin add JamesPrial/go-plugin-release
```

## Usage

Ask Claude Code to set up a release pipeline for your Go plugin:

- "Set up Go binary distribution for my plugin"
- "Create a release pipeline for this Go plugin"
- "Add cross-compilation and GitHub Actions CI"

The skill will read your project's `go.mod`, `plugin.json`, and repo URL to fill in the templates automatically.

## Pipeline Architecture

Uses a **two-branch strategy**:

| Branch | Contents | Purpose |
|--------|----------|---------|
| `main` | Source code, workflows, tests | Development and tagging |
| `releases` | Binaries, wrapper script, plugin metadata | Marketplace distribution (pinned by SHA) |

The `releases` branch is an orphan branch that gets force-pushed on each release — no history accumulation, no source code exposure.

## Supported Platforms

| OS | Architecture | Binary Suffix |
|----|-------------|---------------|
| macOS | Intel | `darwin-amd64` |
| macOS | Apple Silicon | `darwin-arm64` |
| Linux | x86_64 | `linux-amd64` |
| Linux | ARM64 | `linux-arm64` |
| Windows | x86_64 | `windows-amd64.exe` |

## Prerequisites for Target Projects

- Go 1.22+ with `go.mod`
- A `cmd/<name>/main.go` entry point
- `.claude-plugin/plugin.json` manifest
- `hooks/hooks.json` configuration
- GitHub repo with Actions enabled
- No CGO dependencies (for single-runner cross-compilation)

## License

MIT
