Here's a draft email you can send to Octopus Support regarding your issue:

---

**Subject:** 403 Error When Creating Release Using Omega Service Account API Key  

**Body:**  
Dear Octopus Support Team,  

I am reaching out regarding an issue we’re encountering while trying to create a release using the Omega service account API key. We’ve successfully logged in to Octopus using `OctopusDeploy/login@v1`, but when attempting to create a release with `OctopusDeploy/create-release-action@v3`, we receive a **403 error**.  

### Details:  
- **Octopus Server:** `https://ev-octo-server.octopus.app`  
- **Space:** `Databases`  
- **Project:** `githubdeploytestapp`  
- **Action Workflow Snippet:**  
  ```yaml
  jobs:
    deploy:
      runs-on: ubuntu-latest
      steps:
        - name: Login to Octopus
          uses: OctopusDeploy/login@v1
          with:
            server: https://ev-octo-server.octopus.app
            api_key: ${{ secrets.DATA_API_KEY }}

        - name: Create Release
          uses: OctopusDeploy/create-release-action@v3
          with:
            project: 'githubdeploytestapp'
            server: 'https://ev-octo-server.octopus.app'
            api_key: ${{ secrets.DATA_API_KEY }}
            space: 'Databases'
  ```

### Observations:  
1. The login step succeeds, but the release creation fails with a **403 (Forbidden)**.  
2. The API key (`DATA_API_KEY`) has been verified and works for other operations.  

### Questions:  
1. Are there specific permissions the Omega service account requires for `create-release-action` beyond what’s needed for login?  
2. Could this be related to space/project permissions or a misconfigured API key scope?  
3. Are there known issues with the `create-release-action@v3` and Omega accounts?  

We’ve double-checked the project name, space, and API key but would appreciate guidance on troubleshooting this further.  

Thank you for your help!  
[Your Name]  
[Your Contact/Organization Details]  

---  

### Notes:  
- Fix the typo in `ubuntu-latest` (your snippet says `ub**u**untu-latest`).  
- Ensure the `DATA_API_KEY` secret is correctly stored in GitHub Secrets.  
- If applicable, mention whether this worked previously or is a new setup.  

Let me know if you'd like to adjust the tone or add more context!
