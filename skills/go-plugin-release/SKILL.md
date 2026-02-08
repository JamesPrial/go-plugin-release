---
name: go-plugin-release
description: >-
  This skill should be used when the user asks to "set up Go binary distribution",
  "create a release pipeline for a Go plugin", "add cross-compilation to a plugin",
  "set up GitHub Actions for Go", "configure a releases branch strategy",
  "add a wrapper script for platform detection", "distribute compiled Go binaries",
  "set up CI/CD for a Go Claude Code plugin", or needs to establish a two-branch
  release strategy for any Go-based Claude Code plugin with marketplace distribution.
version: 0.1.0
---

# Go Plugin Release Pipeline

Set up cross-platform binary compilation and distribution for Go-based Claude Code plugins. This skill provides templates and guidance for the complete pipeline: local builds, CI testing, cross-compilation, platform-detection wrapper scripts, orphan release branches, and marketplace integration.

## Architecture Overview

The pipeline uses a **two-branch strategy**:

- **`main` branch**: Source code only. The `bin/` directory is gitignored. Development, testing, and tagging happen here.
- **`releases` branch**: Orphan branch containing pre-built binaries for all platforms, a wrapper script, and plugin metadata. No source code. No commit history (force-pushed on each release).

**Supported platforms** (5 targets):

| OS | Architecture | Binary Suffix |
|----|-------------|---------------|
| macOS | Intel | `darwin-amd64` |
| macOS | Apple Silicon | `darwin-arm64` |
| Linux | x86_64 | `linux-amd64` |
| Linux | ARM64 | `linux-arm64` |
| Windows | x86_64 | `windows-amd64.exe` |

**Wrapper script**: A POSIX shell script at `bin/<name>` that detects the current OS/architecture via `uname` and dispatches to the correct platform binary via `exec`. This allows `hooks.json` to reference a single binary path that works on all platforms.

**Marketplace integration**: The marketplace pins to the `releases` branch by commit SHA, providing supply chain integrity guarantees.

## Prerequisites

Before setting up the pipeline, ensure the project has:

- Go 1.22+ with `go.mod` initialized
- A `cmd/<binary-name>/main.go` entry point
- A `.claude-plugin/plugin.json` manifest
- A `hooks/hooks.json` hook configuration
- A GitHub repository with Actions enabled
- No CGO dependencies (required for single-runner cross-compilation)

## Template Placeholders

All templates in `assets/` use `{{PLACEHOLDER}}` syntax. Replace these with project-specific values before committing.

| Placeholder | Description | Example |
|---|---|---|
| `{{BINARY_NAME}}` | Compiled binary name | `save-todos` |
| `{{CMD_PATH}}` | Path to main package | `./cmd/save-todos` |
| `{{GO_VERSION}}` | Primary Go version | `1.22` |
| `{{GO_VERSIONS}}` | YAML list for CI matrix | `['1.22', '1.23']` |
| `{{REPO_OWNER}}` | GitHub repository owner | `JamesPrial` |
| `{{REPO_NAME}}` | GitHub repository name | `todo-log` |
| `{{MARKETPLACE_NAME}}` | Marketplace repository name | `prial-plugins` |

Derive these from the project's `go.mod`, `.claude-plugin/plugin.json`, and GitHub repository URL.

## Setup Procedure

### Step 1: Configure .gitignore

Add `bin/` to `.gitignore` on the main branch to exclude compiled binaries from source control:

```
bin/
```

### Step 2: Create the Makefile

Read `assets/Makefile.tmpl` and write it to the project root as `Makefile` with all placeholders replaced.

The Makefile provides four targets:
- `build` -- Compile the binary to `bin/<name>` with `-trimpath -ldflags="-s -w"`
- `test` -- Run all tests with race detection (`-race`)
- `cover` -- Print test coverage summary
- `clean` -- Remove the built binary

### Step 3: Create the CI workflow

Read `assets/ci.yml.tmpl` and write it to `.github/workflows/ci.yml` with placeholders replaced.

