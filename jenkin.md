# Analyzing Jenkins Jobs Using Jenkinsfile in Delta Path

For jobs configured with "Pipeline script from SCM" pointing to `jenkinsfilesDelta/`, we need to modify our approach to properly detect dependencies. Here's how to handle these cases:

## Enhanced Python Script for Jenkinsfile Analysis

```python
import requests
import csv
import re
from requests.auth import HTTPBasicAuth
import urllib3

# Disable SSL warnings
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# Jenkins configuration
JENKINS_URL = 'https://delta-jenkins.eatonvance.com:8443'
JENKINS_USER = 'your-username'
JENKINS_API_TOKEN = 'your-api-token'
VERIFY_SSL = False
JENKINSFILES_BASE_PATH = 'jenkinsfilesDelta/'  # Update with your actual path

def get_jenkinsfile_content(repo_url, file_path, auth):
    """Retrieve Jenkinsfile content from repository"""
    try:
        # This needs to be adapted to your SCM (Git, SVN, etc.)
        # Example for GitHub:
        if 'github.com' in repo_url:
            api_url = repo_url.replace('github.com', 'api.github.com/repos')
            api_url = api_url.replace('tree/', '') + f"/contents/{file_path}"
            response = requests.get(api_url, auth=auth, verify=VERIFY_SSL)
            if response.status_code == 200:
                return response.json().get('content', '')
    except Exception as e:
        print(f"Error fetching Jenkinsfile: {str(e)}")
    return None

def analyze_jenkinsfile(content):
    """Analyze Jenkinsfile content for dependencies"""
    findings = {
        'build_tools': set(),
        'dependencies': set(),
        'upstream_triggers': set(),
        'downstream_triggers': set()
    }

    # Detect build tools
    if re.search(r'(npm|yarn|nuget|mvn|gradle|pip|dotnet)\s', content, re.IGNORECASE):
        findings['build_tools'].update(re.findall(r'(npm|yarn|nuget|mvn|gradle|pip|dotnet)\s', content, re.IGNORECASE))

    # Detect dependency files
    dependency_files = re.findall(r'(package\.json|pom\.xml|build\.gradle|requirements\.txt|\.csproj|\.nuspec)', content)
    if dependency_files:
        findings['dependencies'].update(dependency_files)

    # Detect upstream/downstream triggers
    upstream = re.findall(r'upstream\s*\(.*?projects\s*:\s*\[?(.*?)\]?\)', content, re.IGNORECASE)
    downstream = re.findall(r'downstream\s*\(.*?projects\s*:\s*\[?(.*?)\]?\)', content, re.IGNORECASE)
    
    for match in upstream:
        findings['upstream_triggers'].update(re.findall(r'"([^"]+)"|\'([^\']+)\'|\[([^\]]+)\]', match))
    for match in downstream:
        findings['downstream_triggers'].update(re.findall(r'"([^"]+)"|\'([^\']+)\'|\[([^\]]+)\]', match))

    return findings

def get_pipeline_job_details(job_url, auth):
    """Get details for pipeline jobs using Jenkinsfile"""
    details = {
        'build_tools': 'None',
        'dependencies': 'None',
        'upstream': 'None',
        'downstream': 'None',
        'jenkinsfile_path': 'None',
        'scm_url': 'None'
    }

    try:
        # Get SCM configuration
        config_url = f"{job_url}api/json?tree=actions[*]"
        response = requests.get(config_url, auth=auth, verify=VERIFY_SSL)
        if response.status_code == 200:
            data = response.json()
            
            # Find SCM configuration
            for action in data.get('actions', []):
                if 'scm' in action:
                    scm = action['scm']
                    details['scm_url'] = scm.get('userRemoteConfigs', [{}])[0].get('url', 'None')
                    details['jenkinsfile_path'] = scm.get('scriptPath', 'Jenkinsfile')
                    
                    # Only analyze if in our target path
                    if details['jenkinsfile_path'].startswith(JENKINSFILES_BASE_PATH):
                        content = get_jenkinsfile_content(details['scm_url'], details['jenkinsfile_path'], auth)
                        if content:
                            analysis = analyze_jenkinsfile(content)
                            details.update({
                                'build_tools': ', '.join(analysis['build_tools']) if analysis['build_tools'] else 'None',
                                'dependencies': ', '.join(analysis['dependencies']) if analysis['dependencies'] else 'None',
                                'upstream': ', '.join([x for t in analysis['upstream_triggers'] for x in t if x]) or 'None',
                                'downstream': ', '.join([x for t in analysis['downstream_triggers'] for x in t if x]) or 'None'
                            })
    except Exception as e:
        print(f"Error analyzing pipeline job {job_url}: {str(e)}")

    return details

def main():
    auth = HTTPBasicAuth(JENKINS_USER, JENKINS_API_TOKEN)
    
    print(f"Fetching all jobs from {JENKINS_URL}...")
    
    # Get all jobs (you can reuse your existing function)
    all_jobs = []  # Replace with your job collection logic
    
    # Prepare CSV output
    csv_file = 'jenkins_job_inventory_with_jenkinsfiles.csv'
    fieldnames = [
        'full_name', 'name', 'url', 'type', 'jenkinsfile_path', 'scm_url',
        'build_tools', 'dependencies', 'upstream', 'downstream'
    ]
    
    with open(csv_file, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=fieldnames)
        writer.writeheader()
        
        for job in all_jobs:
            if job.get('_class', '').endswith('WorkflowJob'):
                details = get_pipeline_job_details(job['url'], auth)
                writer.writerow({
                    'full_name': job['full_name'],
                    'name': job['name'],
                    'url': job['url'],
                    'type': 'Pipeline',
                    **details
                })
    
    print(f"\nInventory saved to {csv_file}")

if __name__ == "__main__":
    main()
```

