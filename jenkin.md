# Python Script to Get Jenkins Job Inventory with API Token

Here's a Python script that uses the Jenkins API to retrieve all jobs and their dependencies:

```python
import requests
import json
from requests.auth import HTTPBasicAuth

# Jenkins configuration
JENKINS_URL = 'https://your-jenkins-instance.com'
JENKINS_USER = 'your-username'  # Or use API token as username
JENKINS_API_TOKEN = 'your-api-token'

def get_all_jobs_with_dependencies():
    # Set up authentication
    auth = HTTPBasicAuth(JENKINS_USER, JENKINS_API_TOKEN)
    
    # Get all jobs
    jobs_url = f"{JENKINS_URL}/api/json?tree=jobs[name,url]"
    response = requests.get(jobs_url, auth=auth)
    
    if response.status_code != 200:
        print(f"Failed to fetch jobs: {response.status_code} - {response.text}")
        return []
    
    jobs_data = response.json()
    job_inventory = []
    
    # Process each job
    for job in jobs_data['jobs']:
        job_name = job['name']
        job_url = job['url']
        
        # Get job details including upstream/downstream projects
        job_details_url = f"{job_url}api/json?tree=upstreamProjects[name],downstreamProjects[name],buildable"
        details_response = requests.get(job_details_url, auth=auth)
        
        if details_response.status_code == 200:
            details = details_response.json()
            upstream = [p['name'] for p in details.get('upstreamProjects', [])]
            downstream = [p['name'] for p in details.get('downstreamProjects', [])]
            buildable = details.get('buildable', False)
            
            job_info = {
                'name': job_name,
                'url': job_url,
                'upstream': upstream,
                'downstream': downstream,
                'buildable': buildable
            }
            job_inventory.append(job_info)
        else:
            print(f"Failed to get details for {job_name}: {details_response.status_code}")
    
    return job_inventory

def main():
    inventory = get_all_jobs_with_dependencies()
    
    # Print results
    print(f"Found {len(inventory)} jobs:")
    for job in inventory:
        print(f"\nJob: {job['name']}")
        print(f"URL: {job['url']}")
        print(f"Upstream dependencies: {', '.join(job['upstream']) or 'None'}")
        print(f"Downstream dependencies: {', '.join(job['downstream']) or 'None'}")
        print(f"Buildable: {job['buildable']}")
    
    # Optionally save to JSON file
    with open('jenkins_job_inventory.json', 'w') as f:
        json.dump(inventory, f, indent=2)
    print("\nInventory saved to jenkins_job_inventory.json")

if __name__ == "__main__":
    main()
```

## How to Use This Script:

1. Replace the configuration variables at the top:
   - `JENKINS_URL`: Your Jenkins instance URL
   - `JENKINS_USER`: Your username (or use API token as username)
   - `JENKINS_API_TOKEN`: Your API token (generate from Jenkins > User > Configure > API Token)

2. Run the script:
   ```
   python jenkins_inventory.py
   ```

3. The script will:
   - Fetch all jobs from your Jenkins instance
   - Get dependency information for each job
   - Print a summary to console
   - Save complete data to a JSON file

## Additional Features You Can Add:

1. **Pipeline Specific Details**:
   ```python
   # Add to the job details URL
   "pipelines[displayName,weatherScore]"
   ```

2. **Build History**:
   ```python
   "builds[number,result,duration,timestamp]"
   ```

3. **Error Handling**:
   Add more robust error handling for network issues or malformed responses.

4. **CSV Export**:
   Add an option to export to CSV for easier analysis in spreadsheets.

Would you like me to modify the script to include any of these additional features or adapt it for a specific use case?
