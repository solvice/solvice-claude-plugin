# Fill Integration Guide

## Skill Definition

- name: fill
- description: Guide for integrating Solvice Fill API - workforce scheduling and shift assignment optimization
- triggers:
  - fill
  - workforce scheduling
  - shift scheduling
  - employee scheduling
  - staff scheduling
  - solvice fill
  - shift assignment

---

## What is Fill?

**Fill** is a workforce scheduling optimization solver that intelligently assigns employees to shifts. Think of it as an automated scheduling manager that considers availability, skills, fairness, and labor rules to create optimal schedules.

### When to Use Fill

Use the Solvice Fill solver when you need to:

- **Schedule employees to shifts**: Assign workers to predefined time slots
- **Match skills to requirements**: Ensure qualified staff for each position
- **Balance workloads**: Distribute shifts fairly across employees
- **Respect availability**: Honor employee preferences and constraints
- **Comply with labor rules**: Enforce rest periods, max hours, and regulations

Fill is ideal when you have **employees with varying availability** who need to be assigned to **shifts with specific requirements**.

---

## Core Entities

### Shifts

A **shift** is a time period that needs to be staffed. Each shift has:

| Field | Description | Example |
|-------|-------------|---------|
| `name` | Unique identifier | `"morning-cashier-001"` |
| `start` | Shift start time | `"2024-01-15T08:00:00Z"` |
| `end` | Shift end time | `"2024-01-15T16:00:00Z"` |
| `skills` | Required qualifications | `["cashier", "customer-service"]` |
| `location` | Where the shift takes place | `"store-downtown"` |
| `demand` | Number of employees needed | `2` |

**Example shift:**
```json
{
  "name": "morning-nurse-icu",
  "start": "2024-01-15T07:00:00Z",
  "end": "2024-01-15T15:00:00Z",
  "skills": ["rn", "icu-certified"],
  "location": "hospital-main",
  "demand": 3
}
```

### Employees

An **employee** is a worker who can be assigned to shifts. Each employee has:

| Field | Description | Example |
|-------|-------------|---------|
| `name` | Unique identifier | `"emp-sarah"` |
| `skills` | Qualifications the employee has | `["rn", "icu-certified", "pediatric"]` |
| `availability` | When they can work | See below |
| `preferences` | Shift preferences (like/dislike) | Optional |
| `maxHours` | Maximum hours per period | `40` |
| `minHours` | Minimum hours per period | `20` |

**Example employee:**
```json
{
  "name": "sarah-johnson",
  "skills": ["rn", "icu-certified"],
  "availability": [
    {"day": "monday", "start": "07:00", "end": "19:00"},
    {"day": "tuesday", "start": "07:00", "end": "19:00"},
    {"day": "wednesday", "start": "07:00", "end": "19:00"}
  ],
  "maxHours": 36,
  "minHours": 24
}
```

### Skills

Skills are qualifications or certifications that define what work an employee can do:

```json
{
  "skills": [
    {"name": "cashier", "description": "Can operate cash register"},
    {"name": "supervisor", "description": "Can manage floor operations"},
    {"name": "forklift", "description": "Certified forklift operator"}
  ]
}
```

Shifts require skills, and employees possess skills. The solver matches them appropriately.

---

## Use Cases

### Healthcare Scheduling

Schedule nurses and caregivers with certifications:

```json
{
  "shifts": [
    {"name": "day-icu-1", "skills": ["rn", "icu"], "start": "07:00", "end": "19:00"},
    {"name": "night-icu-1", "skills": ["rn", "icu"], "start": "19:00", "end": "07:00"}
  ],
  "employees": [
    {"name": "nurse-a", "skills": ["rn", "icu", "pediatric"]},
    {"name": "nurse-b", "skills": ["rn", "icu"]}
  ]
}
```

**Considerations**:
- Enforce minimum rest between shifts (12 hours)
- Match certifications (ICU, pediatric, etc.)
- Balance night shift distribution fairly

### Retail Staffing

Optimize store coverage with part-time availability:

```json
{
  "shifts": [
    {"name": "morning-floor", "skills": ["sales"], "demand": 3},
    {"name": "afternoon-floor", "skills": ["sales"], "demand": 4},
    {"name": "evening-floor", "skills": ["sales", "closing"], "demand": 2}
  ],
  "employees": [
    {"name": "alex", "skills": ["sales"], "availability": "mornings-only"},
    {"name": "jordan", "skills": ["sales", "closing"], "availability": "evenings"}
  ]
}
```

**Considerations**:
- Staff more heavily during peak hours
- Ensure closing-qualified staff for evening shifts
- Honor student/part-time availability

### Hospitality

Schedule restaurant staff across positions:

```json
{
  "shifts": [
    {"name": "lunch-kitchen", "skills": ["cook"], "demand": 2},
    {"name": "lunch-floor", "skills": ["server"], "demand": 4},
    {"name": "lunch-bar", "skills": ["bartender"], "demand": 1}
  ]
}
```

**Considerations**:
- Cross-train employees for multiple positions
- Handle last-minute availability changes
- Balance weekend shifts fairly

---

