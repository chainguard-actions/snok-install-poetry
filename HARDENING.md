<!-- markdownlint-disable -->

# Hardening Report: snok--install-poetry/v1.3.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **snok--install-poetry/v1.3.3** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All workflow files use mutable tag-based action references instead of pinned full SHA digests, making them vulnerable to supply-chain attacks if the referenced tag is moved or overwritten.

- lint.yml: actions/checkout@v3, actions/setup-python@v4, actions/cache@v3, mfinelli/setup-shfmt@v1
- tag_release.yml: actions/checkout@v3
- test.yml: actions/checkout@v3, actions/setup-python@v4, snok/install-poetry@v1, snok/install-poetry@v1.1, snok/install-poetry@v1.2, snok/install-poetry@v1.1.4, snok/install-poetry@v1.1.6

Locations:

- `.github/workflows/lint.yml:9`
- `.github/workflows/lint.yml:10`
- `.github/workflows/lint.yml:15`
- `.github/workflows/lint.yml:18`
- `.github/workflows/tag_release.yml:9`
- `.github/workflows/test.yml:18`
- `.github/workflows/test.yml:36`
- `.github/workflows/test.yml:55`
- `.github/workflows/test.yml:58`
- `.github/workflows/test.yml:75`
- `.github/workflows/test.yml:107`
- `.github/workflows/test.yml:121`
- `.github/workflows/test.yml:122`
- `.github/workflows/test.yml:123`
- `.github/workflows/test.yml:131`
- `.github/workflows/test.yml:133`

### missing-permissions (severity: medium)

None of the three workflow files define a top-level `permissions:` block, and no individual job within them defines job-level permissions either. Without explicit permissions, workflows inherit the default repository token permissions, which may be overly broad (e.g., write access to contents). Each workflow should declare minimal required permissions.

Locations:

- `.github/workflows/lint.yml:1`
- `.github/workflows/test.yml:1`
- `.github/workflows/tag_release.yml:1`

### script-injection (severity: high)

Rule (a) violation: In test.yml, the `test-latest-version-when-unspecified` job's `run:` block directly interpolates `${{ needs.check-latest.outputs.latest-poetry-version }}` into shell commands. The `needs.*.outputs.*` context is workflow-controllable and flows through YAML template substitution before the shell processes it, allowing an attacker who can influence the output value to inject arbitrary shell commands.

Offending lines:
  assert_in "." "${{ needs.check-latest.outputs.latest-poetry-version }}"
  assert_in "${{ needs.check-latest.outputs.latest-poetry-version }}" "$(poetry --version)"

Locations:

- `.github/workflows/test.yml:142`
- `.github/workflows/test.yml:143`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection

**Notes:**

Fixed all three findings across lint.yml, tag_release.yml, and test.yml:

1. unpinned-uses: Pinned all action references to full commit SHAs:
   - actions/checkout@v3 → @a37ce9120846195fa4ece8f58b268e6043cb2f26
   - actions/setup-python@v4 → @7f4fc3e22c37d6ff65e88745f38bd3157c663f7c
   - actions/cache@v3 → @6f8efc29b200d32929f49075959781ed54ec270c
   - mfinelli/setup-shfmt@v1 → @02ff610c5fd6f6b02c50b8b7a256071ebd25a7f4
   - snok/install-poetry@v1 → @a783c322200f0519c7926aa6faa857c4e23e9263
   - snok/install-poetry@v1.1 → @7844a06d2cc061e83d8234a6f8192fa268234712
   - snok/install-poetry@v1.2 → @44d50c1274bf7fe0860f5c0531e36c9092c15b23
   - snok/install-poetry@v1.1.4 → @fe3362f94a7d193ecae442ec43e79680358051ce
   - snok/install-poetry@v1.1.6 → @14a9190388d183e6952804c1cb053ae05412dbbc

2. missing-permissions: Added top-level permissions blocks: lint.yml and test.yml get `contents: read`; tag_release.yml gets `contents: write` (required for pushing tags).

3. script-injection: In test.yml's test-latest-version-when-unspecified job, moved `${{ needs.check-latest.outputs.latest-poetry-version }}` from the run: shell string into the step's env: block as LATEST_POETRY_VERSION, then referenced it as $LATEST_POETRY_VERSION in the shell script.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings:
1. hardened/action/main.sh: Replaced unquoted `$INSTALLATION_ARGUMENTS` expansion (lines 25 & 28) with a safe xargs-based tokenization into a bash array. The array is built with `while IFS= read -r -d '' t; do install_args+=("$t"); done < <(printf '%s' "$INSTALLATION_ARGUMENTS" | xargs printf '%s\0')` guarded by `if [ -n "$INSTALLATION_ARGUMENTS" ]`, then expanded as `"${install_args[@]}"`. This prevents shell metacharacter injection while correctly handling multi-argument inputs like `--force --preview`.
2. hardened/action/scripts/v1.2/main.sh: Added double-quotes around `$installation_script` and `$version` in the python3 invocation (changed to `"$installation_script"` and `--version="$version"`), preventing shell metacharacter injection from these positional parameters.

