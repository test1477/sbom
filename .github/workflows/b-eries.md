Thanks for pasting your full workflow 👌 — I see a few issues that are making it behave oddly and always “PASS”:

---

### 🔎 Problems in the YAML you pasted

1. **YAML formatting/indentation** is broken (steps don’t have `- name:` etc., env block has typos).
2. `languages: auto` isn’t valid → you correctly switched to a `matrix`.
3. You’re running `github/codeql-action/analyze@v3` (good) but then **adding a shell step** that only echoes text, and never fails the job.
4. SARIF upload is already done by the `analyze@v3` action — your manual SARIF checks aren’t needed unless you want to block PRs.

---

### ✅ Fixed + Improved Workflow

Here’s a clean version that:

* Runs on `push` and `pull_request` to `main`.
* Runs **Java + Python** CodeQL analysis in matrix.
* Uploads results to **Security → Code scanning alerts** automatically.
* Adds a **custom check step** that blocks the PR if **high/critical vulnerabilities** are found.

```yaml
name: Security Scan (CodeQL + Dependency Review)

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:
    inputs:
      octopus_deployment_id:
        description: 'Octopus Deploy Deployment ID'
        required: false
        type: string
      change_request_id:
        description: 'Cherwell Change Request ID'
        required: false
        type: string

env:
  OCTOPUS_DEPLOYMENT_ID: ${{ github.event.inputs.octopus_deployment_id || 'manual-trigger' }}
  CHANGE_REQUEST_ID: ${{ github.event.inputs.change_request_id || 'unknown' }}

jobs:
  codeql:
    name: CodeQL Analysis (${{ matrix.language }})
    runs-on: ubuntu-latest

    strategy:
      fail-fast: true
      matrix:
        language: [ 'java', 'python' ]
        # CodeQL also supports: cpp, csharp, go, javascript, ruby

    permissions:
      security-events: write   # needed for Security tab
      actions: read
      contents: read

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
          queries: security-and-quality

      - name: Autobuild
        uses: github/codeql-action/autobuild@v3

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: "/language:${{ matrix.language }}"
          output: results

      # Custom check for high/critical vulnerabilities
      - name: Check for High/Critical Vulnerabilities
        run: |
          SARIF_FILES=$(find results -name "*.sarif" 2>/dev/null || true)
          if [ -n "$SARIF_FILES" ]; then
            echo "SARIF results found: $SARIF_FILES"

            # Count only "error" severity (high/critical)
            alerts=$(jq '[.runs[].results[] 
                          | select(.level == "error")] 
                          | length' $SARIF_FILES)

            echo "High/Critical vulnerabilities detected: $alerts"

            if [ "$alerts" -gt 0 ]; then
              echo "Blocking PR because high/critical vulnerabilities were found"
              exit 1
            fi
          else
            echo "No SARIF files found – check CodeQL job setup"
          fi
```

---

### 🔐 Behavior

* **Security tab** → always shows all results.
* **PR checks** → will go red **only if High/Critical vulnerabilities are detected**.
* **Low/Medium** → still reported in Security tab, but workflow won’t block merge.

---

👉 Do you also want me to fold **Dependency Review** (for third-party libs) into this same workflow so you have a single consolidated pipeline, or keep CodeQL and Dependency Review as separate jobs?
