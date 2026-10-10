<!-- markdownlint-disable -->

# Hardening Report: rutajdash--prettier-cli-action/v1.0.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **rutajdash--prettier-cli-action/v1.0.2** was hardened automatically. 6 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Two `run:` steps in action.yml directly interpolate `${{ inputs.* }}` expressions inside shell command strings (sub-rule a). This allows an attacker who controls the inputs to inject arbitrary shell commands.

1. `prettier-install` step: `npm install --global prettier@${{ inputs.prettier_version }}` — the version string is interpolated directly into the shell command before the shell ever sees it.

2. `prettier-run` step: `echo "$(prettier --list-different --config ${{ inputs.config_path }} --ignore-path ${{ inputs.ignore_path }} --no-error-on-unmatched-pattern ${{ inputs.file_pattern }})"` — three separate `inputs.*` values are interpolated directly into the shell command string. Any of these could contain shell metacharacters (`;`, `|`, `$(...)`, etc.) that would be executed by the shell.

Locations:

- `action.yml:33`
- `action.yml:39`

### github-env-injection (severity: high)

The `prettier-run` step writes content derived from untrusted `inputs.*` values to `$GITHUB_OUTPUT` without sanitization. Specifically, `${{ inputs.config_path }}`, `${{ inputs.ignore_path }}`, and `${{ inputs.file_pattern }}` are embedded in the prettier command whose output is written to `$GITHUB_OUTPUT`. An attacker-controlled input containing newlines could inject additional `key=value` pairs into `$GITHUB_OUTPUT`, overwriting or spoofing subsequent step outputs. The randomized heredoc delimiter (`EOF=$(dd if=/dev/urandom ...)`) protects the delimiter from collision but does NOT sanitize the content written between the delimiters. The required sanitization step (`printf '%s' "$VAR" | tr -d '\n\r'`) is absent before the write.

Locations:

- `action.yml:38`

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

Fixed all findings in action.yml:
1. 'Installing Prettier' step: moved ${{ inputs.prettier_version }} into env: block as PRETTIER_VERSION, referenced as "prettier@$PRETTIER_VERSION" in the npm install command.
2. 'Running Prettier' step: moved ${{ inputs.config_path }}, ${{ inputs.ignore_path }}, and ${{ inputs.file_pattern }} into env: block. Used 'set --' idiom to build optional --config and --ignore-path flags safely. Used xargs tokenization loop for FILE_PATTERN (a list/glob input with default '**/*.js') into a bash array. Sanitized prettier output with 'tr -d '\n\r'' before writing to $GITHUB_OUTPUT to prevent newline injection.