## How to Verify in Jenkins UI

For jobs using `jenkinsfilesDelta/`, follow these steps:

1. **Locate the Jenkinsfile**:
   - Go to the job → Configure
   - In "Pipeline" section, note the "Script Path" (e.g., `jenkinsfilesDelta/AladdinDataLoader.Jenkinsfile`)

2. **Check SCM Configuration**:
   - Under "Pipeline" section, see which repository contains the Jenkinsfile
   - Note the branch specification

3. **View Dependency Triggers**:
   - Look for these patterns in the Jenkinsfile:
     ```groovy
     triggers {
         upstream(upstreamProjects: 'job1,job2', threshold: hudson.model.Result.SUCCESS)
     }
     ```
     ```groovy
     // Manual triggers
     build job: 'downstream-job', parameters: [...]
     ```

4. **Check Build Tools**:
   - Look for build tool commands in the Jenkinsfile:
     ```groovy
     steps {
         sh 'npm install'
         bat 'nuget restore'
     }
     ```

## Common Dependency Patterns in Jenkinsfiles

1. **Explicit Triggers**:
   ```groovy
   triggers {
       upstream(upstreamProjects: 'pre-processor', threshold: hudson.model.Result.SUCCESS)
   }
   ```

2. **Build Steps**:
   ```groovy
   steps {
       build job: 'deploy-prod', parameters: [...]
   }
   ```

3. **Parameterized Triggers**:
   ```groovy
   post {
       success {
           build job: 'notify-success', propagate: false
       }
   }
   ```

## Recommendations

1. **For Accurate Dependency Mapping**:
   - First run the script to identify all jobs using `jenkinsfilesDelta/`
   - Then manually verify a sample of these jobs in UI
   - Adjust the regex patterns in the script based on your actual Jenkinsfile patterns

2. **For Build Tool Detection**:
   - The script looks for both:
     - Direct commands in Jenkinsfile (`npm install`)
     - Dependency files referenced (`package.json`)

3. **For SCM Access**:
   - You may need to add SCM-specific authentication
   - The script currently has basic GitHub support - extend for your SCM

Would you like me to focus on any specific aspect of the Jenkinsfile analysis or dependency detection?
