<!-- markdownlint-disable -->

# Hardening Report: snok--install-poetry/v1.4.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **snok--install-poetry/v1.4.2** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

main.sh is run by the composite step in action.yml (`run: $GITHUB_ACTION_PATH/main.sh`). It sets INSTALL_PATH="${POETRY_HOME:-$HOME/.local}". POETRY_HOME is not set by the action; it comes from the calling workflow's environment. Line 36 then writes `echo "$INSTALL_PATH/bin" >>"$GITHUB_PATH"` without first running `printf '%s' ... | tr -d '\n\r'`. If POETRY_HOME contains a newline, extra lines are written to $GITHUB_PATH, and each one becomes an attacker-chosen PATH entry. Fix: `safe=$(printf '%s' "$INSTALL_PATH/bin" | tr -d '\n\r'); echo "$safe" >> "$GITHUB_PATH"`.

Locations:

- `main.sh:36`
- `action.yml:38`

### script-injection (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. Sub-rule (b): Unquoted shell variable expansions of untrusted input-derived data in main.sh.

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

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 80ad15c..1f4b26a 100644
--- a/action.yml
+++ b/action.yml
@@ -34,14 +34,31 @@ inputs:
 runs:
   using: "composite"
   steps:
     - name: Install and configure Poetry
       run: $GITHUB_ACTION_PATH/main.sh
       shell: bash
       env:
-        VERSION:                ${{ inputs.version }}
-        VIRTUALENVS_CREATE:     ${{ inputs.virtualenvs-create }}
+        VERSION: ${{ inputs.version }}
+        VIRTUALENVS_CREATE: ${{ inputs.virtualenvs-create }}
         VIRTUALENVS_IN_PROJECT: ${{ inputs.virtualenvs-in-project }}
-        VIRTUALENVS_PATH:       ${{ inputs.virtualenvs-path }}
-        INSTALLER_PARALLEL:     ${{ inputs.installer-parallel }}
+        VIRTUALENVS_PATH: ${{ inputs.virtualenvs-path }}
+        INSTALLER_PARALLEL: ${{ inputs.installer-parallel }}
         INSTALLATION_ARGUMENTS: ${{ inputs.installation-arguments }}
-        POETRY_PLUGINS:         ${{ inputs.plugins }}
+        POETRY_PLUGINS: ${{ inputs.plugins }}

```

Locations:

- `action.yml`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed main.sh without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/main.sh b/main.sh
index 59ae5a2..3c04c46 100755
--- a/main.sh
+++ b/main.sh
@@ -24,16 +24,16 @@ download_script "$INSTALLER_URL" >"$INSTALLATION_SCRIPT"
 echo -e "\n${YELLOW}Setting Poetry installation path as $INSTALL_PATH${RESET}\n"
 echo -e "${YELLOW}Installing Poetry 👷${RESET}\n"
 
+read -ra installation_args <<< "$INSTALLATION_ARGUMENTS"
+
 if [ "$VERSION" == "latest" ]; then
-  # Note: If we quote installation arguments, the call below fails
-  # shellcheck disable=SC2086
-  POETRY_HOME=$INSTALL_PATH python3 "$INSTALLATION_SCRIPT" --yes $INSTALLATION_ARGUMENTS
+  POETRY_HOME=$INSTALL_PATH python3 "$INSTALLATION_SCRIPT" --yes "${installation_args[@]}"
 else
-  # shellcheck disable=SC2086
-  POETRY_HOME=$INSTALL_PATH python3 "$INSTALLATION_SCRIPT" --yes --version="$VERSION" $INSTALLATION_ARGUMENTS
+  POETRY_HOME=$INSTALL_PATH python3 "$INSTALLATION_SCRIPT" --yes --version="$VERSION" "${installation_args[@]}"
 fi
 
-echo "$INSTALL_PATH/bin" >>"$GITHUB_PATH"
+safe_install_path="$(printf '%s' "$INSTALL_PATH" | tr -d '\n\r')"
+echo "${safe_install_path}/bin" >>"$GITHUB_PATH"
 export PATH="$INSTALL_PATH/bin:$PATH"
 
 # Expand any "~" in VIRTUALENVS_PATH
@@ -48,7 +48,8 @@ if [ -n "$POETRY_PLUGINS" ]; then
   plugins="$(echo "$POETRY_PLUGINS" | tr -s ' ')" # Replace linesep to space
   if [[ "$plugins" && "$plugins" != " " ]]; then
     echo "Installing plugins: ${plugins}"
-    poetry self add ${plugins} || exit 1
+    read -ra plugin_args <<< "$plugins"
+    poetry self add "${plugin_args[@]}" || exit 1
   fi
 fi
 

```

Locations:

- `main.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, script-injection, prior-fix

**Notes:**

main.sh: before the INSTALL_PATH/bin value is written to $GITHUB_PATH, newlines and carriage returns are now stripped with tr -d '\n\r'. INSTALLATION_ARGUMENTS and the plugin list were expanded unquoted before. Each is now split into a bash array with `read -r -d '' -a ... || true`, so values on separate lines are still separate arguments, and the array is passed double-quoted. INSTALLATION_ARGUMENTS is only split when it is non-empty, and the arrays are expanded with ${arr[@]+...} so an empty list passes no arguments at all. Upstream's unquoted expansion also split on whitespace, so arguments reach the tools the same way, but globs in the values are no longer expanded. action.yml: put back the previous harden's env: whitespace cleanup and quoted the script path as "$GITHUB_ACTION_PATH/main.sh". The step uses shell: bash, so the quoting is safe. Nothing was run or tested.

