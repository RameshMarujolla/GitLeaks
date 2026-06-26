# GitLeaks Workflow Issue & Fix

## 1. Issue Summary

The GitHub Actions workflow `.github/workflows/gitleaks.yaml` was failing with:

```
Error: Path does not exist: results.sarif
```

Additionally, the workflow logs showed a deprecation warning:

```
Warning: CodeQL Action v3 will be deprecated in December 2026.
Please update all occurrences of the CodeQL Action in your workflow files to v4.
```

## 2. Root Cause

The `Run GitLeaks` step in the workflow contained **trailing whitespace** after the backslash (`\`) line continuations in the `run:` block. In YAML, the `|` literal block preserves trailing spaces, and when Bash parsed the script it misinterpreted the escaped newlines.

This caused `gitleaks git` to receive an empty source path, producing the fatal error:

```
FTL stat  : no such file or directory
```

Because gitleaks crashed before writing the report, `results.sarif` was never created. The subsequent `upload-sarif` and `upload-artifact` steps therefore failed with `Path does not exist: results.sarif`.

## 3. Impact

- The SARIF report was **never generated**.
- Findings were **not uploaded** to the GitHub Security tab.
- The workflow artifact was **empty**.
- Secret scanning was effectively broken.

## 4. Example: Before the Fix

### Workflow YAML (problematic lines)

```yaml
      - name: Run GitLeaks
        id: gitleaks
        run: |
          gitleaks git \
            --report-format sarif \
            --report-path results.sarif
```

> Note: Each line above ended with spaces after the backslash. This is invisible in most editors but breaks Bash continuation parsing.

### GitHub Actions log output

```
gitleaks	Run GitLeaks	2026-06-26T17:14:21.3427805Z FTL stat  : no such file or directory
gitleaks	Run GitLeaks	2026-06-26T17:14:21.3451238Z ##[error]Process completed with exit code 1.
```

```
gitleaks	Upload SARIF to GitHub Security tab	2026-06-26T17:14:22.0583272Z ##[error]Path does not exist: results.sarif
```

## 5. Solution

### 5.1 Fix the gitleaks command

- Removed trailing whitespace after backslash continuations.
- Added an explicit source path `.` so gitleaks always knows which directory to scan.
- Added `set -euo pipefail` for robust shell error handling.

### 5.2 Update deprecated actions

- `github/codeql-action/upload-sarif@v3` → `v4`
- `actions/checkout@v4` → `v7` (Node 24 runtime)
- `actions/upload-artifact@v4` → `v7` (Node 24 runtime)

### 5.3 Guard upload steps

Added `hashFiles('results.sarif') != ''` to both upload steps so they only run when the report actually exists:

```yaml
if: always() && hashFiles('results.sarif') != ''
```

### 5.4 Redact secrets in artifacts and logs

Initially, the generated SARIF file contained the raw leaked secret in the snippet field:

```json
"snippet": {
  "text": "AKIA1234567890ABCDEF"
}
```

Because the artifact is downloadable by anyone with repo access, exposing the raw secret is a security risk. The `--redact` flag was added to mask secrets in both the SARIF output and the workflow logs.

## 6. Workflow Comparison

### 6.1 Workflow WITHOUT redaction (NOT recommended)

This version fixes the original `Path does not exist: results.sarif` error and updates the actions, but it does **not** redact secrets. The raw secret will be visible in the workflow logs and in the downloadable SARIF artifact.

```yaml
name: GitLeaks

on:
  push:
    branches:
      - main
  pull_request:

permissions:
  contents: read
  security-events: write

jobs:
  gitleaks:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0

      - name: Install GitLeaks
        run: |
          set -euo pipefail
          wget -q "https://github.com/gitleaks/gitleaks/releases/download/v8.30.1/gitleaks_8.30.1_linux_x64.tar.gz"
          tar -xzf gitleaks_8.30.1_linux_x64.tar.gz
          sudo mv gitleaks /usr/local/bin/

      - name: Run GitLeaks
        id: gitleaks
        run: |
          set -euo pipefail
          gitleaks git \
            --report-format sarif \
            --report-path results.sarif \
            --verbose \
            .

      - name: Upload SARIF to GitHub Security tab
        if: always() && hashFiles('results.sarif') != ''
        uses: github/codeql-action/upload-sarif@v4
        with:
          sarif_file: results.sarif

      - name: Upload GitLeaks report artifact
        if: always() && hashFiles('results.sarif') != ''
        uses: actions/upload-artifact@v7
        with:
          name: gitleaks-report
          path: results.sarif
          retention-days: 30
```

### 6.2 Workflow WITH redaction (recommended)

The only difference from the version above is the addition of `--redact`. This masks secrets in the logs and the SARIF artifact.

```yaml
name: GitLeaks

on:
  push:
    branches:
      - main
  pull_request:

permissions:
  contents: read
  security-events: write

jobs:
  gitleaks:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0

      - name: Install GitLeaks
        run: |
          set -euo pipefail
          wget -q "https://github.com/gitleaks/gitleaks/releases/download/v8.30.1/gitleaks_8.30.1_linux_x64.tar.gz"
          tar -xzf gitleaks_8.30.1_linux_x64.tar.gz
          sudo mv gitleaks /usr/local/bin/

      - name: Run GitLeaks
        id: gitleaks
        run: |
          set -euo pipefail
          gitleaks git \
            --report-format sarif \
            --report-path results.sarif \
            --verbose \
            --redact \
            .

      - name: Upload SARIF to GitHub Security tab
        if: always() && hashFiles('results.sarif') != ''
        uses: github/codeql-action/upload-sarif@v4
        with:
          sarif_file: results.sarif

      - name: Upload GitLeaks report artifact
        if: always() && hashFiles('results.sarif') != ''
        uses: actions/upload-artifact@v7
        with:
          name: gitleaks-report
          path: results.sarif
          retention-days: 30
```

## 7. Example: Before and After Redaction

### Console output BEFORE redaction (security risk)

Before adding `--redact`, the raw secret value was printed directly in the GitHub Actions workflow logs. Anyone with read access to the repository could view it:

```
gitleaks	Run GitLeaks	Finding:     aws_access_key = "AKIA1234567890ABCDEF"  # AWS secret key h...
gitleaks	Run GitLeaks	Secret:      AKIA1234567890ABCDEF
gitleaks	Run GitLeaks	RuleID:      generic-api-key
gitleaks	Run GitLeaks	File:        test.py
gitleaks	Run GitLeaks	Line:        2
gitleaks	Run GitLeaks	Commit:      1dcd2bd89ab6cbe6a40b19bf60be2e3480acce08
gitleaks	Run GitLeaks	Author:      RameshMarujolla
gitleaks	Run GitLeaks	Email:       ramesh.btp69@gmail.com
gitleaks	Run GitLeaks	Date:        2026-06-26T16:35:36Z
gitleaks	Run GitLeaks	Fingerprint: 1dcd2bd89ab6cbe6a40b19bf60be2e3480acce08:test.py:generic-api-key:2
gitleaks	Run GitLeaks	Link:        https://github.com/RameshMarujolla/GitLeaks/blob/1dcd2bd89ab6cbe6a40b19bf60be2e3480acce08/test.py#L2
gitleaks	Run GitLeaks	INF 9 commits scanned.
gitleaks	Run GitLeaks	INF scanned ~7284 bytes (7.28 KB) in 33.2ms
gitleaks	Run GitLeaks	WRN leaks found: 1
```

### Console output AFTER redaction (safe)

After adding `--redact`, the secret is masked in the logs:

```
gitleaks	Run GitLeaks	Finding:     aws_access_key = "REDACTED"  # AWS secret key h...
gitleaks	Run GitLeaks	Secret:      REDACTED
gitleaks	Run GitLeaks	RuleID:      generic-api-key
gitleaks	Run GitLeaks	File:        test.py
gitleaks	Run GitLeaks	Line:        2
gitleaks	Run GitLeaks	WRN leaks found: 1
```

### SARIF artifact snippet BEFORE redaction (security risk)

The raw secret was also embedded in the downloaded artifact before redaction:

```json
"snippet": {
  "text": "AKIA1234567890ABCDEF"
}
```

### SARIF artifact snippet AFTER redaction (safe)

```json
"snippet": {
  "text": "REDACTED"
}
```

### SARIF upload success

```
gitleaks	Upload SARIF to GitHub Security tab	Successfully uploaded results
gitleaks	Upload SARIF to GitHub Security tab	Analysis upload status is complete.
```

### Artifact upload success

```
gitleaks	Upload GitLeaks report artifact	Artifact gitleaks-report has been successfully uploaded!
```

## 8. Verification Steps

1. Push the fixed workflow to `main`.
2. Open **Actions** → **GitLeaks** workflow.
3. Confirm:
   - `Run GitLeaks` executes and reports the finding.
   - `Upload SARIF to GitHub Security tab` shows `Successfully uploaded results`.
   - `Upload GitLeaks report artifact` shows `Artifact gitleaks-report has been successfully uploaded!`.
4. Go to **Security** → **Code scanning** to view the alert.
5. Download the artifact from the workflow run and confirm the raw secret is not present in `results.sarif`.

## 9. Result

- ✅ SARIF report is generated.
- ✅ Findings are uploaded to the GitHub Security tab.
- ✅ Artifact is uploaded and accessible.
- ✅ Raw secrets are redacted in logs and artifact.
- ✅ Node 20 / CodeQL v3 deprecation warnings are cleared.

## 10. Commits Applied

- `3009cf6` — Fix gitleaks workflow (remove trailing whitespace, update upload-sarif to v4, guard uploads).
- `b3191a5` — Update actions to Node 24 versions (`actions/checkout@v7`, `actions/upload-artifact@v7`).
- `8bcc8c3` — Redact secrets from gitleaks SARIF report and logs.
