# APIs and Libraries in Python: A Complete Guide

## Understanding APIs (Application Programming Interfaces)

### What is an API?

An API is a set of rules and protocols that allows different software applications to communicate with each other. Think of it as a waiter in a restaurant: you (the client) give your order to the waiter (the API), who takes it to the kitchen (the server), and brings back your food (the data).

**Real-world example:** When you use a weather app on your phone, it doesn't store all weather data locally. Instead, it sends a request to a weather service's API, which returns current weather information.

### Types of APIs

1. **REST APIs** (most common)
   - Use HTTP methods (GET, POST, PUT, DELETE)
   - Work with URLs and return data (usually JSON)
   - Stateless communication

2. **GraphQL APIs**
   - Request exactly the data you need
   - Single endpoint for all operations

3. **SOAP APIs**
   - XML-based, more rigid structure
   - Common in enterprise systems

---

## Making HTTP Requests in Python

### The `requests` Library

The most popular library for making HTTP requests in Python.

#### Installation

```bash
pip install requests
```

#### Basic GET Request

```python
import requests

# Simple GET request
response = requests.get('https://api.github.com/users/octocat')

# Check if request was successful
if response.status_code == 200:
    print("Success!")
    print(response.json())  # Parse JSON response
else:
    print(f"Error: {response.status_code}")
```

#### Understanding Response Objects

```python
import requests

response = requests.get('https://api.github.com/users/octocat')

# Status code (200 = success, 404 = not found, 500 = server error)
print(response.status_code)

# Response headers (metadata about the response)
print(response.headers)

# Raw content (bytes)
print(response.content)

# Text content (string)
print(response.text)

# JSON parsed content (Python dictionary)
print(response.json())
```

### HTTP Methods Explained

```python
import requests

base_url = "https://api.example.com"

# GET - Retrieve data
response = requests.get(f"{base_url}/users/123")

# POST - Create new data
new_user = {
    "name": "Daniel Araromi",
    "email": "daniel@onswift.com"
}
response = requests.post(f"{base_url}/users", json=new_user)

# PUT - Update existing data completely
updated_user = {
    "name": "Daniel Araromi",
    "email": "daniel@oversubscribed.com",
    "company": "Onswift"
}
response = requests.put(f"{base_url}/users/123", json=updated_user)

# PATCH - Partially update data
partial_update = {"company": "Oversubscribed"}
response = requests.patch(f"{base_url}/users/123", json=partial_update)

# DELETE - Remove data
response = requests.delete(f"{base_url}/users/123")
```

### Adding Headers and Authentication

```python
import requests

# API Key authentication
headers = {
    "Authorization": "Bearer YOUR_API_KEY_HERE",
    "Content-Type": "application/json"
}

response = requests.get(
    "https://api.example.com/data",
    headers=headers
)

# Basic authentication
response = requests.get(
    "https://api.example.com/data",
    auth=('username', 'password')
)

# Query parameters
params = {
    "page": 1,
    "limit": 50,
    "sort": "created_at"
}

response = requests.get(
    "https://api.example.com/items",
    params=params,
    headers=headers
)
```

### Handling Timeouts and Errors

```python
import requests
from requests.exceptions import Timeout, ConnectionError, HTTPError

try:
    # Set timeout (in seconds)
    response = requests.get(
        "https://api.example.com/data",
        timeout=5
    )
    
    # Raise exception for bad status codes (4xx, 5xx)
    response.raise_for_status()
    
    data = response.json()
    print(data)
    
except Timeout:
    print("Request timed out. Server took too long to respond.")
    
except ConnectionError:
    print("Failed to connect to the server. Check your internet connection.")
    
except HTTPError as e:
    print(f"HTTP error occurred: {e}")
    
except Exception as e:
    print(f"An unexpected error occurred: {e}")
```

---

## Working with JSON Data

### What is JSON?

