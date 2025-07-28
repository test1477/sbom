Thanks! To **disable SSL verification** (i.e., set `verify=False` in all `requests.get()` calls), you can either:

1. **Pass `verify=False` in every `requests.get()` call**, or
2. **Use a session object and disable it globally.**

I'll update the original CSV script with **option 1** (simplest):

---

### ✅ Updated Script (with `verify=False` for SSL):

```python
import requests
import csv

# ==== CONFIGURATION ====
OCTOPUS_API_KEY = "API-XXXXXXXXXXXXXXXXXXXXXXXXXX"  # Replace with your API Key
OCTOPUS_URL = "https://your-octopus.octopus.app"   # Replace with your Octopus URL
HEADERS = {
    "X-Octopus-ApiKey": OCTOPUS_API_KEY,
    "Content-Type": "application/json"
}
CSV_FILE = "last_deployments.csv"
VERIFY_SSL = False  # Set to False to disable SSL certificate verification

def get_projects():
    url = f"{OCTOPUS_URL}/api/projects/all"
    response = requests.get(url, headers=HEADERS, verify=VERIFY_SSL)
    response.raise_for_status()
    return response.json()

def get_latest_deployment(project_id):
    url = f"{OCTOPUS_URL}/api/deployments?projects={project_id}&take=1&order=desc"
    response = requests.get(url, headers=HEADERS, verify=VERIFY_SSL)
    response.raise_for_status()
    deployments = response.json().get("Items", [])
    return deployments[0] if deployments else None

def get_release(release_id):
    url = f"{OCTOPUS_URL}/api/releases/{release_id}"
    response = requests.get(url, headers=HEADERS, verify=VERIFY_SSL)
    response.raise_for_status()
    return response.json()

def get_environment(env_id):
    url = f"{OCTOPUS_URL}/api/environments/{env_id}"
    response = requests.get(url, headers=HEADERS, verify=VERIFY_SSL)
    response.raise_for_status()
    return response.json()

def get_task_status(task_id):
    url = f"{OCTOPUS_URL}/api/tasks/{task_id}"
    response = requests.get(url, headers=HEADERS, verify=VERIFY_SSL)
    response.raise_for_status()
    return response.json()

def main():
    projects = get_projects()
    rows = []

    for project in projects:
        project_id = project["Id"]
        project_name = project["Name"]

        deployment = get_latest_deployment(project_id)
        if not deployment:
            continue

        try:
            release = get_release(deployment["ReleaseId"])
            environment = get_environment(deployment["EnvironmentId"])
            task = get_task_status(deployment["TaskId"])

            rows.append({
                "project_name": project_name,
                "environment": environment["Name"],
                "release_version": release["Version"],
                "deployed_at": deployment["Created"],
                "deployment_status": task["State"]
            })
        except Exception as e:
            print(f"Error processing {project_name}: {e}")

    # Write to CSV
    with open(CSV_FILE, mode='w', newline='', encoding='utf-8') as csvfile:
        fieldnames = ["project_name", "environment", "release_version", "deployed_at", "deployment_status"]
        writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(rows)

    print(f"Deployment report saved to: {CSV_FILE}")

if __name__ == "__main__":
    main()
```

---

### ⚠️ Note on `verify=False`:

Disabling SSL verification is insecure and should **only be used in internal/testing environments**. Avoid this in production unless absolutely necessary.

Let me know if you want to switch to **option 2 (requests session with `verify=False`)** or also suppress **InsecureRequestWarning** logs.
