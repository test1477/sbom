Perfect 👍 thanks for clarifying.

So we need:

1. **Security tab integration** → this requires **GitHub Advanced Security (GHAS)** to be enabled at org or repo level. Otherwise CodeQL results **will not show up in the Security tab**.
2. **Language auto-detection** → CodeQL can detect the language(s) of a repo automatically, so you don’t need to hardcode a matrix for every language.

Here’s a **consolidated workflow** that will:

* Run **CodeQL scanning** (auto-detects repo language(s))
* Run **Dependency Review**
* Trigger on **PRs to the default branch**
* Upload results to the **Security tab** (requires GHAS)

```yaml
name: Security Scan (CodeQL + Dependency Review)

on:
  pull_request:
    branches:
      - $default-branch   # Runs only on PRs to the default branch

jobs:
  codeql:
    name: CodeQL Analysis
    runs-on: ubuntu-latest
    permissions:
      security-events: write     # needed for Security tab
      actions: read
      contents: read

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: auto    # Auto-detects languages in the repo

      - name: Autobuild
        uses: github/codeql-action/autobuild@v3

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3

  dependency-review:
    name: Dependency Review
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write  # annotates PR with issues

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Dependency Review
        uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: high    # block PRs with high severity vulnerabilities
```

---

### 🔑 Key Points

* `languages: auto` makes CodeQL detect **only the languages present** (so if repo has Python → it scans only Python).
* With **GHAS enabled**, results from `analyze` will appear in the **Security tab** + inline PR checks.
* Without GHAS, results are only visible in the workflow logs/artifacts.
* Dependency Review runs regardless of GHAS, since it’s free.

---

👉 Since you want to **roll this out org-wide**, I can show you how to make it a **reusable workflow** (defined once in your `.github` org repo and referenced by all repos), so you don’t have to maintain the YAML in every repo.

Do you want me to prepare the **reusable workflow version** for org rollout?