JSON (JavaScript Object Notation) is a lightweight data format that's easy for humans to read and machines to parse. It's the standard format for API responses.

### JSON Structure

```json
{
  "user": {
    "id": 123,
    "name": "Daniel Araromi",
    "email": "daniel@onswift.com",
    "companies": ["Onswift", "Oversubscribed"],
    "active": true,
    "metadata": null
  }
}
```

### Parsing JSON in Python

```python
import requests
import json

# From API response
response = requests.get('https://api.github.com/users/octocat')
data = response.json()  # Converts JSON to Python dict

# Accessing data
print(data['name'])
print(data['company'])

# From JSON string
json_string = '{"name": "Daniel", "age": 20}'
data = json.loads(json_string)

# To JSON string
python_dict = {
    "name": "Daniel Araromi",
    "businesses": ["Onswift", "Oversubscribed"]
}
json_string = json.dumps(python_dict, indent=2)
print(json_string)

# Reading from JSON file
with open('data.json', 'r') as file:
    data = json.load(file)

# Writing to JSON file
with open('output.json', 'w') as file:
    json.dump(python_dict, file, indent=2)
```

### Complex JSON Navigation

```python
import requests

response = requests.get('https://api.github.com/users/octocat/repos')
repos = response.json()

# repos is a list of dictionaries
for repo in repos[:5]:  # First 5 repos
    print(f"Name: {repo['name']}")
    print(f"Stars: {repo['stargazers_count']}")
    print(f"Language: {repo.get('language', 'Not specified')}")
    print("---")

# Using .get() to avoid KeyError
owner_name = repo.get('owner', {}).get('login', 'Unknown')
```

---

## Working with External Packages

### pip - Python Package Installer

```bash
# Install a package
pip install requests

# Install specific version
pip install requests==2.28.0

# Install from requirements file
pip install -r requirements.txt

# Upgrade a package
pip install --upgrade requests

# Uninstall a package
pip uninstall requests

# List installed packages
pip list

# Show package information
pip show requests

# Search for packages
pip search "web scraping"
```

### Creating requirements.txt

```bash
# Generate requirements.txt from current environment
pip freeze > requirements.txt
```

Example `requirements.txt`:
```
requests==2.31.0
python-dotenv==1.0.0
pandas==2.0.3
```

---

## Virtual Environments

### Why Use Virtual Environments?

Virtual environments create isolated Python environments for different projects, preventing package conflicts.

**Without virtual environment:**
- Project A needs requests v2.20
- Project B needs requests v2.31
- Conflict! Only one version can be installed globally.

**With virtual environment:**
- Each project has its own isolated package set
- No conflicts

### Creating and Using Virtual Environments

#### Using `venv` (built-in)

```bash
# Create virtual environment
python -m venv venv

# Activate (Windows)
venv\Scripts\activate

# Activate (Mac/Linux)
source venv/bin/activate

# Your prompt changes to show (venv) when activated

# Install packages (now isolated to this environment)
pip install requests pandas

# Deactivate
deactivate
```

#### Using `virtualenv` (external tool)

```bash
# Install virtualenv
pip install virtualenv

# Create environment
virtualenv myenv

# Activate
source myenv/bin/activate  # Mac/Linux
myenv\Scripts\activate     # Windows
```

### Project Structure with Virtual Environment

```
my-project/
│
├── venv/                 # Virtual environment (don't commit to Git)
├── src/
│   ├── main.py
│   └── api_client.py
├── requirements.txt      # Package dependencies
├── .env                  # Environment variables (don't commit)
├── .gitignore           # Git ignore file
└── README.md
```

Example `.gitignore`:
```
venv/
__pycache__/
*.pyc
.env
```

---

## Dependency Management Best Practices

### 1. Environment Variables for Secrets

Never hardcode API keys or passwords.

```bash
# Install python-dotenv
pip install python-dotenv
```

Create `.env` file:
```
API_KEY=your_secret_key_here
DATABASE_URL=postgresql://user:pass@localhost/db
```

