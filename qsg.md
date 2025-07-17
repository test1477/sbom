Great question. You’re looking to:

1. **Highlight deprecation warnings clearly**
2. **Detect and list packages blocked by JFrog Xray or curation policies**
3. **Include policy reasons for blocks, if possible**

Let’s walk through what’s possible and how to approach it in your **CI pipeline using GitHub Actions + JFrog CLI + npm**.

---

## ✅ 1. **Highlight Deprecation Warnings**

While `npm ci` will show deprecation warnings, it mixes them with other logs.

### ✅ Solution: Capture and filter logs

You can redirect logs and grep for `deprecated`:

```yaml
- name: Install Dependencies (Capture Deprecations)
  run: |
    npm ci --ignore-scripts > npm-log.txt 2>&1 || true
    echo "=== Deprecation Warnings ==="
    grep "deprecated" npm-log.txt || echo "No deprecation warnings"
```

> 💡 This will not fail your build — it just logs cleanly what is deprecated.

---

## ✅ 2. **List Packages Blocked by JFrog Xray or Curation**

JFrog Xray can block packages based on:

* **Security policies** (e.g., CVEs)
* **License rules**
* **Custom curation**

When a package is **blocked**, the install typically fails with an HTTP 403 or a specific JFrog error message.

### ✅ Solution: Use `jf audit` before install

Add this step before `npm ci`:

```yaml
- name: JFrog Xray Audit
  run: |
    jf audit --type npm --format json > xray-result.json || true
    echo "=== Xray Policy Violations ==="
    jq '.violations[] | {component: .components[0], policy: .policy, severity: .severity, reason: .reason}' xray-result.json || echo "No policy violations"
```

> Requires Xray integration on your JFrog Artifactory account and `jf` configured properly.

### ✅ Optional: Fail on certain policy severities

```yaml
- name: Fail if Critical Xray Violations
  run: |
    CRITICAL_COUNT=$(jq '[.violations[] | select(.severity == "High" or .severity == "Critical")] | length' xray-result.json)
    if [ "$CRITICAL_COUNT" -gt 0 ]; then
      echo "Critical/high policy violations found: $CRITICAL_COUNT"
      exit 1
    fi
```

---

## ✅ Combined Strategy in Your GitHub Actions Workflow

### Example YAML snippet:

```yaml
- name: JFrog Xray Audit
  run: |
    jf audit --type npm --format json > xray-result.json || true
    echo "=== JFrog Xray Policy Violations ==="
    jq '.violations[] | {component: .components[0], policy: .policy, severity: .severity, reason: .reason}' xray-result.json || echo "No violations"

- name: Install npm dependencies (capture deprecations)
  run: |
    npm ci --ignore-scripts > npm-log.txt 2>&1 || true
    echo "=== Deprecation Warnings ==="
    grep "deprecated" npm-log.txt || echo "No deprecation warnings"

- name: Fail if Critical Policy Blocks
  run: |
    CRITICAL_COUNT=$(jq '[.violations[] | select(.severity == "High" or .severity == "Critical")] | length' xray-result.json)
    if [ "$CRITICAL_COUNT" -gt 0 ]; then
      echo "🔥 Critical/high policy violations found!"
      jq '.violations[] | select(.severity == "High" or .severity == "Critical")' xray-result.json
      exit 1
    fi
```

---

## 🧠 Summary

| Goal                        | Solution                                                  |
| --------------------------- | --------------------------------------------------------- |
| Highlight deprecated pkgs   | Capture `npm ci` logs and grep for "deprecated"           |
| Show Xray policy blocks     | Use `jf audit --format json` and parse with `jq`          |
| Know which policy was hit   | Parse `.policy`, `.reason`, `.severity` from audit result |
| Fail on critical violations | Count and exit if `severity == High/Critical`             |

---

Would you like me to give you a plug-and-play version of this YAML for your current pipeline?
