<!-- markdownlint-disable -->

# Hardening Report: snok--install-poetry/v1.4.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **snok--install-poetry/v1.4.2** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All `uses:` references in the workflow files are pinned to mutable version tags rather than immutable 40-character SHA commit hashes, making the action vulnerable to supply-chain attacks if any upstream action is compromised or a tag is moved.

Failing references:
- lint.yml: `actions/checkout@v6`, `actions/setup-python@v6`, `actions/cache@v5`, `mfinelli/setup-shfmt@v4`
- tag_release.yml: `actions/checkout@v6`
- test.yml: `actions/checkout@v6`, `actions/setup-python@v6` (multiple), `snok/install-poetry@v1`, `snok/install-poetry@v1.2`, `snok/install-poetry@v1.3`

Locations:

- `.github/workflows/lint.yml:8`
- `.github/workflows/lint.yml:9`
- `.github/workflows/lint.yml:13`
- `.github/workflows/lint.yml:16`
- `.github/workflows/tag_release.yml:11`
- `.github/workflows/test.yml:20`
- `.github/workflows/test.yml:22`
- `.github/workflows/test.yml:24`
- `.github/workflows/test.yml:116`
- `.github/workflows/test.yml:117`
- `.github/workflows/test.yml:118`

### permissions (severity: medium)

None of the workflow files define a top-level `permissions:` key, and no individual jobs define job-level `permissions:` keys. Without explicit permissions, workflows run with the default (potentially broad) token permissions, violating the principle of least privilege.

Locations:

- `.github/workflows/lint.yml:1`
- `.github/workflows/tag_release.yml:1`
- `.github/workflows/test.yml:1`

### script-injection (severity: high)

Sub-rule (b): Unquoted shell variable expansions of untrusted input-derived data in main.sh.

1. `$INSTALLATION_ARGUMENTS` (sourced from `inputs.installation-arguments` via the `INSTALLATION_ARGUMENTS` env var set in action.yml) is expanded unquoted in two `python3` invocations:
   - Line 27: `python3 "$INSTALLATION_SCRIPT" --yes $INSTALLATION_ARGUMENTS`
   - Line 29: `python3 "$INSTALLATION_SCRIPT" --yes --version="$VERSION" $INSTALLATION_ARGUMENTS`
   An attacker-controlled value containing shell metacharacters (`;`, `|`, `&`, `$(...)`) would be interpreted by the shell.

2. `${plugins}` (derived from `inputs.plugins` via `$POETRY_PLUGINS`) is expanded unquoted on line 46:
   - `poetry self add ${plugins}`
   This allows word-splitting and glob expansion of attacker-controlled plugin names.

Locations:

- `main.sh:27`
- `main.sh:29`
- `main.sh:46`

### script-injection (severity: high)

Sub-rule (a): Direct `${{ }}` expression interpolation inside a `run:` shell command block in test.yml.

The `test-latest-version-when-unspecified` job's `run:` block directly interpolates `${{ needs.check-latest.outputs.latest-poetry-version }}` — a step output value from a prior job — into shell commands:

```
assert_in "." "${{ needs.check-latest.outputs.latest-poetry-version }}"
assert_in "${{ needs.check-latest.outputs.latest-poetry-version }}" "$(poetry --version)"
```

The step output is fetched from PyPI via `jq` and written to `$GITHUB_OUTPUT` without sanitization. If the PyPI response were tampered with or the value contained shell metacharacters, it would be executed in the runner shell. The value should be passed via an `env:` variable and double-quoted.

Locations:

- `.github/workflows/test.yml:107`
- `.github/workflows/test.yml:108`

### github-env-injection (severity: high)

In main.sh (line 32), the value `$INSTALL_PATH` is written to `$GITHUB_PATH` without sanitization:

```bash
echo "$INSTALL_PATH/bin" >> "$GITHUB_PATH"
```

`$INSTALL_PATH` is set from `${POETRY_HOME:-$HOME/.local}`. `POETRY_HOME` is an inherited process environment variable that can be set arbitrarily by the calling workflow. Per the check's scope, any process env var read inside a composite action's `run:` block that was not computed in the same script is treated as untrusted. A newline-containing value in `POETRY_HOME` could inject additional entries into `$GITHUB_PATH`, potentially hijacking the `PATH` for subsequent steps. The required sanitization (`printf '%s' "$INSTALL_PATH" | tr -d '\n\r'`) is absent.

Locations:

- `main.sh:32`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, permissions, script-injection, github-env-injection

**Notes:**

Fixed all 5 findings:
1. unpinned-uses: Pinned all action references in lint.yml, tag_release.yml, and test.yml to full 40-char SHAs (actions/checkout@df4cb1c, actions/setup-python@ece7cb0, actions/cache@caa2961, mfinelli/setup-shfmt@a25fda4, snok/install-poetry@a783c32/44d50c1/93ada01).
2. permissions: Added top-level `permissions: {}` to all 3 workflow files plus job-level minimal permissions (contents: read for most jobs, contents: write for tag-v1 job).
3. script-injection (main.sh lines 27,29,46): Used `read -ra` to parse $INSTALLATION_ARGUMENTS and $plugins into arrays, then expanded with `"${array[@]}"` to prevent shell metacharacter injection while preserving word-splitting behavior.
4. script-injection (test.yml lines 107-108): Moved `${{ needs.check-latest.outputs.latest-poetry-version }}` into an `env:` block as LATEST_POETRY_VERSION and referenced it as `"$LATEST_POETRY_VERSION"` in the shell script.
5. github-env-injection (main.sh line 32): Sanitized $INSTALL_PATH with `printf '%s' "$INSTALL_PATH" | tr -d '\n\r'` before writing to $GITHUB_PATH.