Use in Python:
```python
from dotenv import load_dotenv
import os

# Load environment variables
load_dotenv()

# Access variables
api_key = os.getenv('API_KEY')
db_url = os.getenv('DATABASE_URL')

# Use in requests
headers = {
    "Authorization": f"Bearer {api_key}"
}
```

### 2. Project Workflow

```bash
# 1. Create project directory
mkdir my-api-project
cd my-api-project

# 2. Create virtual environment
python -m venv venv

# 3. Activate
source venv/bin/activate  # Mac/Linux

# 4. Install packages
pip install requests python-dotenv

# 5. Create requirements.txt
pip freeze > requirements.txt

# 6. Create .env for secrets
echo "API_KEY=your_key" > .env

# 7. Create .gitignore
echo -e "venv/\n.env\n__pycache__/" > .gitignore
```

---

## Real-World API Example: Building a GitHub Stats Fetcher

```python
import requests
import os
from dotenv import load_dotenv
from datetime import datetime

# Load environment variables
load_dotenv()

class GitHubClient:
    def __init__(self):
        self.base_url = "https://api.github.com"
        self.token = os.getenv('GITHUB_TOKEN')
        self.headers = {
            "Authorization": f"token {self.token}",
            "Accept": "application/vnd.github.v3+json"
        }
    
    def get_user(self, username):
        """Fetch user information"""
        try:
            response = requests.get(
                f"{self.base_url}/users/{username}",
                headers=self.headers,
                timeout=10
            )
            response.raise_for_status()
            return response.json()
        
        except requests.exceptions.HTTPError as e:
            if response.status_code == 404:
                return {"error": "User not found"}
            return {"error": str(e)}
        
        except Exception as e:
            return {"error": f"Unexpected error: {str(e)}"}
    
    def get_repos(self, username, sort='stars'):
        """Fetch user repositories"""
        try:
            response = requests.get(
                f"{self.base_url}/users/{username}/repos",
                headers=self.headers,
                params={"sort": sort, "per_page": 100},
                timeout=10
            )
            response.raise_for_status()
            return response.json()
        
        except Exception as e:
            return {"error": str(e)}
    
    def get_stats(self, username):
        """Get comprehensive user stats"""
        user = self.get_user(username)
        if "error" in user:
            return user
        
        repos = self.get_repos(username)
        
        total_stars = sum(repo['stargazers_count'] for repo in repos)
        total_forks = sum(repo['forks_count'] for repo in repos)
        languages = set(repo['language'] for repo in repos if repo['language'])
        
        return {
            "username": user['login'],
            "name": user['name'],
            "followers": user['followers'],
            "following": user['following'],
            "public_repos": user['public_repos'],
            "total_stars": total_stars,
            "total_forks": total_forks,
            "languages": list(languages),
            "top_repos": sorted(
                repos,
                key=lambda x: x['stargazers_count'],
                reverse=True
            )[:5]
        }

# Usage
if __name__ == "__main__":
    client = GitHubClient()
    stats = client.get_stats("torvalds")
    
    print(f"User: {stats['name']}")
    print(f"Followers: {stats['followers']}")
    print(f"Total Stars: {stats['total_stars']}")
    print(f"\nTop Repositories:")
    for repo in stats['top_repos']:
        print(f"  - {repo['name']}: {repo['stargazers_count']} stars")
```

---

## Popular Python Libraries for Different Use Cases

### Web Requests & APIs
- **requests** - HTTP requests
- **httpx** - Async HTTP client
- **aiohttp** - Async HTTP client/server

### Data Processing
- **pandas** - Data manipulation and analysis
- **numpy** - Numerical computing
- **openpyxl** - Excel file handling

### Web Scraping
- **beautifulsoup4** - HTML parsing
- **scrapy** - Web scraping framework
- **selenium** - Browser automation

