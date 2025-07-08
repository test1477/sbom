The error occurs because you're trying to use the `jf npm` command without first configuring the JFrog CLI with your Artifactory details. Here's how to properly set it up in your GitHub Actions workflow:

### Complete Fixed Workflow

```yaml
name: CI with JFrog npm

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
    
    - name: Install JFrog CLI
      uses: jfrog/setup-jfrog-cli@v3
    
    - name: Configure JFrog CLI
      run: |
        jf c add artifactory \
          --url "${{ secrets.JFROG_URL }}" \
          --access-token "${{ secrets.QSG_JFROG_API_KEY }}" \
          --interactive=false
      env:
        JFROG_URL: "https://frigate.jfrog.io/artifactory"
    
    - name: Configure npm registry
      run: |
        jf npm config \
          --global \
          --repo npm-remote
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run tests
      run: npm test
```

### Key Fixes:

1. **Added JFrog CLI configuration step**:
   ```yaml
   - name: Configure JFrog CLI
     run: |
       jf c add artifactory \
         --url "${{ secrets.JFROG_URL }}" \
         --access-token "${{ secrets.QSG_JFROG_API_KEY }}" \
         --interactive=false
   ```

2. **Properly structured the npm configuration**:
   ```yaml
   - name: Configure npm registry
     run: |
       jf npm config \
         --global \
         --repo npm-remote
   ```

### Alternative Solution (without JFrog CLI)

If you prefer not to use the JFrog CLI, here's a simpler version using direct npm configuration:

```yaml
- name: Configure npm registry
  run: |
    echo "registry=https://frigate.jfrog.io/artifactory/api/npm/npm-remote/" > ~/.npmrc
    echo "//frigate.jfrog.io/artifactory/api/npm/npm-remote/:_auth=${JFROG_API_KEY_BASE64}" >> ~/.npmrc
    echo "always-auth=true" >> ~/.npmrc
  env:
    JFROG_API_KEY_BASE64: ${{ toJSON(secrets.QSG_JFROG_API_KEY) | base64 }}
```

### Required Secrets:

Make sure these secrets are configured in your GitHub repository:
1. `QSG_JFROG_API_KEY` - Your JFrog Artifactory API key
2. `JFROG_URL` - Your Artifactory URL (e.g., "https://frigate.jfrog.io/artifactory")

This should resolve the "no config file was found" error and allow your workflow to properly authenticate with Artifactory.