The CI workflow runs on every push and pull request across a matrix of 3 operating systems (ubuntu, macos, windows) and the configured Go versions. A separate coverage job reports test coverage on Ubuntu.

### Step 4: Create the Release workflow

Read `assets/release.yml.tmpl` and write it to `.github/workflows/release.yml` with placeholders replaced.

This is the core of the pipeline. It triggers on `v*` tags and `workflow_dispatch`, then:

1. Runs the test suite as a safety gate
2. Cross-compiles binaries for all 5 platforms
3. Generates the platform-detection wrapper script
4. Copies plugin metadata (`.claude-plugin/`, `hooks/`, docs)
5. Pushes to an orphan `releases` branch
6. Creates a GitHub Release with all binaries and SHA256 checksums
7. Prints the marketplace.json update instructions with the new SHA

For a detailed walkthrough of each step, consult `references/release-workflow-details.md`.

### Step 5: Verify hooks.json

Confirm that `hooks/hooks.json` references the wrapper script using the plugin root variable:

```json
{
  "command": "\"${CLAUDE_PLUGIN_ROOT}\"/bin/{{BINARY_NAME}}\""
}
```

Replace `{{BINARY_NAME}}` with the actual binary name. The `${CLAUDE_PLUGIN_ROOT}` variable is resolved by Claude Code at runtime to the plugin's installation directory.

### Step 6: Test locally

```bash
make build    # Compile the binary
make test     # Run tests with race detection
./bin/<name>  # Verify the binary runs (may need stdin input)
```

### Step 7: Tag and release

```bash
git add -A && git commit -m "Add release pipeline"
git tag v0.1.0
git push origin main v0.1.0
```

Monitor the GitHub Actions run. On success, copy the printed SHA into the marketplace repository's `marketplace.json`.

## Release Flow (Ongoing)

After initial setup, each release follows this cycle:

1. Merge changes to `main`, update version in `plugin.json` and `CHANGELOG.md`
2. Tag: `git tag vX.Y.Z && git push origin vX.Y.Z`
3. Wait for the release workflow to complete
4. Copy the SHA from the workflow output to `marketplace.json` in the marketplace repo
5. Submit a PR to the marketplace repo

## Customization

### Additional platforms

Add targets to the `targets` string in the cross-compilation step (e.g., `freebsd/amd64`, `windows/arm64`). Add corresponding `case` entries in the wrapper script.

### Version injection

Embed the release tag in the binary via linker flags. Add `-X main.version=$TAG` to the `-ldflags` in `release.yml`. See `references/troubleshooting.md` for a complete example.

### Multiple binaries

If the plugin compiles more than one binary, duplicate the build loop or add a nested loop over command paths. Each binary needs its own wrapper script.

### CGO dependencies

If the project requires CGO (e.g., `mattn/go-sqlite3`), cross-compilation from a single Ubuntu runner won't work. Options:
- Switch to a pure-Go alternative (e.g., `modernc.org/sqlite`)
- Use a matrix strategy with one runner per target OS
- Use Docker containers with cross-compilation toolchains

## Bundled Resources

### Asset Templates

Adapt and write these templates into the target project:

- **`assets/Makefile.tmpl`** -- Makefile with build, test, cover, and clean targets
- **`assets/ci.yml.tmpl`** -- GitHub Actions CI workflow with multi-OS/Go-version matrix
- **`assets/release.yml.tmpl`** -- Full release workflow: cross-compile, wrapper, orphan branch, GitHub Release, marketplace instructions
- **`assets/wrapper.sh.tmpl`** -- Standalone platform-detection wrapper script (also embedded in release.yml, but useful for local testing)

### Reference Files

For deeper understanding and debugging:

- **`references/release-workflow-details.md`** -- Step-by-step walkthrough of every release.yml job with rationale for each decision
- **`references/troubleshooting.md`** -- Common issues organized by category: build failures, wrapper script problems, orphan branch conflicts, marketplace integration, CI matrix issues, and version injection
