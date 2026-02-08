# Troubleshooting

Common issues when setting up or running the Go plugin release pipeline, organized by category.

## Build Failures

### CGO dependency detected

**Symptom**: Cross-compilation fails with errors about missing C headers or `gcc` not found.

**Cause**: A dependency requires CGO. Go's cross-compilation only works out of the box for pure-Go code.

**Fix**: If the dependency has a pure-Go alternative (e.g., `modernc.org/sqlite` instead of `mattn/go-sqlite3`), switch to it. Otherwise, set `CGO_ENABLED=1` and use Docker-based cross-compilation with platform-specific toolchains, or use separate GitHub Actions runners per platform.

### Module not found during build

**Symptom**: `go build` fails with "package not found" or "missing go.sum entry".

**Cause**: `go mod download` was not run before the build step, or `go.sum` is out of date.

**Fix**: Ensure the workflow has `go mod download` before the cross-compilation step. Locally, run `go mod tidy` to update `go.sum`.

### Binary too large

**Symptom**: Compiled binaries are unexpectedly large (e.g., 20MB+).

**Cause**: Missing `-ldflags="-s -w"` strip flags, or the dependency tree is large.

**Fix**: Verify `-trimpath -ldflags="-s -w"` is applied in the build command. For further reduction, consider `upx --best` compression (adds ~50ms startup latency). Check `go tool nm` output for unexpected symbols.

## Wrapper Script Issues

### "Binary not found" at runtime

**Symptom**: The wrapper script prints `Binary not found: /path/to/binary-<os>-<arch>` and exits with code 1.

**Cause**: Architecture or OS detection produced an unexpected value that doesn't match any built binary.

**Fix**: Run `uname -s` and `uname -m` on the target machine to see what values are returned. Ensure the wrapper's `case` statements cover all variants. On some Linux ARM systems, `uname -m` may return `armv7l` instead of `aarch64` -- add a mapping if 32-bit ARM support is needed.

### "Permission denied" when running wrapper or binary

**Symptom**: `bash: ./bin/<name>: Permission denied`

**Cause**: The wrapper script or platform binary lacks execute permission.

**Fix**: Ensure the release workflow has `chmod +x release-tree/bin/{{BINARY_NAME}}` after creating the wrapper. Go-compiled binaries should already be executable, but verify with `ls -la release-tree/bin/` in the workflow.

### Windows: uname not available

**Symptom**: The wrapper script fails on native Windows cmd.exe or PowerShell because `uname` is not a recognized command.

**Cause**: The POSIX wrapper requires a Unix-like shell. Native Windows environments don't have `uname`.

**Fix**: The wrapper works in Git Bash, MSYS2, and Cygwin (all detected via `mingw*|msys*|cygwin*`). For native Windows support, create an additional `bin/<name>.bat` or `bin/<name>.ps1` that directly invokes `<name>-windows-amd64.exe`. Claude Code on Windows typically runs in a Git Bash-compatible environment.

## Orphan Branch Issues

### "failed to push" to releases branch

**Symptom**: `git push --force origin releases` fails with a rejection error.

**Cause**: Branch protection rules on the `releases` branch prevent force pushes, even from the workflow bot.

**Fix**: In GitHub repository settings, either:
1. Remove branch protection from `releases` (recommended -- it's machine-managed)
2. Add the `github-actions[bot]` user to the bypass list for force push restrictions

### "remote rejected" with permission error

**Symptom**: Push fails with `remote: Permission to <repo> denied to github-actions[bot]`.

**Cause**: The `GITHUB_TOKEN` lacks `contents: write` permission.

**Fix**: Ensure the workflow has the `permissions: contents: write` block at the top level. If the repository uses a custom Actions token or has restricted default permissions, update the repository settings under Settings > Actions > General > Workflow permissions.

## Marketplace Integration

### SHA mismatch after update

**Symptom**: Plugin installation fails or fetches unexpected content after updating marketplace.json with the new SHA.

**Cause**: The `releases` branch was updated again (e.g., by a second release) after copying the SHA but before updating the marketplace.

**Fix**: Always use the SHA printed by the specific workflow run. Do not read the SHA from `git log origin/releases` after the fact, as it may have changed. Re-run the workflow if needed and use the freshly printed SHA.

### Plugin not loading after install

**Symptom**: Plugin installs but Claude Code doesn't recognize hooks or metadata.

**Cause**: The `releases` branch is missing required files. The "Copy plugin metadata" step may not have copied `.claude-plugin/plugin.json` or `hooks/hooks.json`.

**Fix**: Verify the release workflow's metadata copy step includes all required directories:
```bash
cp -r .claude-plugin release-tree/
cp -r hooks release-tree/
```
Check that these directories exist in the source tree at the tagged commit.

### Hook not executing after install

**Symptom**: Plugin is installed and visible, but the hook binary never runs when the matched tool is used.

**Cause**: The `hooks.json` command path doesn't match the actual binary name in `bin/`.

**Fix**: Verify that the `command` field in `hooks.json` references the correct binary name:
```json
"command": "\"${CLAUDE_PLUGIN_ROOT}\"/bin/{{BINARY_NAME}}\""
```
The `{{BINARY_NAME}}` here must match the wrapper script name on the releases branch. The `${CLAUDE_PLUGIN_ROOT}` variable is resolved by Claude Code at runtime.

## CI Matrix Issues

### Windows test failures with path separators

**Symptom**: Tests pass on macOS/Linux but fail on Windows with path-related assertion errors.

**Cause**: Windows uses `\` as path separator, while macOS/Linux use `/`.

**Fix**: Use `filepath.Join()` and `filepath.FromSlash()` in Go code instead of string concatenation with `/`. In test assertions, normalize paths before comparison or use `filepath.Clean()`.

### macOS symlink resolution discrepancies

**Symptom**: Tests using `t.TempDir()` fail on macOS because paths don't match after `filepath.EvalSymlinks()`.

**Cause**: On macOS, `t.TempDir()` returns paths like `/var/folders/...`, but `/var` is a symlink to `/private/var`. After `EvalSymlinks()`, the path becomes `/private/var/folders/...`.

**Fix**: When comparing paths in tests, always apply `filepath.EvalSymlinks()` to both sides, or use the resolved path consistently. This is a macOS-specific quirk that doesn't affect Linux or Windows.

### Go version compatibility

**Symptom**: Tests pass on newer Go version but fail on the minimum version declared in `go.mod`.

**Cause**: Code uses language features or standard library functions introduced after the declared minimum version.

**Fix**: Ensure the CI matrix includes the minimum Go version from `go.mod`. If using Go 1.22 features (like range-over-int), declare `go 1.22` in `go.mod`. Test locally with the minimum version via `go install golang.org/dl/go1.22@latest && go1.22 test ./...`.

## Version Injection (Optional)

To embed the version tag in the compiled binary:

1. Declare a variable in `main.go`: `var version = "dev"`
2. Update the build command in `release.yml`:
   ```bash
   GOOS="$GOOS" GOARCH="$GOARCH" go build -trimpath \
     -ldflags="-s -w -X main.version=${{ steps.tag.outputs.tag }}" \
     -o "release-tree/bin/${output}" {{CMD_PATH}}
   ```
3. Update the Makefile similarly for local builds:
   ```makefile
   VERSION ?= dev
   GOFLAGS := -trimpath -ldflags="-s -w -X main.version=$(VERSION)"
   ```
4. Access the version in code: `fmt.Println(version)`
