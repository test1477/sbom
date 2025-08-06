To use the **`server_tasks` output** from `deploy-release-action@v2` and wait for deployment completion using **`await-task-action@v3`**, follow this structured workflow:

---

### **Step-by-Step Workflow**
#### **1. Deploy a Release & Capture `server_tasks`**
```yaml
- name: Deploy Release in Octopus 🚀
  id: deploy_release  # Assign an ID to reference outputs later
  uses: OctopusDeploy/deploy-release-action@v2
  env:
    OCTOPUS_URL: ${{ secrets.OCTOPUS_URL }}
    OCTOPUS_API_KEY: ${{ secrets.OCTOPUS_API_KEY }}
  with:
    space: 'Your-Space'
    project: 'Your-Project'
    release_number: '1.0.0'
    environments: 'Production'  # Comma-separated for multiple environments
```

#### **2. Wait for Deployment Completion Using `await-task-action`**
```yaml
- name: Await Octopus Deployment Completion ⏳
  uses: OctopusDeploy/await-task-action@v3
  env:
    OCTOPUS_URL: ${{ secrets.OCTOPUS_URL }}
    OCTOPUS_API_KEY: ${{ secrets.OCTOPUS_API_KEY }}
    OCTOPUS_SPACE: 'Your-Space'
  with:
    # Extract the first serverTaskId from the JSON output
    server_task_id: ${{ fromJson(steps.deploy_release.outputs.server_tasks)[0].serverTaskId }}
```

---

### **Key Details**
1. **`server_tasks` Output** (from `deploy-release-action@v2`):
   - Returns a **JSON array** of objects like:
     ```json
     [
       { "serverTaskId": "ServerTasks-123", "environmentName": "Production" },
       { "serverTaskId": "ServerTasks-456", "environmentName": "Staging" }
     ]
     ```
   - Accessible via `steps.[DEPLOY_STEP_ID].outputs.server_tasks`.

2. **`await-task-action@v3`**:
   - Waits for the specified `server_task_id` to complete.
   - Fails the job if the Octopus task fails.

---

### **Advanced Use Cases**
#### **Wait for Multiple Environments**
If you deploy to multiple environments (e.g., `Production,Staging`), loop through `server_tasks`:
```yaml
- name: Await All Deployments
  run: |
    TASKS='${{ steps.deploy_release.outputs.server_tasks }}'
    for task in $(echo "$TASKS" | jq -r '.[].serverTaskId'); do
      echo "Waiting for task: $task"
      octo wait-for-task --task-id="$task" --server="$OCTOPUS_URL" --apiKey="$OCTOPUS_API_KEY"
    done
  env:
    OCTOPUS_URL: ${{ secrets.OCTOPUS_URL }}
    OCTOPUS_API_KEY: ${{ secrets.OCTOPUS_API_KEY }}
```

#### **Extract Deployment Status**
```yaml
- name: Log Deployment Status
  run: |
    TASK_ID=${{ fromJson(steps.deploy_release.outputs.server_tasks)[0].serverTaskId }}
    STATUS=$(octo get-task --task-id="$TASK_ID" --server="$OCTOPUS_URL" --apiKey="$OCTOPUS_API_KEY" --format=json | jq -r '.State')
    echo "Deployment Status: $STATUS"
```

---

### **Troubleshooting**
- **Error**: `Invalid server_task_id`  
  → Verify the JSON path: `steps.[STEP_ID].outputs.server_tasks` must be valid.
- **Error**: `Task failed in Octopus`  
  → Check the Octopus Deploy task logs for details.

---

This approach ensures **real-time tracking** of Octopus deployments in GitHub Actions. Let me know if you need help customizing it further! 🎯
