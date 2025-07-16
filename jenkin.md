# Python Script to Get Nested Jenkins Jobs and Export to CSV

Here's an enhanced Python script that handles nested job structures (like `Investment Technology/AladdinDataLoader`) and exports the inventory to CSV:

```python
import requests
import csv
from requests.auth import HTTPBasicAuth

# Jenkins configuration
JENKINS_URL = 'https://delta-jenkins.com:8443'
JENKINS_USER = 'your-username'  # Or use API token as username
JENKINS_API_TOKEN = 'your-api-token'

def get_job_details(job_url, auth):
    """Get detailed information about a specific job"""
    details_url = f"{job_url}api/json?tree=upstreamProjects[name],downstreamProjects[name],buildable,description"
    response = requests.get(details_url, auth=auth)
    
    if response.status_code == 200:
        details = response.json()
        return {
            'upstream': [p['name'] for p in details.get('upstreamProjects', [])],
            'downstream': [p['name'] for p in details.get('downstreamProjects', [])],
            'buildable': details.get('buildable', False),
            'description': details.get('description', '')
        }
    return None

def get_nested_jobs(folder_url, auth, base_path=""):
    """Recursively get all jobs including nested folder structures"""
    jobs = []
    folder_url = folder_url.rstrip('/')
    
    # Get folder contents
    contents_url = f"{folder_url}/api/json?tree=jobs[name,url,color,_class]"
    response = requests.get(contents_url, auth=auth)
    
    if response.status_code != 200:
        print(f"Failed to fetch folder contents: {response.status_code} - {response.text}")
        return jobs
    
    data = response.json()
    
    for item in data['jobs']:
        item_class = item.get('_class', '')
        full_path = f"{base_path}/{item['name']}" if base_path else item['name']
        
        # Handle nested folders
        if 'folder' in item_class.lower():
            jobs.extend(get_nested_jobs(item['url'], auth, full_path))
        # Handle regular jobs
        else:
            details = get_job_details(item['url'], auth)
            if details:
                jobs.append({
                    'full_name': full_path,
                    'name': item['name'],
                    'url': item['url'],
                    'status': 'disabled' if item['color'] == 'disabled' else 'enabled',
                    **details
                })
    
    return jobs

def main():
    auth = HTTPBasicAuth(JENKINS_USER, JENKINS_API_TOKEN)
    
    print(f"Fetching all jobs from {JENKINS_URL}...")
    all_jobs = get_nested_jobs(JENKINS_URL, auth)
    
    # Prepare CSV output
    csv_file = 'jenkins_job_inventory.csv'
    fieldnames = [
        'full_name', 'name', 'url', 'status', 'buildable',
        'description', 'upstream', 'downstream'
    ]
    
    with open(csv_file, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=fieldnames)
        writer.writeheader()
        
        for job in all_jobs:
            # Convert lists to strings for CSV
            job['upstream'] = ', '.join(job['upstream'])
            job['downstream'] = ', '.join(job['downstream'])
            writer.writerow(job)
    
    print(f"\nSuccessfully exported {len(all_jobs)} jobs to {csv_file}")
    
    # Print sample output
    print("\nSample of exported data:")
    sample = all_jobs[:3] if len(all_jobs) > 3 else all_jobs
    for job in sample:
        print(f"\nJob: {job['full_name']}")
        print(f"URL: {job['url']}")
        print(f"Status: {job['status']}")
        print(f"Upstream: {job['upstream']}")
        print(f"Downstream: {job['downstream']}")

if __name__ == "__main__":
    main()
```

## Key Features of This Script:

1. **Handles Nested Job Structures**: Properly processes folders like `Investment Technology/AladdinDataLoader`

2. **CSV Export**: Creates a clean CSV file with these columns:
   - full_name (with path)
   - name (job name only)
   - url
   - status (enabled/disabled)
   - buildable
   - description
   - upstream dependencies
   - downstream dependencies

3. **Comprehensive Data Collection**: Gets all relevant job information including:
   - Job hierarchy and organization
   - Upstream/downstream dependencies
   - Build status (enabled/disabled)
   - Job descriptions

4. **Error Handling**: Basic error handling for API requests

## How to Use:

1. Replace the configuration at the top with your credentials:
   ```python
   JENKINS_URL = 'https://delta-jenkins.com:8443'
   JENKINS_USER = 'your-username'
   JENKINS_API_TOKEN = 'your-api-token'
   ```

2. Run the script:
   ```
   python jenkins_inventory.py
   ```

3. The script will:
   - Recursively scan all jobs and folders
   - Create a CSV file named `jenkins_job_inventory.csv`
   - Print a summary with sample data

## Sample CSV Output:

```
full_name,name,url,status,buildable,description,upstream,downstream
Investment Technology/AladdinDataLoader,AladdinDataLoader,https://delta-jenkins.com:8443/job/Investment%20Technology/job/AladdinDataLoader/,enabled,True,Loader for Aladdin data,JobA, JobB,JobC, JobD
...
```

Would you like me to add any additional features like:
- Parallel requests for faster execution
- More detailed pipeline information
- Custom CSV formatting options
- Specific filtering capabilities?
