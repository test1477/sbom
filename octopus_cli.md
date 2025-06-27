Here's the **full step-by-step process** to:

✅ Download the latest **Octopus CLI**
✅ Upload it to your **JFrog Artifactory**
✅ Use it in your **GitHub Actions workflow**

---

## ✅ Step 1: Download the Latest Octopus CLI on Your Machine

### 🔧 Terminal Command (for Linux x64 version)

```bash
curl -L "$(curl -s https://api.github.com/repos/OctopusDeploy/OctopusCLI/releases/latest | grep browser_download_url | grep linux-x64.tar.gz | cut -d '"' -f 4)" -o OctopusTools-linux-x64.tar.gz
```

> 💡 You can change `linux-x64.tar.gz` to `osx-x64.tar.gz` or `win-x64.zip` if needed.

---

## ✅ Step 2: Upload the Octopus CLI Archive to JFrog Artifactory

### 🛠️ Example Upload Command

```bash
curl -u <JFROG_USERNAME>:<JFROG_API_KEY> \
  -T OctopusTools-linux-x64.tar.gz \
  "https://yourcompany.jfrog.io/artifactory/tools-local/octopus/OctopusTools-linux-x64.tar.gz"
```

> 🔐 Replace with:
>
> * `yourcompany.jfrog.io` = your Artifactory domain
> * `tools-local/octopus/` = your target repo/path
> * Use an **API key** or **access token** instead of a password

---

## ✅ Step 3: Use Octopus CLI in GitHub Actions from JFrog

Here’s a full working example of a GitHub Actions workflow that:

1. Downloads the CLI from JFrog
2. Extracts it
3. Uses it in a step

### 📄 `.github/workflows/use-octopus-cli.yml`

```yaml
name: Use Octopus CLI from JFrog

on:
  push:

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Download Octopus CLI from JFrog
        run: |
          curl -u ${{ secrets.JFROG_USERNAME }}:${{ secrets.JFROG_API_KEY }} \
            -L "https://yourcompany.jfrog.io/artifactory/tools-local/octopus/OctopusTools-linux-x64.tar.gz" \
            -o octo.tar.gz

          mkdir octo
          tar -xzf octo.tar.gz -C octo
          echo "$PWD/octo" >> $GITHUB_PATH

      - name: Check Octopus CLI version
        run: octo version
```

> 🛡️ Save your JFrog credentials as [**GitHub secrets**](https://docs.github.com/en/actions/security-guides/encrypted-secrets):
>
> * `JFROG_USERNAME`
> * `JFROG_API_KEY`

---

## ✅ Summary

| Step | Description                                                           |
| ---- | --------------------------------------------------------------------- |
| 1️⃣  | Download the latest CLI from GitHub                                   |
| 2️⃣  | Upload the `.tar.gz` to JFrog                                         |
| 3️⃣  | Use the CLI in GitHub Actions by downloading and adding it to `$PATH` |

---

Would you like me to generate:

* A shell script to automate Steps 1–2?
* A version of this workflow for Windows runners?

Let me know!
