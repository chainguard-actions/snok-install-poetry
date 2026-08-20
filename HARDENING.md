<!-- markdownlint-disable -->

# Hardening Report: snok--install-poetry/v1.4.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **snok--install-poetry/v1.4.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files use mutable tag-based `uses:` references instead of pinned 40-character SHA commits, making them vulnerable to supply-chain attacks if the referenced tag is moved.

lint.yml: actions/checkout@v4, actions/setup-python@v5, actions/cache@v4, mfinelli/setup-shfmt@v3
tag_release.yml: actions/checkout@v4
test.yml: actions/checkout@v4, actions/setup-python@v5, snok/install-poetry@v1, snok/install-poetry@v1.2, snok/install-poetry@v1.3

Locations:

- `.github/workflows/lint.yml:8`
- `.github/workflows/lint.yml:9`
- `.github/workflows/lint.yml:13`
- `.github/workflows/lint.yml:15`
- `.github/workflows/tag_release.yml:11`
- `.github/workflows/test.yml:20`
- `.github/workflows/test.yml:21`
- `.github/workflows/test.yml:41`
- `.github/workflows/test.yml:42`
- `.github/workflows/test.yml:109`
- `.github/workflows/test.yml:117`
- `.github/workflows/test.yml:126`
- `.github/workflows/test.yml:135`
- `.github/workflows/test.yml:155`
- `.github/workflows/test.yml:156`
- `.github/workflows/test.yml:157`

### missing-permissions (severity: medium)

None of the workflow files define a top-level `permissions:` key, and no job within them defines job-level permissions. Without explicit permissions, workflows run with the default (potentially broad) token permissions, violating the principle of least privilege.

Locations:

- `.github/workflows/lint.yml:1`
- `.github/workflows/tag_release.yml:1`
- `.github/workflows/test.yml:1`

### script-injection (severity: high)

Sub-rule (a): The `run:` block in the `test-latest-version-when-unspecified` job directly interpolates `${{ needs.check-latest.outputs.latest-poetry-version }}` into shell commands. This value flows from a prior step's output and is substituted into the shell script before execution, allowing an attacker who can influence the output (e.g., via a compromised PyPI response or a malicious version string) to inject arbitrary shell commands.

Offending lines:
  assert_in "." "${{ needs.check-latest.outputs.latest-poetry-version }}"
  assert_in "${{ needs.check-latest.outputs.latest-poetry-version }}" "$(poetry --version)"

Locations:

- `.github/workflows/test.yml:148`
- `.github/workflows/test.yml:149`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection

**Notes:**

Fixed all three findings across lint.yml, tag_release.yml, and test.yml:

1. unpinned-uses: Pinned all 7 action references to full 40-char SHAs (actions/checkout@v4, actions/setup-python@v5, actions/cache@v4, mfinelli/setup-shfmt@v3, snok/install-poetry@v1, snok/install-poetry@v1.2, snok/install-poetry@v1.3) with tag comments for readability.

2. missing-permissions: Added top-level `permissions: {}` to all three workflow files, plus job-level permissions (contents: read for checkout jobs, contents: write for the tag-pushing job, permissions: {} for the curl-only job).

3. script-injection: In test.yml's test-latest-version-when-unspecified job, moved `${{ needs.check-latest.outputs.latest-poetry-version }}` from the run: shell script into the step's env: block as LATEST_POETRY_VERSION, then referenced it as $LATEST_POETRY_VERSION in the shell commands.

### Iteration 2

**Fixes applied:** github-env-injection, script-injection

**Notes:**

Fixed three security findings in hardened/action/main.sh:
1. github-env-injection (line 32): Sanitized INSTALL_PATH/bin before writing to $GITHUB_PATH using `safe_install_path="$(printf '%s' "$INSTALL_PATH/bin" | tr -d '\n\r')"` to strip newlines that could inject arbitrary PATH entries.
2. script-injection (lines 27, 30): Replaced unquoted `$INSTALLATION_ARGUMENTS` expansion with xargs-based tokenization into a bash array (`install_args`), guarded by `[ -n "$INSTALLATION_ARGUMENTS" ]`, then expanded safely as `"${install_args[@]}"`.
3. script-injection (lines 39, 42): Replaced unquoted `$POETRY_PLUGINS` expansion (including the intermediate `echo $POETRY_PLUGINS` and `${plugins}`) with xargs-based tokenization into a bash array (`plugins`), guarded by `[ -n "$POETRY_PLUGINS" ]`, then expanded safely as `"${plugins[@]}"` in the `poetry self add` call.

