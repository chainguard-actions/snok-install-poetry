<!-- markdownlint-disable -->

# Hardening Report: snok--install-poetry/v1.4.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **snok--install-poetry/v1.4.1** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (b) violation: `$INSTALLATION_ARGUMENTS` is expanded unquoted in two `python3` invocations in main.sh (lines 28 and 31). This env var is populated from `inputs.installation-arguments` (via the `env:` block in action.yml). Because the expansion is unquoted, the shell performs word-splitting and glob expansion on the value, allowing an attacker-controlled input containing metacharacters (`;`, `|`, `&`, `$(...)`) to inject arbitrary shell commands. The script even suppresses the shellcheck warning with `# shellcheck disable=SC2086`, confirming the unquoted expansion is intentional but unsafe.

Offending lines:
- Line 28: `POETRY_HOME=$INSTALL_PATH python3 "$INSTALLATION_SCRIPT" --yes $INSTALLATION_ARGUMENTS`
- Line 31: `POETRY_HOME=$INSTALL_PATH python3 "$INSTALLATION_SCRIPT" --yes --version="$VERSION" $INSTALLATION_ARGUMENTS`

Locations:

- `main.sh:28`
- `main.sh:31`

### script-injection (severity: high)

Rule (b) violation: `$POETRY_PLUGINS` is expanded unquoted on line 41 (`plugins="$(echo $POETRY_PLUGINS | tr -s ' ')"`) and the derived `${plugins}` variable is also expanded unquoted on line 44 (`poetry self add ${plugins}`). This env var is populated from `inputs.plugins` (via the `env:` block in action.yml). An attacker-controlled plugin list containing shell metacharacters (`;`, `|`, `&`, `$(...)`) would be interpreted by the shell, enabling arbitrary command injection.

Offending lines:
- Line 41: `plugins="$(echo $POETRY_PLUGINS | tr -s ' ')"`
- Line 44: `poetry self add ${plugins} || exit 1`

Locations:

- `main.sh:41`
- `main.sh:44`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection vulnerabilities in main.sh:

1. INSTALLATION_ARGUMENTS (lines 28 & 31): Replaced unquoted `$INSTALLATION_ARGUMENTS` expansion with a bash array. Used `read -ra installation_args <<< "$INSTALLATION_ARGUMENTS"` to safely split arguments, then `"${installation_args[@]}"` for safe expansion. Removed the `# shellcheck disable=SC2086` comments that were suppressing the warning.

2. POETRY_PLUGINS (lines 41 & 44): Replaced unquoted `$POETRY_PLUGINS` and `${plugins}` expansions with a bash array. Used `printf '%s' "$POETRY_PLUGINS"` (quoted) to normalize whitespace/newlines safely, then `read -ra plugins <<< "$normalized_plugins"` to split into an array, then `"${plugins[@]}"` for safe expansion in the `poetry self add` call.

Both fixes prevent word-splitting and glob expansion on attacker-controlled values while preserving the original functionality of passing multiple space-separated arguments.

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

Fixed github-env-injection in actions/hardened/snok--install-poetry/v1.4.1/main.sh. The INSTALL_PATH variable (derived from the untrusted POETRY_HOME environment variable) was being written directly to $GITHUB_PATH without sanitization. Added a sanitization step using `safe=$(printf '%s' "$INSTALL_PATH/bin" | tr -d '\n\r')` before writing to $GITHUB_PATH, preventing newline injection attacks that could add arbitrary entries to the runner's PATH.

