To **validate the OIDC Issuer `https://token.actions.githubusercontent.com`** in the context of GitHub Actions and Octopus Deploy, you’re essentially verifying that:

* GitHub issues the token from this issuer
* The token is trusted and valid
* The claims in the token (issuer, audience, subject) match what Octopus expects

Here’s how to do that:

---

## ✅ Step-by-Step: Validate the OIDC Issuer in GitHub Actions

---

### **1. Create a GitHub Actions test workflow**

This workflow will generate a token and output its decoded contents:

```yaml
name: Validate GitHub OIDC Token

on:
  workflow_dispatch:

jobs:
  oidc-test:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - name: Request OIDC Token
        id: oidc
        run: |
          echo "Fetching OIDC token..."
          TOKEN=$(curl -s -H "Authorization: Bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=api.octopus.com")
          echo "TOKEN=$TOKEN" >> $GITHUB_ENV
          echo "$TOKEN" > token.json

      - name: Decode JWT (for debugging)
        run: |
          TOKEN_VALUE=$(cat token.json | jq -r '.value')
          echo "$TOKEN_VALUE" | cut -d '.' -f2 | base64 -d | jq .
```

> 🔧 Make sure to enable `id-token: write` in `permissions`. This is required for GitHub OIDC.

---

### **2. Run the workflow and inspect output**

After the workflow runs, you'll see output like this in the logs:

```json
{
  "iss": "https://token.actions.githubusercontent.com",
  "aud": "api.octopus.com",
  "sub": "repo:your-org/your-repo:ref:refs/heads/main",
  ...
}
```

✅ This confirms:

| Field | Expected Value                                  |
| ----- | ----------------------------------------------- |
| `iss` | `https://token.actions.githubusercontent.com` ✅ |
| `aud` | `api.octopus.com` (or your custom audience) ✅   |
| `sub` | GitHub repo/ref identity used in OIDC filters ✅ |

---

### **3. Cross-check with Octopus OIDC Config**

* Make sure your **Octopus OIDC account** is configured with:

  * `Issuer: https://token.actions.githubusercontent.com`
  * `Audience: api.octopus.com` (or what you set in the token request)
  * `Subject filter`: matches `sub` claim (`your-org/*` or specific repo)

---

## 🧪 Optional: Use `jwt.io` to decode manually

Paste the token into [https://jwt.io](https://jwt.io) to inspect the issuer (`iss`) claim visually.

---

Let me know if you want help parsing the full token or matching the claims with your Octopus setup.
