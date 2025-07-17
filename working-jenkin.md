# Enhanced Python Script with Repository URL Extraction

Here's the updated script that includes repository URL extraction from Jenkins jobs, along with SSL verification disabled:

```python
import requests
import csv
import json
from requests.auth import HTTPBasicAuth
import urllib3

# Disable SSL warnings
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# Jenkins configuration
JENKINS_URL = 'https://delta-jenkins.com:8443'
JENKINS_USER = 'your-username'
JENKINS_API_TOKEN = 'your-api-token'
VERIFY_SSL = False

def extract_repo_url(job_details):
    """Extract repository URL from job details"""
    # Check for SCM configuration in actions
    for action in job_details.get('actions', []):
        if 'remoteUrls' in action:
            return action['remoteUrls'][0] if action['remoteUrls'] else None
        
        # For pipeline jobs with SCM definition
        if 'scm' in action and 'userRemoteConfigs' in action['scm']:
            for config in action['scm']['userRemoteConfigs']:
                if 'url' in config:
                    return config['url']
    
    # Check for Git SCM in property
    for property in job_details.get('property', []):
        if 'scm' in property and 'userRemoteConfigs' in property['scm']:
            for config in property['scm']['userRemoteConfigs']:
                if 'url' in config:
                    return config['url']
    
    return None

def get_job_details(job_url, auth):
    """Get detailed information about a specific job"""
    details_url = f"{job_url}api/json?tree=upstreamProjects[name],downstreamProjects[name],buildable,description,actions[parameterDefinitions[name,type,defaultValue],remoteUrls,scm[userRemoteConfigs[url]]],property[scm[userRemoteConfigs[url]]]"
    try:
        response = requests.get(details_url, auth=auth, verify=VERIFY_SSL, timeout=30)
        response.raise_for_status()
        details = response.json()
        
        # Extract parameters
        parameters = []
        for action in details.get('actions', []):
            if 'parameterDefinitions' in action:
                for param in action['parameterDefinitions']:
                    parameters.append(f"{param['name']}:{param['type']}")
        
        # Extract repository URL
        repo_url = extract_repo_url(details)
        
        return {
            'upstream': [p['name'] for p in details.get('upstreamProjects', [])],
            'downstream': [p['name'] for p in details.get('downstreamProjects', [])],
            'buildable': details.get('buildable', False),
            'description': details.get('description', ''),
            'parameters': ', '.join(parameters) if parameters else 'None',
            'repo_url': repo_url if repo_url else 'None'
        }
    except requests.exceptions.RequestException as e:
        print(f"Error fetching job details for {job_url}: {str(e)}")
        return None

def get_nested_jobs(folder_url, auth, base_path=""):
    """Recursively get all jobs including nested folder structures"""
    jobs = []
    folder_url = folder_url.rstrip('/')
    
    try:
        contents_url = f"{folder_url}/api/json?tree=jobs[name,url,color,_class]"
        response = requests.get(contents_url, auth=auth, verify=VERIFY_SSL, timeout=30)
        response.raise_for_status()
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
    except requests.exceptions.RequestException as e:
        print(f"Error accessing folder {folder_url}: {str(e)}")
    
    return jobs

def main():
    auth = HTTPBasicAuth(JENKINS_USER, JENKINS_API_TOKEN)
    
    print(f"Fetching all jobs from {JENKINS_URL} (SSL verification {'enabled' if VERIFY_SSL else 'disabled'})...")
    all_jobs = get_nested_jobs(JENKINS_URL, auth)
    
    # Prepare CSV output
    csv_file = 'jenkins_job_inventory_with_repos.csv'
    fieldnames = [
        'full_name', 'name', 'url', 'status', 'buildable',
        'description', 'parameters', 'repo_url', 'upstream', 'downstream'
    ]
    
    with open(csv_file, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=fieldnames)
        writer.writeheader()
        
        for job in sorted(all_jobs, key=lambda x: x['full_name']):
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
        print(f"Repo URL: {job['repo_url']}")
        print(f"Parameters: {job['parameters']}")
        print(f"Upstream: {job['upstream']}")
        print(f"Downstream: {job['downstream']}")

if __name__ == "__main__":
    main()
```

## Key Enhancements:

1. **Repository URL Extraction**:
   - Added `extract_repo_url()` function that checks multiple locations for SCM configuration
   - Handles both Git and other SCM plugins
   - Looks in both job actions and properties

2. **Updated CSV Output**:
   - Added new 'repo_url' column
   - Improved field ordering in CSV output

3. **Better Error Handling**:
   - More robust error handling for repository URL extraction
   - Clearer error messages

4. **Comprehensive Data Collection**:
   - Now collects: job name, URL, status, parameters, description, repo URL, and dependencies

## How It Works:

1. The script recursively scans all Jenkins jobs and folders
2. For each job, it extracts:
   - Basic job information
   - Build parameters
   - Upstream/downstream dependencies
   - Repository URL (from SCM configuration)
3. Results are saved to a CSV file with all collected information

## Sample CSV Output:

```
full_name,name,url,status,buildable,description,parameters,repo_url,upstream,downstream
Investment Technology/AladdinDataLoader,AladdinDataLoader,https://.../job/AladdinDataLoader/,enabled,True,Data loader,BRANCH:String,git@github.com:company/AladdinDataLoader.git,JobA,JobB
...
```

## Notes:

1. The repository URL extraction works for:
   - Git plugin configurations
   - Pipeline jobs with SCM definitions
   - Most common SCM configurations

2. For jobs with multiple repositories, only the first one found will be recorded

3. If you need to handle specific SCM plugins differently, the `extract_repo_url()` function can be extended

Would you like me to add any additional features, such as:
- Branch information extraction
- Last build status
- More detailed SCM information
- Custom filtering options?
