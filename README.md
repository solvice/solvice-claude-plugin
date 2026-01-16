# Solvice Plugin for Claude Code

A Claude Code plugin that helps developers integrate the Solvice optimization API into their applications. Supports both VRP (Vehicle Routing Problem) and Fill (workforce scheduling) solvers.

**Target audience**: Developers new to Solvice who want guidance on API integration.

## Installation

### Prerequisites

- [Claude Code CLI](https://code.claude.com) installed
- A Solvice API key (get one at [solvice.io](https://solvice.io))

### Install the Plugin

```bash
# Clone the repository
git clone https://github.com/solvice/claude-plugin.git

# Add to Claude Code
claude plugins add /path/to/claude-plugin
```

### Configure Authentication

Set your Solvice API key as an environment variable:

```bash
# Add to your shell profile (~/.bashrc, ~/.zshrc, etc.)
export SOLVICE_API_KEY="your-api-key-here"
```

Or create a `.env` file in your project (add to `.gitignore`!):

```
SOLVICE_API_KEY=your-api-key-here
```

## Features

| Feature | Description |
|---------|-------------|
| **MCP Integration** | Direct access to Solvice API through Claude Code |
| **VRP Solver** | Route optimization for delivery, field service, and logistics |
| **Fill Solver** | Workforce scheduling and shift assignment |
| **Guided Integration** | Step-by-step help for building API requests |
| **Debugging Tools** | Analyze infeasible solutions and constraint violations |
| **Explainable AI** | Understand why the solver made specific decisions |

## Quick Start

### 1. Start a Conversation

```
You: Help me integrate Solvice VRP for my delivery app
```

Claude will guide you through:
- Understanding if VRP is right for your use case
- Building your first API request
- Handling the async workflow
- Interpreting the solution

### 2. Use Commands

```
/solvice:solvice vrp     # Get help with VRP integration
/solvice:solvice fill    # Get help with Fill integration
/solvice:solvice-debug   # Debug optimization issues
```

### 3. Build Your First Request

Here's a minimal VRP example to get started:

```json
{
  "jobs": [
    {
      "name": "delivery-1",
      "location": {"lat": 51.054, "lng": 3.717},
      "duration": 300
    },
    {
      "name": "delivery-2",
      "location": {"lat": 51.062, "lng": 3.725},
      "duration": 300
    }
  ],
  "resources": [
    {
      "name": "van-1",
      "shifts": [{
        "start": "2024-01-15T08:00:00Z",
        "end": "2024-01-15T17:00:00Z",
        "startLocation": {"lat": 51.05, "lng": 3.72},
        "endLocation": {"lat": 51.05, "lng": 3.72}
      }]
    }
  ]
}
```

## Commands

### `/solvice:solvice`

Interactive integration helper that guides you through:

- Choosing between VRP and Fill solvers
- Building correct API request payloads
- Setting up authentication
- Handling async/sync workflows
- Interpreting solutions

**Usage:**
```
/solvice:solvice           # General help
/solvice:solvice vrp       # VRP-specific help
/solvice:solvice fill      # Fill-specific help
```

### `/solvice:solvice-debug`

Troubleshooting assistant for optimization problems:

- Diagnose why jobs are unassigned
- Interpret constraint violations
- Analyze API errors
- Suggest fixes

**Usage:**
```
/solvice:solvice-debug              # Start debugging session
/solvice:solvice-debug job-abc123   # Debug specific job
```

## Skills (Documentation)

The plugin includes comprehensive documentation that Claude can reference:

### VRP Guide (`skills/solvice/vrp/`)

- What is VRP and when to use it
- Core entities: Jobs, Resources, Locations
- Constraint hierarchy (hard/medium/soft)
- Capacity management (single and multi-dimensional)
- Time windows and breaks
- Skill matching with tags
- Step-by-step tutorial: "Build your first route"
- Common mistakes and how to avoid them

### Fill Guide (`skills/solvice/fill/`)

- What is Fill and when to use it
- Core entities: Shifts, Employees, Skills
- Use cases: healthcare, retail, hospitality
- Workload balancing and fairness
- Step-by-step tutorial: "Schedule your first shift"

### Integration Guide (`skills/solvice/integration/`)

- Authentication setup
- Async workflow: solve → poll → get solution
- Sync workflow: when and how to use it
- Reading solutions and scores
- Explainable AI features
- Error handling
- Code examples in Python, TypeScript, and curl

## API Reference

### VRP Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v2/vrp/solve` | POST | Submit async optimization |
| `/v2/vrp/sync/solve` | POST | Submit sync optimization |
| `/v2/vrp/evaluate` | POST | Score existing solution |
| `/v2/vrp/suggest` | POST | Insert new job into routes |
| `/v2/vrp/jobs/{id}/status` | GET | Check job status |
| `/v2/vrp/jobs/{id}/solution` | GET | Get solution |
| `/v2/vrp/jobs/{id}/explanation` | GET | Get AI explanation |

### Fill Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v2/fill/solve` | POST | Submit async scheduling |
| `/v2/fill/jobs/{id}/status` | GET | Check job status |
| `/v2/fill/jobs/{id}/solution` | GET | Get solution |
| `/v2/fill/jobs/{id}/explanation` | GET | Get AI explanation |

### Authentication

All requests require an API key in the Authorization header:

```
Authorization: Bearer YOUR_API_KEY
```

## Examples

### Using the SDK (Recommended)

**Python:**
```bash
pip install --pre solvice-vrp-solver
```

```python
from solvice_vrp_solver import SolviceVrpSolver

client = SolviceVrpSolver(api_key="your_api_key")

solution = client.vrp.sync_solve({
    "jobs": [
        {"name": "delivery-1", "location": {"latitude": 51.05, "longitude": 3.73}, "duration": 900}
    ],
    "resources": [{
        "name": "vehicle-1",
        "shifts": [{"from": "2024-01-15T08:00:00Z", "to": "2024-01-15T17:00:00Z", "start": {"latitude": 51.05, "longitude": 3.72}}]
    }]
})

print(solution.routes)
```

**TypeScript:**
```bash
npm install solvice-vrp-solver
```

```typescript
import SolviceVrpSolver from 'solvice-vrp-solver';

const client = new SolviceVrpSolver({ apiKey: process.env.SOLVICE_API_KEY });
const solution = await client.vrp.syncSolve({ jobs: [...], resources: [...] });
console.log(solution.routes);
```

### Raw HTTP API

If you prefer direct HTTP calls or need to use another language, see the [API documentation](https://docs.solvice.io) for the full REST API reference.

```python
import os, time, requests

API_KEY = os.environ["SOLVICE_API_KEY"]
BASE_URL = "https://api.solvice.io/v2"

# Submit, poll, fetch
response = requests.post(f"{BASE_URL}/vrp/solve", headers={"Authorization": f"Bearer {API_KEY}"}, json=problem)
job_id = response.json()["jobId"]

while requests.get(f"{BASE_URL}/vrp/jobs/{job_id}/status", headers={"Authorization": f"Bearer {API_KEY}"}).json()["status"] != "SOLVED":
    time.sleep(2)

solution = requests.get(f"{BASE_URL}/vrp/jobs/{job_id}/solution", headers={"Authorization": f"Bearer {API_KEY}"}).json()
```

### curl: Quick Test

```bash
# Submit
JOB_ID=$(curl -s -X POST "https://api.solvice.io/v2/vrp/solve" \
  -H "Authorization: Bearer $SOLVICE_API_KEY" \
  -H "Content-Type: application/json" \
  -d @request.json | jq -r '.jobId')

# Check status
curl "https://api.solvice.io/v2/vrp/jobs/$JOB_ID/status" \
  -H "Authorization: Bearer $SOLVICE_API_KEY"

# Get solution
curl "https://api.solvice.io/v2/vrp/jobs/$JOB_ID/solution" \
  -H "Authorization: Bearer $SOLVICE_API_KEY"
```

## Plugin Structure

```
claude-plugin/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest
├── .mcp.json                 # MCP server configuration
├── commands/
│   ├── solvice.md           # Integration helper command
│   └── solvice-debug.md     # Debug command
├── skills/
│   └── solvice/
│       ├── vrp/
│       │   └── SKILL.md     # VRP documentation
│       ├── fill/
│       │   └── SKILL.md     # Fill documentation
│       └── integration/
│           └── SKILL.md     # Integration patterns
├── agents/
│   └── solvice-guide.md     # Beginner assistant agent
└── README.md
```

## Verification

After installation, verify the plugin is working:

1. **Check plugin is loaded:**
   ```
   claude plugins list
   ```
   You should see `solvice` in the list.

2. **Test commands:**
   ```
   /solvice:solvice help
   ```

3. **Test MCP connectivity:**
   If you have a valid `SOLVICE_API_KEY`, try:
   ```
   You: Can you check if the Solvice API is accessible?
   ```

4. **Test skills:**
   ```
   You: Help me integrate Solvice VRP
   ```
   Claude should load the VRP skill and guide you through integration.

## Troubleshooting

### "Plugin not found"

Ensure you added the plugin with the correct path:
```bash
claude plugins add /absolute/path/to/claude-plugin
```

### "Unauthorized" API errors

Check your API key is set correctly:
```bash
echo $SOLVICE_API_KEY
```

### Skills not triggering

Try being more explicit:
```
You: I need help with Solvice VRP integration
```

## Resources

- [Solvice Documentation](https://docs.solvice.io)
- [API Reference](https://api.solvice.io/docs)
- [Solvice Website](https://solvice.io)

## License

MIT

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.
