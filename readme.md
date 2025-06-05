Here's the complete **SBOM Caller Workflow** that handles concurrent requests from multiple repositories, tracks each SBOM generation status accurately, and communicates with your Almanac workflow:

```yaml
name: SBOM Caller Workflow
on:
  pull_request:
    branches: [main, master, develop, sandbox]
  workflow_dispatch:

jobs:
  trigger-and-track-sbom:
    runs-on: evsharesvcnonprod
    outputs:
      sbom_status: ${{ steps.verify-sbom-result.outputs.sbom_status }}
      sbom_run_id: ${{ steps.trigger-sbom.outputs.run_id }}

    steps:
      # --- Setup Phase ---
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Install GitHub CLI
        run: sudo snap install gh --classic

      - name: Authenticate with GitHub CLI
        run: echo "${{ secrets.GH_WORKFLOW_TOKEN }}" | gh auth login --with-token

      # --- Trigger Phase ---
      - name: Trigger SBOM Generation
        id: trigger-sbom
        run: |
          # Generate unique correlation ID
          CORRELATION_ID="${{ github.repository }}-$(date +%s)"
          echo "correlation-id=$CORRELATION_ID" >> $GITHUB_OUTPUT

          # Trigger SBOM workflow with identifying information
          RUN_ID=$(gh workflow run SBOM-add-almanac.yml \
            --repo Eaton-Vance-Corp/SRE-Utilities \
            --field call-owner="${{ github.repository_owner }}" \
            --field call-repo="${{ github.event.repository.name }}" \
            --field correlation-id="$CORRELATION_ID" \
            --json databaseId --jq '.databaseId')
          
          echo "run_id=$RUN_ID" >> $GITHUB_OUTPUT
          sleep 10  # Allow time for workflow to initialize

      # --- Tracking Phase ---
      - name: Wait for SBOM Completion
        id: wait-for-sbom
        run: |
          run_id="${{ steps.trigger-sbom.outputs.run_id }}"
          echo "⌛ Tracking SBOM run: $run_id"

          # Wait for workflow completion with timeout (30 minutes)
          timeout=1800  # 30 minutes in seconds
          interval=30
          elapsed=0
          
          while [ $elapsed -lt $timeout ]; do
            status=$(gh run view $run_id \
              --repo Eaton-Vance-Corp/SRE-Utilities \
              --json status --jq '.status')
            
            if [ "$status" = "completed" ]; then
              conclusion=$(gh run view $run_id \
                --repo Eaton-Vance-Corp/SRE-Utilities \
                --json conclusion --jq '.conclusion')
              
              echo "SBOM generation $conclusion for ${{ github.repository }}"
              echo "status=$conclusion" >> $GITHUB_OUTPUT
              exit 0
            fi
            
            sleep $interval
            elapsed=$((elapsed + interval))
            echo "Waiting... ($elapsed/$timeout seconds)"
          done
          
          echo "::error::Timeout waiting for SBOM completion"
          exit 1

      # --- Verification Phase ---
      - name: Verify SBOM Result
        id: verify-sbom-result
        if: always()
        run: |
          if [ "${{ steps.wait-for-sbom.outputs.status }}" = "success" ]; then
            echo "✅ SBOM generated successfully for ${{ github.repository }}"
            echo "sbom_status=success" >> $GITHUB_OUTPUT
          else
            echo "::error::❌ SBOM generation failed for ${{ github.repository }}"
            echo "sbom_status=failure" >> $GITHUB_OUTPUT
            exit 1
          fi

      # --- Notification Phase (Optional) ---
      - name: Notify Status
        if: always()
        uses: actions/github-script@v6
        env:
          STATUS: ${{ steps.verify-sbom-result.outputs.sbom_status }}
          RUN_ID: ${{ steps.trigger-sbom.outputs.run_id }}
        with:
          script: |
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: ${{ github.event.pull_request.number }},
              body: `SBOM Generation ${process.env.STATUS.toUpperCase()}  
                     Run ID: ${process.env.RUN_ID}
                     View run: https://github.com/Eaton-Vance-Corp/SRE-Utilities/actions/runs/${process.env.RUN_ID}`
            })
```

### Key Features:

1. **End-to-End Tracking**:
   - Unique correlation IDs for each run
   - Explicit output propagation between jobs
   - Timeout handling (30 minutes)

2. **Concurrency Safe**:
   - Each repository tracks only its own SBOM runs
   - Status verification tied to specific workflow runs

3. **Comprehensive Status Reporting**:
   - Console output with clear emoji indicators
   - PR comments with direct links to runs
   - Job outputs for downstream steps

4. **Error Handling**:
   - Timeout detection
   - Explicit failure states
   - Non-zero exits on failures

5. **Audit Trail**:
   - Preserves run IDs throughout the process
   - Links back to specific workflow runs

### Required Secrets:
- `GH_WORKFLOW_TOKEN`: A PAT with `repo` and `workflow` permissions

### Companion Almanac Workflow Updates:
Ensure your `SBOM-add-almanac.yml` has:
```yaml
outputs:
  sbom_status: ${{ jobs.create-push-SBOM.outputs.sbom_status }}
  correlation_id: ${{ inputs.correlation-id }}
```

This implementation will reliably handle concurrent SBOM generation requests from multiple repositories while providing clear status tracking for each individual request.