### Authentication & Security
- **python-dotenv** - Environment variables
- **pyjwt** - JSON Web Tokens
- **cryptography** - Cryptographic operations

### Database
- **sqlalchemy** - SQL toolkit and ORM
- **pymongo** - MongoDB driver
- **psycopg2** - PostgreSQL adapter

---

## Common API Patterns and Best Practices

### 1. Rate Limiting Handling

```python
import requests
import time

def make_request_with_retry(url, max_retries=3):
    for attempt in range(max_retries):
        response = requests.get(url)
        
        if response.status_code == 429:  # Rate limited
            retry_after = int(response.headers.get('Retry-After', 60))
            print(f"Rate limited. Waiting {retry_after} seconds...")
            time.sleep(retry_after)
            continue
        
        return response
    
    raise Exception("Max retries exceeded")
```

### 2. Pagination

```python
def fetch_all_pages(base_url, headers):
    all_data = []
    page = 1
    
    while True:
        response = requests.get(
            base_url,
            headers=headers,
            params={"page": page, "per_page": 100}
        )
        
        data = response.json()
        
        if not data:  # No more data
            break
        
        all_data.extend(data)
        page += 1
    
    return all_data
```

### 3. Caching Responses

```python
import requests
from functools import lru_cache

@lru_cache(maxsize=128)
def fetch_user(username):
    response = requests.get(f"https://api.github.com/users/{username}")
    return response.json()

# First call hits API
user1 = fetch_user("octocat")

# Second call uses cache
user2 = fetch_user("octocat")  # Instant, no API call
```

---

## Advanced Topics

### Async Requests with `httpx`

For making multiple API calls simultaneously (much faster for bulk operations):

```python
import httpx
import asyncio

async def fetch_user(client, username):
    response = await client.get(f"https://api.github.com/users/{username}")
    return response.json()

async def fetch_multiple_users(usernames):
    async with httpx.AsyncClient() as client:
        tasks = [fetch_user(client, username) for username in usernames]
        results = await asyncio.gather(*tasks)
        return results

# Usage
usernames = ["octocat", "torvalds", "gvanrossum"]
results = asyncio.run(fetch_multiple_users(usernames))

for user in results:
    print(f"{user['name']}: {user['followers']} followers")
```

### Session Management for Multiple Requests

```python
import requests

# Using sessions for better performance
session = requests.Session()

# Set default headers for all requests in this session
session.headers.update({
    "Authorization": "Bearer YOUR_TOKEN",
    "User-Agent": "MyApp/1.0"
})

# All requests now use these headers
response1 = session.get("https://api.example.com/endpoint1")
response2 = session.get("https://api.example.com/endpoint2")
response3 = session.get("https://api.example.com/endpoint3")

# Close session when done
session.close()
```

### Building a Robust API Wrapper Class

```python
import requests
from typing import Dict, Any, Optional
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class APIClient:
    def __init__(self, base_url: str, api_key: Optional[str] = None):
        self.base_url = base_url.rstrip('/')
        self.session = requests.Session()
        
        if api_key:
            self.session.headers.update({
                "Authorization": f"Bearer {api_key}"
            })
    
    def _request(
        self, 
        method: str, 
        endpoint: str, 
        **kwargs
    ) -> Dict[Any, Any]:
        """Generic request method with error handling"""
        url = f"{self.base_url}/{endpoint.lstrip('/')}"
        
        try:
            response = self.session.request(
                method, 
                url, 
                timeout=kwargs.pop('timeout', 10),
                **kwargs
            )
            response.raise_for_status()
            
            logger.info(f"{method} {url} - Status: {response.status_code}")
            return response.json()
            
        except requests.exceptions.HTTPError as e:
            logger.error(f"HTTP Error: {e}")
            return {"error": str(e), "status_code": response.status_code}
            
        except requests.exceptions.Timeout:
            logger.error(f"Timeout error for {url}")
            return {"error": "Request timeout"}
            
        except Exception as e:
            logger.error(f"Unexpected error: {e}")
            return {"error": str(e)}
    
    def get(self, endpoint: str, params: Optional[Dict] = None) -> Dict:
        return self._request("GET", endpoint, params=params)
    
    def post(self, endpoint: str, data: Optional[Dict] = None) -> Dict:
        return self._request("POST", endpoint, json=data)
    
    def put(self, endpoint: str, data: Optional[Dict] = None) -> Dict:
        return self._request("PUT", endpoint, json=data)
    
    def delete(self, endpoint: str) -> Dict:
        return self._request("DELETE", endpoint)
    
    def __enter__(self):
        return self
    
    def __exit__(self, *args):
        self.session.close()

# Usage
with APIClient("https://api.example.com", api_key="your_key") as client:
    user = client.get("/users/123")
    new_item = client.post("/items", data={"name": "New Item"})
```

