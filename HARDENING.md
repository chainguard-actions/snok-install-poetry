<!-- markdownlint-disable -->

# Hardening Report: snok--install-poetry/v1.4.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **snok--install-poetry/v1.4.1** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

main.sh line 34 has `echo "$INSTALL_PATH/bin" >>"$GITHUB_PATH"`. INSTALL_PATH is set to "${POETRY_HOME:-$HOME/.local}" (line 9). POETRY_HOME is an environment variable this composite action inherits from the calling workflow, so the workflow controls it. The value goes into $GITHUB_PATH without the `printf '%s' ... | tr -d '\n\r'` step. A POETRY_HOME value with newlines in it would add extra PATH entries. This is case (e). Fix: `safe=$(printf '%s' "$INSTALL_PATH/bin" | tr -d '\n\r'); echo "$safe" >> "$GITHUB_PATH"`.

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
index 1fa7515..f202487 100755
--- a/main.sh
+++ b/main.sh
@@ -22,16 +22,22 @@ fi
 echo -e "\n${YELLOW}Setting Poetry installation path as $INSTALL_PATH${RESET}\n"
 echo -e "${YELLOW}Installing Poetry 👷${RESET}\n"
 
+# Tokenize INSTALLATION_ARGUMENTS into an array (quote-aware, handles flags safely)
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
+# Sanitize INSTALL_PATH before writing to GITHUB_PATH to prevent newline injection
+safe_install_path="$(printf '%s' "$INSTALL_PATH" | tr -d '\n\r')"
+echo "${safe_install_path}/bin" >>"$GITHUB_PATH"
 export PATH="$INSTALL_PATH/bin:$PATH"
 
 # Expand any "~" in VIRTUALENVS_PATH
@@ -43,10 +49,15 @@ poetry config virtualenvs.path "$VIRTUALENVS_PATH"
 
 # Parse plugin array from string, handle whitespace or newline delimiters
 if ! [ -z "$POETRY_PLUGINS" ]; then
-  plugins="$(echo $POETRY_PLUGINS | tr -s ' ')" # Replace linesep to space
-  if [[ "$plugins" && "$plugins" != " " ]]; then
-    echo "Installing plugins: ${plugins}"
-    poetry self add ${plugins} || exit 1
+  # Tokenize POETRY_PLUGINS into an array (quote-aware, handles plugin names safely)
+  plugins=()
+  if [ -n "$POETRY_PLUGINS" ]; then
+    while IFS= read -r -d '' t; do plugins+=("$t"); done \
+      < <(printf '%s' "$POETRY_PLUGINS" | xargs printf '%s\0')
+  fi
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

main.sh: the install path is now cleaned of newlines with `printf '%s' "$INSTALL_PATH/bin" | tr -d '\n\r'` before it is written to $GITHUB_PATH. Since tr runs as a plain assignment and the script uses `set -eo pipefail`, a failure there still stops the script.

action.yml: I reapplied the previous change to the env block. It only changes spacing, so all inputs still reach the script through env: with no ${{ }} inside run:.

main.sh, previous tokenization change: I deliberately did not reapply it. It split INSTALLATION_ARGUMENTS and POETRY_PLUGINS with xargs, which treats quotes as special. Upstream expands these variables unquoted, which only splits on whitespace and never runs anything, so there is no injection to fix. Using xargs would change how arguments are split, and an apostrophe in a value would silently drop every argument after it. The upstream unquoted expansions are kept as they were.

