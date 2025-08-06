Yes! The `OctopusDeploy/deploy-release-action@v2` **automatically includes the `server_tasks` output**, but you need to:

1. **Capture it** by giving the step an `id`
2. **Reference it correctly** in subsequent steps

Here's how to properly use it:

### Complete Working Example
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # Step 1: Deploy and capture server_tasks
      - name: Deploy Release
        id: octopus_deploy  # THIS IS CRUCIAL FOR OUTPUTS
        uses: OctopusDeploy/deploy-release-action@v2
        env:
          OCTOPUS_URL: ${{ secrets.OCTOPUS_URL }}
          OCTOPUS_API_KEY: ${{ secrets.OCTOPUS_API_KEY }}
        with:
          space: 'MySpace'
          project: 'MyProject'
          release_number: '1.0.0'
          environments: 'Production'  # Can be comma-separated list

      # Step 2: Debug the output (optional but recommended)
      - name: Show server_tasks output
        run: |
          echo "Raw server_tasks: ${{ steps.octopus_deploy.outputs.server_tasks }}"
          echo "Formatted:"
          echo '${{ steps.octopus_deploy.outputs.server_tasks }}' | jq .

      # Step 3: Wait for deployment completion
      - name: Wait for Octopus Deployment
        uses: OctopusDeploy/await-task-action@v3
        env:
          OCTOPUS_URL: ${{ secrets.OCTOPUS_URL }}
          OCTOPUS_API_KEY: ${{ secrets.OCTOPUS_API_KEY }}
        with:
          server_task_id: ${{ fromJson(steps.octopus_deploy.outputs.server_tasks)[0].serverTaskId }}
```

### Key Points:

1. **Mandatory `id` field**  
   You must give your deployment step an `id` (e.g., `octopus_deploy`) to reference outputs.

2. **Output Format**  
   The `server_tasks` output looks like this:
   ```json
   [
     {
       "serverTaskId": "ServerTasks-123",
       "environmentName": "Production"
     }
   ]
   ```

3. **Handling Multiple Environments**  
   If you deploy to multiple environments (e.g., `environments: 'Production,Staging'`), the array will contain multiple entries.

### Advanced Usage
For multiple environments, you can process all tasks:
```yaml
- name: Wait for All Deployments
  run: |
    for task in $(echo '${{ steps.octopus_deploy.outputs.server_tasks }}' | jq -r '.[].serverTaskId'); do
      echo "Waiting for task $task"
      octo wait-for-task --task-id="$task" --server="$OCTOPUS_URL" --apiKey="$OCTOPUS_API_KEY"
    done
  env:
    OCTOPUS_URL: ${{ secrets.OCTOPUS_URL }}
    OCTOPUS_API_KEY: ${{ secrets.OCTOPUS_API_KEY }}
```

### Troubleshooting
If you still see JSON errors:
1. **Verify the deployment succeeded** (check previous step logs)
2. **Check for output existence** with the debug step
3. **Ensure proper escaping** when using `fromJson`

This approach gives you full visibility and control over Octopus deployments in your GitHub Actions workflow.
