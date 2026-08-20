<!-- markdownlint-disable -->

# Hardening Report: snok--install-poetry/v1.3.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **snok--install-poetry/v1.3.2** was hardened automatically. 3 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): The `set-matrix-vars` step in test.yml directly interpolates `${{ env.SUPPORTED_POETRY_VERSIONS }}`, `${{ env.SUPPORTED_PYTHON_VERSIONS }}`, and `${{ env.SUPPORTED_OPERATING_SYSTEMS }}` inside a `run:` shell command string. Even though these are `env.*` values defined in the same file, they are substituted by the GitHub Actions template engine before the shell ever sees them, meaning any special shell characters in those values are parsed by bash — constituting a script-injection risk. Offending lines: `echo "::set-output name=full-matrix::{${{ env.SUPPORTED_POETRY_VERSIONS }},${{ env.SUPPORTED_PYTHON_VERSIONS }},${{ env.SUPPORTED_OPERATING_SYSTEMS }}}"` and similar.

Locations:

- `.github/workflows/test.yml:41`

### unpinned-uses (severity: high)

Multiple workflow files reference actions using mutable tags or branch names instead of pinned 40-character SHA digests, making them vulnerable to supply-chain attacks if the referenced tag or branch is moved or compromised. Failing references include: `actions/checkout@v2`, `actions/setup-python@v2`, `ludeeus/action-shellcheck@master`, `snok/install-poetry@v1.1.4`, `snok/install-poetry@v1.1.6`, `snok/install-poetry@v1`, `snok/install-poetry@v1.1`, `snok/install-poetry@v1.2`.

Locations:

- `.github/workflows/test.yml:55`
- `.github/workflows/shellcheck.yml:8`
- `.github/workflows/tag_release.yml:12`

### missing-permissions (severity: medium)

None of the three workflow files define a top-level `permissions:` key, and no individual job within them defines a `permissions:` block either. Without explicit permissions, workflows run with the default token permissions (which can be `write-all` on some repository configurations), granting broader access than necessary. All three files — test.yml, shellcheck.yml, and tag_release.yml — are affected.

Locations:

- `.github/workflows/test.yml:1`
- `.github/workflows/shellcheck.yml:1`
- `.github/workflows/tag_release.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all three findings across test.yml, shellcheck.yml, and tag_release.yml:

1. script-injection: Moved ${{ env.SUPPORTED_POETRY_VERSIONS }}, ${{ env.SUPPORTED_PYTHON_VERSIONS }}, and ${{ env.SUPPORTED_OPERATING_SYSTEMS }} out of the run: shell string in the set-matrix-vars step and into the step's env: block as POETRY_VERSIONS, PYTHON_VERSIONS, and OPERATING_SYSTEMS. The shell script now references them as plain $VAR environment variables.

2. unpinned-uses: Pinned all 8 unpinned action references to full 40-character SHA digests with the original tag preserved as a comment: actions/checkout@v2, actions/setup-python@v2, ludeeus/action-shellcheck@master, snok/install-poetry@v1.1.4, snok/install-poetry@v1.1.6, snok/install-poetry@v1, snok/install-poetry@v1.1, snok/install-poetry@v1.2.

3. missing-permissions: Added permissions: {} at the top level of all three workflow files. The tag_release.yml job also has a job-level permissions: contents: write since it needs to push git tags.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script injection findings: (1) In .github/workflows/test.yml, moved `${{ env.LATEST_POETRY }}` from inside the `run:` shell string to the step's `env:` block as `LATEST_POETRY: ${{ env.LATEST_POETRY }}`, then referenced it as `$LATEST_POETRY` in the shell script. (2) In main.sh, replaced the two unquoted `${INSTALLATION_ARGUMENTS}` expansions (lines 20 and 23) with a safe xargs-based array tokenization pattern: the value is parsed with `printf '%s' "${INSTALLATION_ARGUMENTS}" | xargs printf '%s\0'` into a null-delimited stream, read into a bash array `installation_args`, and then expanded as `"${installation_args[@]+"${installation_args[@]}"}"` — preserving argument boundaries while preventing shell metacharacter injection.

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed script injection vulnerability in hardened/action/scripts/v1.2/main.sh line 17. Changed `python3 $installation_script --yes --version=$version` to `python3 "$installation_script" --yes --version="$version"` by double-quoting both the `$installation_script` (positional arg $6) and `$version` (positional arg $5) variables to prevent shell metacharacter interpretation.

