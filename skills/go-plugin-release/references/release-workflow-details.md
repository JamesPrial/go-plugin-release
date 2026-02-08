# Release Workflow Deep Dive

Detailed walkthrough of every step in the `release.yml` GitHub Actions workflow template. Consult this reference when customizing the workflow or debugging release failures.

## Trigger Configuration

The workflow fires on two events:

**Tag push** (`on.push.tags: ['v*']`): Automatically triggers when a tag matching `v*` is pushed (e.g., `v1.0.0`, `v2.3.1-beta`). This is the primary release mechanism.

**Manual dispatch** (`workflow_dispatch`): Allows re-running a release for any tag via the GitHub Actions UI. The `inputs.tag` field accepts a tag string (e.g., `v1.0.0`). Useful for re-releases after fixing workflow issues without creating new tags.

**Permissions**: `contents: write` grants the workflow token permission to push branches and create releases. Without this, the orphan branch push and `gh release create` steps fail with 403 errors.

## Test Job

The test job runs before the release job (`needs: test` dependency). It checks out the code, sets up Go, downloads/verifies modules, and runs the full test suite with race detection.

This is a safety gate: even though tests likely passed on CI before tagging, running them again at release time confirms the tagged commit is sound. The test job uses the same Go version as the release build to avoid version-specific failures.

## Tag Determination

The `Determine tag` step resolves the release tag from either source:

```bash
if [ -n "$INPUT_TAG" ]; then
  TAG="$INPUT_TAG"          # From workflow_dispatch input
else
  TAG="${GITHUB_REF#refs/tags/}"  # From git tag push event
fi
```

The `${GITHUB_REF#refs/tags/}` shell parameter expansion strips the `refs/tags/` prefix from the full ref, yielding just `v1.0.0`. The tag is exported via `$GITHUB_OUTPUT` so subsequent steps can reference it as `${{ steps.tag.outputs.tag }}`.

## Checkout at Tag

```yaml
- uses: actions/checkout@v4
  with:
    ref: ${{ steps.tag.outputs.tag }}
```

Checks out the exact tagged commit, not HEAD of main. This ensures the binaries are built from the committed, tagged source.

## Cross-Compilation

The core of the release: building platform-specific binaries in a loop.

```bash
targets="darwin/amd64 darwin/arm64 linux/amd64 linux/arm64 windows/amd64"
for target in $targets; do
  GOOS="${target%/*}"     # Everything before the /
  GOARCH="${target#*/}"   # Everything after the /
  output="{{BINARY_NAME}}-${GOOS}-${GOARCH}"
  if [ "$GOOS" = "windows" ]; then
    output="${output}.exe"
  fi
  GOOS="$GOOS" GOARCH="$GOARCH" go build -trimpath -ldflags="-s -w" \
    -o "release-tree/bin/${output}" {{CMD_PATH}}
done
```

### Shell parameter expansion

- `${target%/*}` removes the shortest suffix matching `/*`, extracting the OS (e.g., `darwin` from `darwin/amd64`)
- `${target#*/}` removes the shortest prefix matching `*/`, extracting the architecture (e.g., `amd64` from `darwin/amd64`)

### Build flags

- `-trimpath`: Removes local filesystem paths from the binary, improving reproducibility and avoiding information leakage
- `-ldflags="-s -w"`: `-s` strips the symbol table, `-w` strips DWARF debug information. Together they reduce binary size by ~30%

### Platform targets

| Target | Binary Name | Use Case |
|--------|-------------|----------|
| `darwin/amd64` | `name-darwin-amd64` | macOS on Intel |
| `darwin/arm64` | `name-darwin-arm64` | macOS on Apple Silicon |
| `linux/amd64` | `name-linux-amd64` | Linux on x86_64 |
| `linux/arm64` | `name-linux-arm64` | Linux on ARM64 (AWS Graviton, Raspberry Pi) |
| `windows/amd64` | `name-windows-amd64.exe` | Windows on x86_64 |

### Why cross-compilation works from Ubuntu

Go's toolchain natively supports cross-compilation without additional tools. Setting `GOOS` and `GOARCH` environment variables before `go build` is sufficient, provided the code has no CGO dependencies. Pure-Go libraries (like `modernc.org/sqlite`) enable this.

## Wrapper Script Generation

The wrapper is a POSIX shell script that detects the current platform and dispatches to the correct binary:

### OS detection

