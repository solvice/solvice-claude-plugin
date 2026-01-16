# Solvice API Integration Guide

## Skill Definition

- name: integration
- description: General integration patterns for Solvice API - authentication, workflows, error handling, and code examples
- triggers:
  - solvice integration
  - solvice api
  - solvice authentication
  - solvice setup
  - integrate solvice

---

## Authentication Setup

### Getting Your API Key

1. Sign up at [solvice.io](https://solvice.io)
2. Navigate to your dashboard → API Keys
3. Create a new API key
4. Store it securely (never commit to version control)

### Environment Variables

Store your API key as an environment variable:

```bash
# .env file (add to .gitignore!)
SOLVICE_API_KEY=your-api-key-here

# Or export directly
export SOLVICE_API_KEY="your-api-key-here"
```

### Using the API Key

Include the key in the `Authorization` header:

```bash
curl -H "Authorization: Bearer $SOLVICE_API_KEY" \
  https://api.solvice.io/v2/vrp/solve
```

---

## Async vs. Sync Workflows

Solvice offers two solving modes:

### Async Workflow (Recommended)

Best for:
- Large problems (many jobs/resources)
- Production workloads
- When you can wait for results

**Flow:**
1. Submit problem → Get job ID
2. Poll status endpoint until complete
3. Fetch solution

```
POST /solve → {"jobId": "abc123"}
     ↓
GET /jobs/abc123/status → {"status": "RUNNING"}
     ↓ (poll)
GET /jobs/abc123/status → {"status": "SOLVED"}
     ↓
GET /jobs/abc123/solution → {solution data}
```

### Sync Workflow

Best for:
- Small problems
- Real-time applications
- When you need immediate results

**Flow:**
1. Submit problem → Wait → Get solution directly

```
POST /sync/solve → {solution data}
```

**Tradeoffs:**
- Simpler code (no polling)
- Limited to smaller problems
- HTTP timeout constraints (~30 seconds)

---

## Official SDKs (Recommended)

Solvice provides official SDKs with full type safety and a cleaner developer experience.

### Python SDK

```bash
pip install --pre solvice-vrp-solver
```

```python
from solvice_vrp_solver import SolviceVrpSolver

client = SolviceVrpSolver(api_key="your_api_key")

# Synchronous solve (simplest approach)
solution = client.vrp.sync_solve({
    "jobs": [
        {
            "name": "delivery-1",
            "location": {"latitude": 51.0500, "longitude": 3.7300},
            "duration": 900
        },
        {
            "name": "delivery-2",
            "location": {"latitude": 51.0543, "longitude": 3.7174},
            "duration": 600
        }
    ],
    "resources": [{
        "name": "vehicle-1",
        "shifts": [{
            "from": "2024-01-15T08:00:00Z",
            "to": "2024-01-15T17:00:00Z",
            "start": {"latitude": 51.0543, "longitude": 3.7174}
        }]
    }]
})

print(solution.routes)
```

### TypeScript SDK

```bash
npm install solvice-vrp-solver
```

```typescript
import SolviceVrpSolver from 'solvice-vrp-solver';

const client = new SolviceVrpSolver({
  apiKey: process.env.SOLVICE_API_KEY,
});

const solution = await client.vrp.syncSolve({
  jobs: [
    {
      name: 'delivery-1',
      location: { latitude: 51.0500, longitude: 3.7300 },
      duration: 900
    }
  ],
  resources: [{
    name: 'vehicle-1',
    shifts: [{
      from: '2024-01-15T08:00:00Z',
      to: '2024-01-15T17:00:00Z',
      start: { latitude: 51.0543, longitude: 3.7174 }
    }]
  }],
});

console.log(solution.routes);
```

The TypeScript SDK works across Node.js, Deno, Bun, and browsers.

---

## Raw HTTP API (Alternative)

If you prefer not to use the SDK, you can call the REST API directly. See [docs.solvice.io](https://docs.solvice.io) for the full API reference.

### Python (using requests)

```python
import os
import time
import requests

API_KEY = os.environ["SOLVICE_API_KEY"]
BASE_URL = "https://api.solvice.io/v2"
HEADERS = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json"
}

def solve_vrp(problem: dict) -> dict:
    """Solve a VRP problem asynchronously."""

    # 1. Submit the problem
    response = requests.post(
        f"{BASE_URL}/vrp/solve",
        headers=HEADERS,
        json=problem
    )
    response.raise_for_status()
    job_id = response.json()["jobId"]
    print(f"Job submitted: {job_id}")

    # 2. Poll for completion
    while True:
        status_response = requests.get(
            f"{BASE_URL}/vrp/jobs/{job_id}/status",
            headers=HEADERS
        )
        status_response.raise_for_status()
        status = status_response.json()["status"]

        if status == "SOLVED":
            break
        elif status == "ERROR":
            raise Exception(f"Solve failed: {status_response.json()}")

        print(f"Status: {status}, waiting...")
        time.sleep(2)

    # 3. Fetch the solution
    solution_response = requests.get(
        f"{BASE_URL}/vrp/jobs/{job_id}/solution",
        headers=HEADERS
    )
    solution_response.raise_for_status()
    return solution_response.json()


# Example usage
problem = {
    "jobs": [
        {"name": "job-1", "location": {"lat": 51.05, "lng": 3.72}, "duration": 300},
        {"name": "job-2", "location": {"lat": 51.06, "lng": 3.73}, "duration": 300}
    ],
    "resources": [
        {
            "name": "vehicle-1",
            "shifts": [{
                "start": "2024-01-15T08:00:00Z",
                "end": "2024-01-15T17:00:00Z",
                "startLocation": {"lat": 51.05, "lng": 3.72}
            }]
        }
    ]
}

solution = solve_vrp(problem)
print(f"Solution score: {solution['score']}")
for resource in solution["resources"]:
    print(f"\nRoute for {resource['name']}:")
    for stop in resource["route"]:
        print(f"  - {stop.get('name', stop['type'])}")
```

### Python (async with aiohttp)

```python
import os
import asyncio
import aiohttp

API_KEY = os.environ["SOLVICE_API_KEY"]
BASE_URL = "https://api.solvice.io/v2"

async def solve_vrp_async(problem: dict) -> dict:
    """Solve a VRP problem asynchronously using aiohttp."""
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json"
    }

    async with aiohttp.ClientSession() as session:
        # Submit
        async with session.post(
            f"{BASE_URL}/vrp/solve",
            headers=headers,
            json=problem
        ) as response:
            response.raise_for_status()
            job_id = (await response.json())["jobId"]

        # Poll
        while True:
            async with session.get(
                f"{BASE_URL}/vrp/jobs/{job_id}/status",
                headers=headers
            ) as response:
                status = (await response.json())["status"]
                if status == "SOLVED":
                    break
                elif status == "ERROR":
                    raise Exception("Solve failed")
            await asyncio.sleep(2)

        # Fetch solution
        async with session.get(
            f"{BASE_URL}/vrp/jobs/{job_id}/solution",
            headers=headers
        ) as response:
            return await response.json()
```

### TypeScript (using fetch)

```typescript
const API_KEY = process.env.SOLVICE_API_KEY!;
const BASE_URL = "https://api.solvice.io/v2";

interface VRPProblem {
  jobs: Array<{
    name: string;
    location: { lat: number; lng: number };
    duration: number;
  }>;
  resources: Array<{
    name: string;
    shifts: Array<{
      start: string;
      end: string;
      startLocation: { lat: number; lng: number };
    }>;
  }>;
}

interface VRPSolution {
  status: string;
  score: { hardScore: number; mediumScore: number; softScore: number };
  resources: Array<{
    name: string;
    route: Array<{ type: string; name?: string }>;
  }>;
  unassigned: Array<{ name: string; reason: string }>;
}

async function solveVRP(problem: VRPProblem): Promise<VRPSolution> {
  const headers = {
    Authorization: `Bearer ${API_KEY}`,
    "Content-Type": "application/json",
  };

  // 1. Submit the problem
  const submitResponse = await fetch(`${BASE_URL}/vrp/solve`, {
    method: "POST",
    headers,
    body: JSON.stringify(problem),
  });

  if (!submitResponse.ok) {
    throw new Error(`Submit failed: ${submitResponse.statusText}`);
  }

  const { jobId } = await submitResponse.json();
  console.log(`Job submitted: ${jobId}`);

  // 2. Poll for completion
  while (true) {
    const statusResponse = await fetch(
      `${BASE_URL}/vrp/jobs/${jobId}/status`,
      { headers }
    );
    const { status } = await statusResponse.json();

    if (status === "SOLVED") break;
    if (status === "ERROR") throw new Error("Solve failed");

    console.log(`Status: ${status}, waiting...`);
    await new Promise((resolve) => setTimeout(resolve, 2000));
  }

  // 3. Fetch the solution
  const solutionResponse = await fetch(
    `${BASE_URL}/vrp/jobs/${jobId}/solution`,
    { headers }
  );

  return solutionResponse.json();
}

// Example usage
async function main() {
  const problem: VRPProblem = {
    jobs: [
      { name: "job-1", location: { lat: 51.05, lng: 3.72 }, duration: 300 },
      { name: "job-2", location: { lat: 51.06, lng: 3.73 }, duration: 300 },
    ],
    resources: [
      {
        name: "vehicle-1",
        shifts: [
          {
            start: "2024-01-15T08:00:00Z",
            end: "2024-01-15T17:00:00Z",
            startLocation: { lat: 51.05, lng: 3.72 },
          },
        ],
      },
    ],
  };

  const solution = await solveVRP(problem);
  console.log("Solution:", JSON.stringify(solution, null, 2));
}

main().catch(console.error);
```

### curl (Bash)

```bash
#!/bin/bash

# Configuration
API_KEY="${SOLVICE_API_KEY}"
BASE_URL="https://api.solvice.io/v2"

# Submit the problem
JOB_ID=$(curl -s -X POST "${BASE_URL}/vrp/solve" \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "jobs": [
      {"name": "job-1", "location": {"lat": 51.05, "lng": 3.72}, "duration": 300}
    ],
    "resources": [
      {
        "name": "vehicle-1",
        "shifts": [{
          "start": "2024-01-15T08:00:00Z",
          "end": "2024-01-15T17:00:00Z",
          "startLocation": {"lat": 51.05, "lng": 3.72}
        }]
      }
    ]
  }' | jq -r '.jobId')

echo "Job submitted: ${JOB_ID}"

# Poll for completion
while true; do
  STATUS=$(curl -s "${BASE_URL}/vrp/jobs/${JOB_ID}/status" \
    -H "Authorization: Bearer ${API_KEY}" | jq -r '.status')

  echo "Status: ${STATUS}"

  if [ "$STATUS" == "SOLVED" ]; then
    break
  elif [ "$STATUS" == "ERROR" ]; then
    echo "Solve failed!"
    exit 1
  fi

  sleep 2
done

# Fetch solution
curl -s "${BASE_URL}/vrp/jobs/${JOB_ID}/solution" \
  -H "Authorization: Bearer ${API_KEY}" | jq .
```

---

## Reading Solutions

### Solution Structure

```json
{
  "status": "SOLVED",
  "score": {
    "hardScore": 0,
    "mediumScore": 0,
    "softScore": -12500
  },
  "resources": [
    {
      "name": "vehicle-1",
      "route": [
        {"type": "start", "arrival": "08:00:00"},
        {"type": "job", "name": "job-1", "arrival": "08:15:00", "departure": "08:20:00"},
        {"type": "job", "name": "job-2", "arrival": "08:35:00", "departure": "08:40:00"},
        {"type": "end", "arrival": "09:00:00"}
      ],
      "stats": {
        "travelTime": 1800,
        "serviceTime": 600,
        "totalTime": 3600
      }
    }
  ],
  "unassigned": []
}
```

### Understanding Scores

| Score | Meaning | Example |
|-------|---------|---------|
| `hardScore = 0` | All hard constraints satisfied | Good! |
| `hardScore < 0` | Hard constraints violated | Problem - some jobs can't be done |
| `mediumScore` | Medium constraint violations | Trade-offs were made |
| `softScore` | Optimization cost (lower = worse) | -12500 means 12500 "cost units" |

### Handling Unassigned Jobs

When jobs can't be assigned:

```json
{
  "unassigned": [
    {
      "name": "job-42",
      "reason": "No eligible resource: missing required tag 'hazmat'"
    }
  ]
}
```

**Common reasons:**
- No resource with required skills
- Time window conflicts
- Capacity exceeded
- No available resource during job's time window

---

## Explainable AI

Use the `/explanation` endpoint to understand **why** the solver made specific decisions:

```bash
curl "https://api.solvice.io/v2/vrp/jobs/{job-id}/explanation" \
  -H "Authorization: Bearer $SOLVICE_API_KEY"
```

### Explanation Response

```json
{
  "assignments": [
    {
      "job": "delivery-1",
      "resource": "van-1",
      "reasons": [
        "Closest available resource (2.3 km)",
        "Has required tags: [fragile]",
        "Within capacity limits (45/100 packages)"
      ]
    }
  ],
  "unassigned": [
    {
      "job": "delivery-99",
      "reasons": [
        "Required tag 'hazmat' not found on any resource",
        "Considered resources: van-1 (missing tag), van-2 (missing tag)"
      ]
    }
  ]
}
```

Use explanations to:
- Debug why jobs are unassigned
- Understand routing decisions
- Validate constraints are working correctly
- Build trust in optimization results

---

## Error Handling

### HTTP Status Codes

| Code | Meaning | Action |
|------|---------|--------|
| `200` | Success | Process response |
| `400` | Bad Request | Check request format |
| `401` | Unauthorized | Check API key |
| `403` | Forbidden | Check permissions/quota |
| `404` | Not Found | Check job ID |
| `429` | Rate Limited | Back off and retry |
| `500` | Server Error | Retry with exponential backoff |

### Common Errors

**Invalid JSON:**
```json
{
  "error": "Invalid JSON",
  "message": "Unexpected token at position 42"
}
```

**Validation Error:**
```json
{
  "error": "Validation Error",
  "details": [
    {"field": "jobs[0].duration", "message": "Duration must be positive"}
  ]
}
```

**Quota Exceeded:**
```json
{
  "error": "Quota Exceeded",
  "message": "Monthly solve limit reached"
}
```

### Retry Strategy

```python
import time
from requests.exceptions import RequestException

def solve_with_retry(problem, max_retries=3):
    for attempt in range(max_retries):
        try:
            return solve_vrp(problem)
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt  # Exponential backoff
            print(f"Retry {attempt + 1} in {wait_time}s: {e}")
            time.sleep(wait_time)
```

---

## Best Practices

### 1. Validate Input Locally

Check for obvious issues before sending:

```python
def validate_problem(problem):
    errors = []

    for job in problem.get("jobs", []):
        if job.get("duration", 0) <= 0:
            errors.append(f"Job {job['name']}: duration must be positive")
        if not job.get("location"):
            errors.append(f"Job {job['name']}: missing location")

    for resource in problem.get("resources", []):
        if not resource.get("shifts"):
            errors.append(f"Resource {resource['name']}: no shifts defined")

    if errors:
        raise ValueError("\n".join(errors))
```

### 2. Use Meaningful Names

```json
// Good
{"name": "delivery-customer-acme-001"}

// Bad
{"name": "j1"}
```

### 3. Handle Partial Solutions

Sometimes not all jobs can be assigned. Design your application to handle this:

```python
solution = solve_vrp(problem)

if solution["unassigned"]:
    print(f"Warning: {len(solution['unassigned'])} jobs unassigned")
    for job in solution["unassigned"]:
        print(f"  - {job['name']}: {job['reason']}")
        # Maybe alert operations team
```

### 4. Log Job IDs

Always log job IDs for debugging:

```python
import logging

logger = logging.getLogger(__name__)

def solve_vrp(problem):
    response = submit(problem)
    job_id = response["jobId"]
    logger.info(f"Submitted VRP job: {job_id}")

    # ... rest of solve logic

    return solution
```

---

## Next Steps

- See the [VRP Guide](../vrp/SKILL.md) for VRP-specific concepts
- See the [Fill Guide](../fill/SKILL.md) for scheduling concepts
- Use `/solvice:solvice-debug` to troubleshoot issues
- Visit [docs.solvice.io](https://docs.solvice.io) for complete API reference
