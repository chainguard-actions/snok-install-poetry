<!-- markdownlint-disable -->

# Hardening Report: snok--install-poetry/v1.3.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **snok--install-poetry/v1.3.4** was hardened automatically. 2 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (b) violation: The env var $INSTALLATION_ARGUMENTS, which holds the value of inputs.installation-arguments (a user-controlled composite action input), is expanded unquoted in two shell commands in main.sh. Unquoted expansion allows an attacker to inject shell metacharacters (semicolons, pipes, backticks, etc.) and achieve arbitrary command execution. The shellcheck disable comments (SC2086) acknowledge the lack of quoting. Offending lines: `POETRY_HOME=$INSTALL_PATH python3 "$INSTALLATION_SCRIPT" --yes $INSTALLATION_ARGUMENTS` (both the 'latest' and versioned branches).

Locations:

- `main.sh:23`
- `main.sh:26`

### github-env-injection (severity: high)

The variable INSTALL_PATH is derived from the inherited process env var POETRY_HOME (via `INSTALL_PATH="${POETRY_HOME:-$HOME/.local}"`). POETRY_HOME is not set by this action — it is inherited from the calling workflow environment and must be treated as untrusted. Its value is written directly to $GITHUB_PATH (`echo "$INSTALL_PATH/bin" >> "$GITHUB_PATH"`) without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). A calling workflow that sets POETRY_HOME to a value containing newlines can inject arbitrary additional entries into the runner's PATH.

Locations:

- `main.sh:30`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed two high-severity findings in hardened/action/main.sh:

1. script-injection: Replaced unquoted `$INSTALLATION_ARGUMENTS` expansion (with SC2086 suppression comments) with a proper xargs-based tokenization into a bash array (`inst_args`). The array is built with a guard for empty input, xargs for quote-aware splitting, and NUL-delimited read loop (compatible with bash 3.2). Both the 'latest' and versioned branches now expand `"${inst_args[@]}"` safely.

2. github-env-injection: Added sanitization of `$INSTALL_PATH` (which is derived from the untrusted `POETRY_HOME` env var) before writing to `$GITHUB_PATH`. The value is stripped of newlines/carriage-returns via `printf '%s' "$INSTALL_PATH" | tr -d '\n\r'` and stored in `safe_install_path`, which is used for both the GITHUB_PATH write and the PATH export.

### Iteration 2

**Fixes applied:** unpinned-uses, missing-permissions, script-injection

**Notes:**

Fixed all three findings across .github/workflows/lint.yml, .github/workflows/tag_release.yml, and .github/workflows/test.yml:

1. unpinned-uses: Pinned all action references to full 40-char commit SHAs with tag comments. actions/checkout@v3→a37ce91, actions/setup-python@v4→7f4fc3e, actions/cache@v3→6f8efc2, mfinelli/setup-shfmt@v2→8ec8aa2 (v2.0.1), snok/install-poetry@v1→93ada01 (v1.3.4), snok/install-poetry@v1.2→44d50c1, snok/install-poetry@v1.3→93ada01 (v1.3.4). Note: snok/install-poetry@v1 and @v1.3 tags returned persistent 504 errors from GitHub API; used v1.3.4 SHA as the latest known v1.x release.

2. missing-permissions: Added top-level `permissions: {}` to all three workflow files. For tag_release.yml, added job-level `permissions: contents: write` since that job pushes git tags.

3. script-injection: In test.yml's test-latest-version-when-unspecified job, moved `${{ needs.check-latest.outputs.latest-poetry-version }}` from inline shell interpolation into an `env:` block as `LATEST_POETRY_VERSION`, then referenced it as `$LATEST_POETRY_VERSION` in the run script.

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted variable expansions in .github/workflows/tag_release.yml. Changed `git tag $major_tag` and `git tag $minor_tag` to `git tag "$major_tag"` and `git tag "$minor_tag"` respectively. These variables are derived from GITHUB_REF and must be quoted to prevent shell metacharacter interpretation.

