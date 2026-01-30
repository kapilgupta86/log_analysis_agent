# Multi-Step Query Refinement Guide

This document describes how the Log Analysis Agent handles ambiguous queries and provides timestamp hints to users.

## 1. Ambiguity Detection Logic

The agent evaluates every user query against a set of "Clarity Criteria":
- **Target Component**: Is a specific service, pod, or node mentioned?
- **Temporal Context**: Is a specific time or duration provided?
- **Environment**: Is a namespace or location specified?
- **Action/Error Type**: Is the specific issue clear (e.g., OOM, crash, timeout)?

### Ambiguity Scoring
- Score < 0.4: Highly specific (Proceed with RCA)
- Score 0.4 - 0.7: Partial information (Proceed with assumptions, but notify user)
- Score > 0.7: Ambiguous (Trigger interactive clarification)

## 2. Interactive Clarification Workflow

When a query is ambiguous, the agent will present a structured response:

```text
I understand you want to analyze [Summary of Query], but I need more details to be accurate.

Please clarify:
1. Which service or node? (e.g., payment-service, OLT-01)
2. Which namespace? (e.g., prod, staging)
3. When did this happen? (e.g., '2026-01-23 14:30', 'last hour')
```

## 3. Timestamp and Date Format Hints

### Default Behavior
If no timestamp is given, the agent assumes **Real-Time** context:
- **Search Range**: `now - 15 minutes` to `now + 15 minutes`
- **Notification**: "No timestamp provided. Searching logs from [Calculated Start] to [Calculated End]."

### Supported Date Formats
The agent accepts the following formats:
- **Absolute (UTC)**: `2026-01-23 14:30`, `2026-01-23T14:30:00Z`
- **Relative**: `5 minutes ago`, `last 2 hours`, `yesterday`
- **Unix**: `1674432600`

### User Guidance Message
```text
Hint: For better results, provide a timestamp in one of these formats:
- '2026-01-23 14:30' (UTC)
- 'last 30 minutes'
- '2026-01-23T10:00:00Z'
```

## 4. Implementation Example (Python)

```python
class QueryRefiner:
    def evaluate(self, query: str):
        # Logic to check for mentions of services, nodes, and times
        has_resource = detect_resource(query)
        has_time = detect_time(query)
        
        if not has_resource or not has_time:
            return self.generate_clarification_request(has_resource, has_time)
        return None

    def get_time_range(self, query: str):
        # If no time found, return current time +/- 15 mins
        if not detect_time(query):
            start = now() - timedelta(minutes=15)
            end = now() + timedelta(minutes=15)
            return start, end
        return parse_time(query)
```

## 5. Deployment with Interactive Mode

Ensure the following environment variables are set:
```env
ENABLE_INTERACTIVE_MODE=true
AMBIGUITY_THRESHOLD=0.7
DEFAULT_BUFFER_MINUTES=15
```

---
**Status**: Ready for integration into `src/agent.py` and `src/main.py`.