`uname -s` returns the kernel name:
- `Darwin` on macOS
- `Linux` on Linux
- `MINGW64_NT-*`, `MSYS_NT-*`, `CYGWIN_NT-*` on Windows (via Git Bash, MSYS2, Cygwin)

The `tr '[:upper:]' '[:lower:]'` normalizes to lowercase. The `case` statement then maps to Go-convention OS names.

### Architecture detection

`uname -m` returns the machine hardware name:
- `x86_64` on 64-bit Intel/AMD (mapped to `amd64`)
- `aarch64` on ARM64 Linux (mapped to `arm64`)
- `arm64` on ARM64 macOS (kept as `arm64`)

### Binary resolution

The wrapper computes `SCRIPT_DIR` using `$(cd "$(dirname "$0")" && pwd)` to get an absolute path to its own directory. The target binary is then `${SCRIPT_DIR}/{{BINARY_NAME}}-${OS}-${ARCH}`, with `.exe` appended for Windows.

### Dispatch via exec

`exec "$BINARY" "$@"` replaces the wrapper shell process with the actual binary. This means:
- The binary inherits the wrapper's PID
- stdin/stdout/stderr pass through cleanly
- Exit codes propagate correctly
- No extra shell process remains in the process tree

### Why the wrapper exists

`hooks.json` references a single path: `"${CLAUDE_PLUGIN_ROOT}/bin/{{BINARY_NAME}}"`. The wrapper at that path handles platform dispatch transparently, so hooks.json never needs platform-conditional logic.

## Plugin Metadata Copy

```bash
cp -r .claude-plugin release-tree/
cp -r hooks release-tree/
for f in README.md LICENSE CHANGELOG.md; do
  [ -f "$f" ] && cp "$f" release-tree/
done
```

The release tree must contain everything needed to install the plugin:
- `.claude-plugin/plugin.json` - Plugin manifest (name, version, description)
- `hooks/hooks.json` - Hook configuration
- Documentation files (optional, guarded by `[ -f "$f" ]`)

Source code (`cmd/`, `internal/`, `go.mod`, etc.) is intentionally excluded from the release tree.

## Orphan Branch Push

```bash
cd release-tree
git init
git checkout --orphan releases
git add -A
git -c user.name="github-actions[bot]" \
    -c user.email="github-actions[bot]@users.noreply.github.com" \
    commit -m "Release ${{ steps.tag.outputs.tag }}"
SHA=$(git rev-parse HEAD)
git push --force origin releases
```

### Why an orphan branch

`git checkout --orphan releases` creates a branch with no parent commits. This means:
- The releases branch has no connection to the main branch's commit history
- Each release replaces the entire branch (via `--force`)
- Clone size stays minimal (only one commit with current binaries, no history accumulation)
- Source code never appears in the releases branch

### Force push safety

The `--force` is intentional and safe here because:
- The releases branch is machine-managed (never manually edited)
- Only one commit exists at any time
- The previous SHA is obsolete once a new release is published
- Marketplace pins to SHA, so existing installations are unaffected

### SHA capture

`SHA=$(git rev-parse HEAD)` captures the new commit's SHA immediately after commit. This SHA is output via `$GITHUB_OUTPUT` for use in the marketplace update instructions.

## GitHub Release Creation

```bash
sha256sum bin/{{BINARY_NAME}}-* > checksums-sha256.txt
gh release create "${{ steps.tag.outputs.tag }}" \
  --repo "${{ github.repository }}" \
  --title "${{ steps.tag.outputs.tag }}" \
  --generate-notes \
  bin/{{BINARY_NAME}}-* checksums-sha256.txt
```

- `sha256sum` generates integrity checksums for all platform binaries
- `--generate-notes` auto-generates release notes from commits since the previous tag
- All platform binaries plus the checksums file are attached as release assets

The GitHub Release serves as a secondary distribution channel and provides a clear record of each version with downloadable artifacts.

## Marketplace Update Instructions

The final step prints the JSON snippet to update the marketplace:

```json
{
  "source": {
    "source": "github",
    "repo": "{{REPO_OWNER}}/{{REPO_NAME}}",
    "ref": "releases",
    "sha": "<commit-sha>"
  }
}
```

### SHA pinning for supply chain integrity

The marketplace pins to a specific commit SHA rather than just the branch name. This ensures:
- Users get exactly the binaries from the tested release
- A compromised `releases` branch push after the fact won't affect existing marketplace entries
- Version reproducibility: the SHA uniquely identifies the release contents

This is a manual step: copy the SHA from the workflow output into the marketplace repository's `marketplace.json` and submit a PR.
