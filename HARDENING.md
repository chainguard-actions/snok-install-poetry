<!-- markdownlint-disable -->

# Hardening Report: snok--install-poetry/v1.4.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **snok--install-poetry/v1.4.0** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

main.sh, which the composite step runs via `$GITHUB_ACTION_PATH/main.sh`, sets INSTALL_PATH="${POETRY_HOME:-$HOME/.local}". POETRY_HOME is a process env var inherited from the calling workflow, so the workflow controls it. The script then runs `echo "$INSTALL_PATH/bin" >>"$GITHUB_PATH"` without removing newlines first (`printf '%s' ... | tr -d '\n\r'`). If POETRY_HOME contains a newline, the script writes extra, attacker-chosen PATH entries. Fix: safe=$(printf '%s' "$INSTALL_PATH/bin" | tr -d '\n\r'); echo "$safe" >> "$GITHUB_PATH".

Locations:

- `main.sh:34`

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
index d3e3717..1d092cf 100755
--- a/main.sh
+++ b/main.sh
@@ -22,16 +22,20 @@ fi
 echo -e "\n${YELLOW}Setting Poetry installation path as $INSTALL_PATH${RESET}\n"
 echo -e "${YELLOW}Installing Poetry 👷${RESET}\n"
 
+install_args=()
+if [ -n "$INSTALLATION_ARGUMENTS" ]; then
+  while IFS= read -r -d '' t; do install_args+=("$t"); done \
+    < <(printf '%s' "$INSTALLATION_ARGUMENTS" | xargs printf '%s\0')
+fi
+
 if [ "$VERSION" == "latest" ]; then
-  # Note: If we quote installation arguments, the call below fails
-  # shellcheck disable=SC2086
-  POETRY_HOME=$INSTALL_PATH python3 "$INSTALLATION_SCRIPT" --yes $INSTALLATION_ARGUMENTS
+  POETRY_HOME=$INSTALL_PATH python3 "$INSTALLATION_SCRIPT" --yes "${install_args[@]}"
 else
-  # shellcheck disable=SC2086
-  POETRY_HOME=$INSTALL_PATH python3 "$INSTALLATION_SCRIPT" --yes --version="$VERSION" $INSTALLATION_ARGUMENTS
+  POETRY_HOME=$INSTALL_PATH python3 "$INSTALLATION_SCRIPT" --yes --version="$VERSION" "${install_args[@]}"
 fi
 
-echo "$INSTALL_PATH/bin" >>"$GITHUB_PATH"
+safe_install_path="$(printf '%s' "$INSTALL_PATH/bin" | tr -d '\n\r')"
+echo "$safe_install_path" >>"$GITHUB_PATH"
 export PATH="$INSTALL_PATH/bin:$PATH"
 
 # Expand any "~" in VIRTUALENVS_PATH
@@ -42,11 +46,13 @@ poetry config virtualenvs.in-project "$VIRTUALENVS_IN_PROJECT"
 poetry config virtualenvs.path "$VIRTUALENVS_PATH"
 
 # Parse plugin array from string, handle whitespace or newline delimiters
-if ! [ -z "$POETRY_PLUGINS" ]; then
-  plugins="$(echo $POETRY_PLUGINS | tr -s ' ')" # Replace linesep to space
-  if [[ "$plugins" && "$plugins" != " " ]]; then
-    echo "Installing plugins: ${plugins}"
-    poetry self add ${plugins} || exit 1
+if [ -n "$POETRY_PLUGINS" ]; then
+  plugins=()
+  while IFS= read -r -d '' t; do plugins+=("$t"); done \
+    < <(printf '%s' "$POETRY_PLUGINS" | xargs printf '%s\0')
+  if [ "${#plugins[@]}" -gt 0 ]; then
+    echo "Installing plugins: ${plugins[*]}"
+    poetry self add "${plugins[@]}" || exit 1
   fi
 fi
 

```

Locations:

- `main.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, prior-fix

**Notes:**

main.sh: the value written to $GITHUB_PATH ("$INSTALL_PATH/bin") now goes through printf | tr -d '\n\r' first. The installation-arguments and plugins inputs are now bash arrays, so each word stays a separate argument. They are filled with `read -r -d '' -a ... || true`, which splits on whitespace and newlines the same way upstream's unquoted $VAR did. The previous harden used an xargs split here, which honors quotes and would change the arguments, so I didn't reuse it. action.yml: re-applied the previous harden's change to the env block, which only removes extra spaces for alignment.

