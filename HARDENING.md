<!-- markdownlint-disable -->

# Hardening Report: snok--install-poetry/v1.4.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **snok--install-poetry/v1.4.1** was hardened automatically. 8 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All `uses:` references in lint.yml use mutable version tags instead of pinned 40-character SHA commits: `actions/checkout@v4` (line 9), `actions/setup-python@v5` (line 10), `actions/cache@v4` (line 14), `mfinelli/setup-shfmt@v3` (line 19). These are vulnerable to supply-chain attacks if the upstream tag is moved.

Locations:

- `.github/workflows/lint.yml:9`
- `.github/workflows/lint.yml:10`
- `.github/workflows/lint.yml:14`
- `.github/workflows/lint.yml:19`

### unpinned-uses (severity: high)

The `uses:` reference in tag_release.yml uses a mutable version tag instead of a pinned 40-character SHA commit: `actions/checkout@v4` (line 12). This is vulnerable to supply-chain attacks if the upstream tag is moved.

Locations:

- `.github/workflows/tag_release.yml:12`

### unpinned-uses (severity: high)

Multiple `uses:` references in test.yml use mutable version tags instead of pinned 40-character SHA commits: `actions/checkout@v4` (lines 26, 44, 63, 72, 82, 107), `actions/setup-python@v5` (lines 27, 45), `snok/install-poetry@v1` (line 128), `snok/install-poetry@v1.2` (line 129), `snok/install-poetry@v1.3` (line 130). These are vulnerable to supply-chain attacks if the upstream tags are moved.

Locations:

- `.github/workflows/test.yml:26`
- `.github/workflows/test.yml:27`
- `.github/workflows/test.yml:44`
- `.github/workflows/test.yml:45`
- `.github/workflows/test.yml:63`
- `.github/workflows/test.yml:72`
- `.github/workflows/test.yml:82`
- `.github/workflows/test.yml:107`
- `.github/workflows/test.yml:128`
- `.github/workflows/test.yml:129`
- `.github/workflows/test.yml:130`

### permissions (severity: medium)

missing-permissions: lint.yml has no top-level `permissions:` key and no job-level `permissions:` key on any job. Without explicit permissions, the workflow inherits the default repository permissions, which may be overly broad (e.g., write access to contents). Add `permissions: {}` at the top level and grant only the minimum required scopes.

Locations:

- `.github/workflows/lint.yml:1`

### permissions (severity: medium)

missing-permissions: tag_release.yml has no top-level `permissions:` key and no job-level `permissions:` key on any job. The workflow pushes tags and uses GITHUB_TOKEN, but does not restrict other permission scopes. Add a top-level `permissions:` block granting only `contents: write` (for tagging) and nothing else.

Locations:

- `.github/workflows/tag_release.yml:1`

### permissions (severity: medium)

missing-permissions: test.yml has no top-level `permissions:` key and no job-level `permissions:` key on any job. Without explicit permissions, the workflow inherits the default repository permissions. Add `permissions: {}` at the top level and grant only the minimum required scopes per job.

Locations:

- `.github/workflows/test.yml:1`

### script-injection (severity: high)

Rule (a) violation: The `run:` block in the `test-latest-version-when-unspecified` job directly interpolates `${{ needs.check-latest.outputs.latest-poetry-version }}` into shell commands. This value originates from a `jq` parse of an external HTTP response (PyPI JSON) and is injected verbatim into the shell before execution. A malicious or unexpected value in the PyPI response could inject arbitrary shell commands. The offending lines are:
  `assert_in "." "${{ needs.check-latest.outputs.latest-poetry-version }}"`
  `assert_in "${{ needs.check-latest.outputs.latest-poetry-version }}" "$(poetry --version)"`
Fix: store the value in an env var and reference it as `"$ENV_VAR"` instead.

Locations:

- `.github/workflows/test.yml:115`
- `.github/workflows/test.yml:116`

### github-env-injection (severity: high)

The `check-latest` job's `run:` block writes the output of `jq` (parsing an external HTTP response from PyPI) directly to `$GITHUB_OUTPUT` without sanitization (`printf '%s' ... | tr -d '\n\r'`). The raw `jq` output could contain newlines that allow injection of additional key=value pairs into the output file, which are then consumed by downstream jobs. The offending write is:
  `| jq -r '"version=" + .info.version' >> $GITHUB_OUTPUT`
Fix: capture the value, sanitize with `tr -d '\n\r'`, then write to `$GITHUB_OUTPUT`.

Locations:

- `.github/workflows/test.yml:108`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, permissions, script-injection, github-env-injection

**Notes:**

Fixed all 8 findings across 3 workflow files:

**lint.yml**: Added `permissions: {}` at top level. Pinned actions/checkout@v4→SHA, actions/setup-python@v5→SHA, actions/cache@v4→SHA, mfinelli/setup-shfmt@v3→SHA.

**tag_release.yml**: Added `permissions: contents: write` (minimum needed for git tagging/pushing). Pinned actions/checkout@v4→SHA.

**test.yml**: Added `permissions: {}` at top level. Pinned all 11 unpinned action references (actions/checkout@v4, actions/setup-python@v5, snok/install-poetry@v1/v1.2/v1.3) to full 40-char SHAs with tag comments. Fixed script-injection by moving `${{ needs.check-latest.outputs.latest-poetry-version }}` into an `env:` block. Fixed github-env-injection by capturing jq output, sanitizing with `tr -d '\n\r'`, then writing to `$GITHUB_OUTPUT`.

### Iteration 2

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed all three findings in hardened/action/main.sh:

1. script-injection (INSTALLATION_ARGUMENTS): Replaced unquoted `$INSTALLATION_ARGUMENTS` expansion with a quote-aware xargs-based tokenization into a bash array (`install_args`), guarded by `if [ -n "$INSTALLATION_ARGUMENTS" ]`. Expanded as `"${install_args[@]}"` in both the 'latest' and versioned installation branches.

2. script-injection (POETRY_PLUGINS): Replaced `echo $POETRY_PLUGINS | tr -s ' '` and unquoted `${plugins}` with a quote-aware xargs-based tokenization into a bash array (`plugins`), guarded by `if [ -n "$POETRY_PLUGINS" ]`. Expanded as `"${plugins[@]}"` in the `poetry self add` call.

3. github-env-injection (INSTALL_PATH): Added `safe_install_path="$(printf '%s' "$INSTALL_PATH" | tr -d '\n\r')"` before writing to `$GITHUB_PATH`, and used `${safe_install_path}/bin` in the echo to prevent newline injection from a caller-controlled `POETRY_HOME` environment variable.

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted shell variable expansions in .github/workflows/tag_release.yml lines 20-21. Changed `git tag $major_tag` and `git tag $minor_tag` to `git tag "$major_tag"` and `git tag "$minor_tag"` respectively. Both variables are derived from GITHUB_REF (a workflow-controllable env var) via a Python script, so quoting them prevents shell metacharacter injection.

