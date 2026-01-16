# VRP Integration Guide

## Skill Definition

- name: vrp
- description: Guide for integrating Solvice VRP (Vehicle Routing Problem) API - route optimization for delivery, field service, and logistics
- triggers:
  - vrp
  - vehicle routing
  - route optimization
  - delivery routing
  - field service routing
  - logistics optimization
  - solvice vrp

---

## What is VRP?

The **Vehicle Routing Problem (VRP)** is an optimization challenge that determines the most efficient routes for vehicles or field workers to complete a set of tasks. Think of it as "what Google Maps does for a single trip, but for an entire fleet over a full day."

### When to Use VRP

Use the Solvice VRP solver when you need to:

- **Optimize delivery routes**: Assign packages to drivers and sequence stops efficiently
- **Schedule field service**: Route technicians to customer sites with skill requirements
- **Plan logistics operations**: Coordinate pickups and deliveries with capacity constraints
- **Manage mobile workforces**: Assign jobs to workers with time windows and breaks

VRP is ideal when you have **multiple resources** (vehicles, drivers, workers) that need to visit **multiple locations** within constraints like time windows, capacity limits, and skill requirements.

---

## Core Entities

### Jobs

A **job** is a single task that needs to be completed at a specific location. Each job has:

| Field | Description | Example |
|-------|-------------|---------|
| `name` | Unique identifier | `"delivery-123"` |
| `location` | Coordinates or location reference | `{"lat": 51.05, "lng": 3.72}` |
| `duration` | Service time in seconds | `900` (15 minutes) |
| `timeWindows` | When the job can be serviced | `[{"start": "09:00", "end": "12:00"}]` |
| `load` | Capacity consumed | `{"weight": 50, "volume": 2}` |
| `allowedResources` | Which resources can do this job | `["driver-1", "driver-2"]` |
| `tags` | Required skills/qualifications | `["refrigerated", "heavy-lift"]` |

**Example job:**
```json
{
  "name": "delivery-001",
  "location": {"lat": 51.054, "lng": 3.717},
  "duration": 600,
  "timeWindows": [{"start": "2024-01-15T09:00:00Z", "end": "2024-01-15T12:00:00Z"}],
  "load": {"packages": 3},
  "tags": ["standard"]
}
```

### Resources

A **resource** is a vehicle, driver, or worker that can perform jobs. Each resource has:

| Field | Description | Example |
|-------|-------------|---------|
| `name` | Unique identifier | `"van-1"` |
| `shifts` | Working periods with start/end locations | See below |
| `capacity` | Maximum load the resource can carry | `{"packages": 100}` |
| `tags` | Skills/capabilities this resource has | `["refrigerated", "heavy-lift"]` |

**Example resource:**
```json
{
  "name": "driver-alice",
  "shifts": [{
    "start": "2024-01-15T08:00:00Z",
    "end": "2024-01-15T17:00:00Z",
    "startLocation": {"lat": 51.05, "lng": 3.72},
    "endLocation": {"lat": 51.05, "lng": 3.72}
  }],
  "capacity": {"packages": 50, "weight": 500},
  "tags": ["standard", "fragile"]
}
```

### Locations

Locations can be specified inline with jobs/resources or in a separate `locations` array for reuse:

```json
{
  "locations": [
    {"name": "depot", "lat": 51.05, "lng": 3.72},
    {"name": "customer-a", "lat": 51.06, "lng": 3.73}
  ]
}
```

Jobs and resources can then reference locations by name:
```json
{"name": "job-1", "location": "customer-a"}
```

---

## Constraint Hierarchy

Solvice uses a **three-tier constraint system**:

### Hard Constraints (Must be satisfied)

These constraints **cannot** be violated. If a job cannot be assigned without breaking a hard constraint, it remains unassigned.

- **Capacity limits**: Resource cannot exceed its capacity
- **Time windows** (when configured as hard): Job must be serviced within the window
- **Required tags**: Resource must have all required tags
- **Allowed/disallowed resources**: Job can only go to permitted resources

### Medium Constraints (Should be satisfied)

These constraints are **strongly preferred** but can be violated if necessary. The solver minimizes violations.

- **Soft time windows**: Penalized if violated but allowed
- **Preferred resources**: Job prefers certain resources but accepts others
- **Break preferences**: Breaks should occur at preferred times/locations

### Soft Constraints (Nice to have)

These are optimization objectives that the solver tries to improve:

- **Minimize travel time**: Reduce total driving distance
- **Minimize overtime**: Keep resources within normal hours
- **Balance workload**: Distribute jobs evenly across resources
- **Cluster geographically**: Group nearby jobs together

---

## Capacity Management

### Single-Dimension Capacity

Simple capacity with one unit type:

```json
// Resource
{"name": "van-1", "capacity": {"items": 100}}

// Job
{"name": "delivery-1", "load": {"items": 5}}
```

### Multi-Dimensional Capacity

Track multiple capacity dimensions simultaneously:

```json
// Resource
{
  "name": "truck-1",
  "capacity": {
    "weight": 1000,
    "volume": 50,
    "pallets": 10
  }
}

// Job
{
  "name": "shipment-1",
  "load": {
    "weight": 250,
    "volume": 12,
    "pallets": 2
  }
}
```