## Workload Balancing and Fairness

Fill includes built-in fairness mechanisms:

### Even Distribution

Spread shifts evenly across employees:

```json
{
  "options": {
    "balancing": {
      "enabled": true,
      "scope": "weekly"
    }
  }
}
```

### Group-Based Fairness

Balance within employee groups (e.g., full-time vs. part-time):

```json
{
  "employees": [
    {"name": "emp-1", "group": "full-time"},
    {"name": "emp-2", "group": "part-time"}
  ],
  "options": {
    "balancing": {
      "withinGroups": true
    }
  }
}
```

### Weekend Fairness

Distribute weekend shifts equitably:

```json
{
  "options": {
    "weekendBalancing": true
  }
}
```

---

## Constraints and Rules

### Rest Requirements

Enforce minimum rest between shifts:

```json
{
  "rules": {
    "minRestBetweenShifts": 43200  // 12 hours in seconds
  }
}
```

### Maximum Hours

Limit hours per week:

```json
{
  "employees": [
    {"name": "emp-1", "maxHoursPerWeek": 40}
  ]
}
```

### Shift Patterns

Define preferred or prohibited sequences:

```json
{
  "rules": {
    "prohibitedPatterns": [
      {"sequence": ["night", "morning"]}  // No morning shift after night shift
    ]
  }
}
```

### Locked Assignments

Pre-assign certain employee-shift combinations:

```json
{
  "locks": [
    {"employee": "sarah", "shift": "morning-icu-monday"}
  ]
}
```

Locks are respected as hard constraints (unless you specify them as preferences).

---

## Building Your First Schedule

### Step 1: Define your shifts

```json
{
  "shifts": [
    {
      "name": "monday-morning",
      "start": "2024-01-15T08:00:00Z",
      "end": "2024-01-15T14:00:00Z",
      "skills": ["cashier"]
    },
    {
      "name": "monday-afternoon",
      "start": "2024-01-15T14:00:00Z",
      "end": "2024-01-15T20:00:00Z",
      "skills": ["cashier"]
    },
    {
      "name": "monday-evening",
      "start": "2024-01-15T18:00:00Z",
      "end": "2024-01-15T22:00:00Z",
      "skills": ["cashier", "closing"]
    }
  ]
}
```

### Step 2: Define your employees

```json
{
  "employees": [
    {
      "name": "alice",
      "skills": ["cashier", "closing"],
      "availability": [
        {"day": "monday", "start": "08:00", "end": "22:00"}
      ]
    },
    {
      "name": "bob",
      "skills": ["cashier"],
      "availability": [
        {"day": "monday", "start": "08:00", "end": "18:00"}
      ]
    },
    {
      "name": "carol",
      "skills": ["cashier"],
      "availability": [
        {"day": "monday", "start": "12:00", "end": "22:00"}
      ]
    }
  ]
}
```

### Step 3: Combine and solve

```json
{
  "shifts": [...],
  "employees": [...],
  "options": {
    "balancing": {"enabled": true}
  }
}
```

### Step 4: Send the request

```bash
curl -X POST https://api.solvice.io/v2/fill/solve \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d @request.json
```

### Step 5: Poll for results

```bash
# Check status
curl https://api.solvice.io/v2/fill/jobs/{job-id}/status \
  -H "Authorization: Bearer YOUR_API_KEY"

# Get solution when ready
curl https://api.solvice.io/v2/fill/jobs/{job-id}/solution \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Interpreting Solutions

A successful solution shows assignments:

```json
{
  "status": "SOLVED",
  "score": {
    "hardScore": 0,
    "mediumScore": 0,
    "softScore": -150
  },
  "assignments": [
    {"shift": "monday-morning", "employee": "bob"},
    {"shift": "monday-afternoon", "employee": "carol"},
    {"shift": "monday-evening", "employee": "alice"}
  ],
  "unassigned": []
}
```

### Understanding Results

- **All shifts assigned**: Check that `unassigned` is empty
- **hardScore = 0**: All hard constraints satisfied
- **Assignments make sense**: Verify skill matching and availability

### Common Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| Shift unassigned | No available employee with required skills | Add qualified employees or relax skill requirements |
| Employee overworked | Too few employees for demand | Add employees or reduce shift count |
| Unfair distribution | Balancing not enabled | Enable balancing in options |

---

## Demand-Based Scheduling

Instead of pre-defining shifts, specify coverage needs:

```json
{
  "demands": [
    {
      "period": {"start": "2024-01-15T08:00:00Z", "end": "2024-01-15T12:00:00Z"},
      "skill": "cashier",
      "count": 3
    },
    {
      "period": {"start": "2024-01-15T12:00:00Z", "end": "2024-01-15T14:00:00Z"},
      "skill": "cashier",
      "count": 5  // Lunch rush
    }
  ]
}
```

The solver creates optimal shifts to meet demand while minimizing costs.

---

## Next Steps

- See the [Integration Guide](../integration/SKILL.md) for authentication and workflow patterns
- Explore advanced features at [docs.solvice.io](https://docs.solvice.io)
- Use `/solvice:solvice-debug` to troubleshoot scheduling issues