---

## Troubleshooting Common Issues

### SSL Certificate Errors

```python
import requests

# Disable SSL verification (not recommended for production)
response = requests.get("https://example.com", verify=False)

# Specify custom CA bundle
response = requests.get("https://example.com", verify="/path/to/certfile")
```

### Proxy Configuration

```python
proxies = {
    "http": "http://proxy.example.com:8080",
    "https": "https://proxy.example.com:8080",
}

response = requests.get("https://api.example.com", proxies=proxies)
```

### Large File Downloads

```python
import requests

url = "https://example.com/large-file.zip"

# Stream download to avoid loading entire file in memory
with requests.get(url, stream=True) as r:
    r.raise_for_status()
    with open("large-file.zip", "wb") as f:
        for chunk in r.iter_content(chunk_size=8192):
            f.write(chunk)
```

### File Uploads

```python
# Upload a file
files = {"file": open("document.pdf", "rb")}
response = requests.post("https://api.example.com/upload", files=files)

# Multiple files
files = {
    "file1": open("doc1.pdf", "rb"),
    "file2": open("doc2.pdf", "rb")
}
response = requests.post("https://api.example.com/upload", files=files)
```

---

## Testing API Integrations

### Using `responses` Library for Testing

```bash
pip install responses
```

```python
import responses
import requests

@responses.activate
def test_api_call():
    # Mock the API response
    responses.add(
        responses.GET,
        "https://api.example.com/users/123",
        json={"id": 123, "name": "Daniel Araromi"},
        status=200
    )
    
    # Make the request
    response = requests.get("https://api.example.com/users/123")
    
    # Test the response
    assert response.status_code == 200
    assert response.json()["name"] == "Daniel Araromi"

test_api_call()
print("Test passed!")
```

---

## Quick Reference: Common Status Codes

| Code | Meaning | Description |
|------|---------|-------------|
| 200 | OK | Request successful |
| 201 | Created | Resource created successfully |
| 204 | No Content | Success, but no data to return |
| 400 | Bad Request | Invalid request format |
| 401 | Unauthorized | Authentication required |
| 403 | Forbidden | No permission to access |
| 404 | Not Found | Resource doesn't exist |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server-side error |
| 503 | Service Unavailable | Server temporarily down |

---

## Next Steps

1. **Practice with public APIs:**
   - GitHub API: https://api.github.com
   - JSONPlaceholder (fake API for testing): https://jsonplaceholder.typicode.com
   - OpenWeatherMap: https://openweathermap.org/api
   - NewsAPI: https://newsapi.org

2. **Build projects:**
   - Weather dashboard
   - GitHub profile analyzer
   - Social media post scheduler
   - Data aggregator from multiple APIs

3. **Learn advanced topics:**
   - OAuth authentication
   - WebSockets for real-time data
   - GraphQL queries
   - API rate limiting strategies

---

**Remember:** Always read the API documentation, respect rate limits, secure your API keys, and handle errors gracefully. Building with APIs is one of the most powerful skills in modern software development.