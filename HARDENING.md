<!-- markdownlint-disable -->

# Hardening Report: snok--install-poetry/v1.4.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **snok--install-poetry/v1.4.2** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (b): The env var $INSTALLATION_ARGUMENTS (sourced from inputs.installation-arguments via the env: block) is expanded **unquoted** in two shell invocations. An attacker-controlled value containing shell metacharacters (spaces, globs, semicolons, etc.) will be word-split by the shell, enabling argument injection into the python3 installer call. The shellcheck disable comment acknowledges this but does not mitigate the security risk.

Offending lines:
  Line 28: `POETRY_HOME=$INSTALL_PATH python3 "$INSTALLATION_SCRIPT" --yes $INSTALLATION_ARGUMENTS`
  Line 31: `POETRY_HOME=$INSTALL_PATH python3 "$INSTALLATION_SCRIPT" --yes --version="$VERSION" $INSTALLATION_ARGUMENTS`

Locations:

- `main.sh:28`
- `main.sh:31`

### script-injection (severity: high)

Rule (b): The shell variable `${plugins}` (derived from $POETRY_PLUGINS, which is sourced from inputs.plugins via the env: block) is expanded **unquoted** in `poetry self add ${plugins}`. An attacker-controlled value with shell metacharacters or extra whitespace-delimited tokens could inject arbitrary arguments or commands into the poetry invocation.

Offending line:
  Line 46: `poetry self add ${plugins} || exit 1`

Locations:

- `main.sh:46`

### github-env-injection (severity: high)

The value `$INSTALL_PATH/bin` is written to $GITHUB_PATH without sanitization. $INSTALL_PATH is derived from the inherited process env var $POETRY_HOME (set by the calling workflow via `INSTALL_PATH="${POETRY_HOME:-$HOME/.local}"`). Since $POETRY_HOME is a workflow-controlled env var, it is untrusted and could contain embedded newlines, allowing an attacker to inject additional arbitrary entries into $GITHUB_PATH. The required sanitization step (`printf '%s' "$INSTALL_PATH" | tr -d '\n\r'`) is absent.

Offending line:
  Line 33: `echo "$INSTALL_PATH/bin" >> "$GITHUB_PATH"`

Locations:

- `main.sh:33`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed all three findings in main.sh:
1. Lines 28/31 (script-injection): Replaced unquoted `$INSTALLATION_ARGUMENTS` with a bash array: `read -ra installation_args <<< "$INSTALLATION_ARGUMENTS"` then `"${installation_args[@]}"`. Removed the now-unnecessary `# shellcheck disable=SC2086` comments.
2. Line 46 (script-injection): Replaced unquoted `${plugins}` with a bash array: `read -ra plugin_args <<< "$plugins"` then `"${plugin_args[@]}"` in the `poetry self add` call.
3. Line 33 (github-env-injection): Added sanitization of INSTALL_PATH before writing to $GITHUB_PATH: `safe_install_path="$(printf '%s' "$INSTALL_PATH" | tr -d '\n\r')"` and then `echo "${safe_install_path}/bin" >> "$GITHUB_PATH"`.