The solver ensures **no dimension exceeds its limit** at any point in the route.

---

## Time Windows and Breaks

### Job Time Windows

Specify when a job can be serviced:

```json
{
  "name": "morning-delivery",
  "timeWindows": [
    {"start": "2024-01-15T08:00:00Z", "end": "2024-01-15T12:00:00Z"}
  ]
}
```

Multiple windows allow flexibility:
```json
{
  "name": "flexible-delivery",
  "timeWindows": [
    {"start": "2024-01-15T09:00:00Z", "end": "2024-01-15T11:00:00Z"},
    {"start": "2024-01-15T14:00:00Z", "end": "2024-01-15T16:00:00Z"}
  ]
}
```

### Resource Breaks

Configure mandatory breaks for drivers:

```json
{
  "name": "driver-1",
  "shifts": [{
    "start": "2024-01-15T08:00:00Z",
    "end": "2024-01-15T17:00:00Z",
    "breaks": [{
      "duration": 1800,
      "timeWindow": {"start": "2024-01-15T12:00:00Z", "end": "2024-01-15T13:00:00Z"}
    }]
  }]
}
```

---

## Skill Matching with Tags

Use tags to match job requirements with resource capabilities:

```json
// Resource with skills
{
  "name": "technician-senior",
  "tags": ["electrical", "hvac", "certified"]
}

// Job requiring specific skills
{
  "name": "ac-repair",
  "tags": ["hvac", "certified"]
}
```

The resource must have **all tags** specified on the job to be eligible.

---

## Building Your First Route

### Step 1: Define your jobs

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
    },
    {
      "name": "delivery-3",
      "location": {"lat": 51.048, "lng": 3.731},
      "duration": 300
    }
  ]
}
```

### Step 2: Define your resources

```json
{
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

### Step 3: Combine and solve

```json
{
  "jobs": [...],
  "resources": [...],
  "options": {
    "routing": {"engine": "osm"}
  }
}
```

### Step 4: Send the request

```bash
curl -X POST https://api.solvice.io/v2/vrp/solve \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d @request.json
```

### Step 5: Poll for results

The async endpoint returns a job ID. Poll the status endpoint:

```bash
curl https://api.solvice.io/v2/vrp/jobs/{job-id}/status \
  -H "Authorization: Bearer YOUR_API_KEY"
```

When status is `"SOLVED"`, fetch the solution:

```bash
curl https://api.solvice.io/v2/vrp/jobs/{job-id}/solution \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Common Mistakes and How to Avoid Them

### 1. Impossible Time Windows

**Problem**: Job time window is shorter than the job duration.
```json
// BAD: 15-minute window for a 30-minute job
{"duration": 1800, "timeWindows": [{"start": "09:00", "end": "09:15"}]}
```
**Fix**: Ensure the window is at least as long as the duration.

### 2. No Eligible Resources

**Problem**: Job requires tags that no resource has.
```json
// Job requires "certified" tag
{"tags": ["certified"]}
// But no resource has this tag
```
**Fix**: Add the required tag to at least one resource, or remove the tag requirement.

### 3. Capacity Exceeded

**Problem**: Total job loads exceed available resource capacity.
**Fix**: Add more resources or reduce job loads.

### 4. Time Window vs. Shift Mismatch

**Problem**: Job time window is outside all resource shift times.
```json
// Job must be done 6-7 PM
{"timeWindows": [{"start": "18:00", "end": "19:00"}]}
// But all shifts end at 5 PM
```
**Fix**: Extend shift times or adjust job time windows.

### 5. Missing Locations

**Problem**: Jobs reference locations that don't exist.
**Fix**: Ensure all location references are valid, or use inline coordinates.

---

## Interpreting Solutions

A successful solution includes:

```json
{
  "status": "SOLVED",
  "score": {
    "hardScore": 0,
    "mediumScore": 0,
    "softScore": -12345
  },
  "resources": [
    {
      "name": "van-1",
      "route": [
        {"type": "start", "location": "depot", "arrival": "08:00"},
        {"type": "job", "name": "delivery-1", "arrival": "08:15", "departure": "08:20"},
        {"type": "job", "name": "delivery-3", "arrival": "08:35", "departure": "08:40"},
        {"type": "end", "location": "depot", "arrival": "09:00"}
      ]
    }
  ],
  "unassigned": [
    {"name": "delivery-2", "reason": "No eligible resource with required tags"}
  ]
}
```

### Understanding Scores

- **hardScore = 0**: All hard constraints satisfied
- **hardScore < 0**: Some hard constraints violated (bad!)
- **mediumScore**: Medium constraint violations
- **softScore**: Lower (more negative) is worse; represents optimization cost

### Unassigned Jobs

Check the `unassigned` array to understand why jobs couldn't be assigned. Use the `/explanation` endpoint for detailed reasoning.

---

## Next Steps

- See the [Integration Guide](../integration/SKILL.md) for authentication and workflow patterns
- Use `/solvice:solvice-debug` to troubleshoot infeasible solutions
- Check [docs.solvice.io](https://docs.solvice.io) for advanced features
