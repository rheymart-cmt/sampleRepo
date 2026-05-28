---
input data:
{{ $binary.data }}


---
# Automation Failure Monitoring_Rh

## 1. Overview

### Purpose
Monitors workflow execution failures and automation anomalies, formats error details, filters noise, and sends Slack alerts for critical issues and data-quality reports.

### Use Case
This workflow provides centralized operational monitoring for n8n automations: it catches real-time execution failures (Error Trigger), periodically audits recent executions for elevated failure rates or workflow downtime (Schedule + HTTP request + analysis), and accepts explicit bad-output reports from other workflows (Webhook). The automation reduces mean time to detection by notifying a Slack channel with context and stack traces, and helps surface data-quality problems for immediate investigation.

### Trigger Type
**Trigger:** Event-based (primary) with optional Schedule and Webhook components

**Details:**
- Event-based (Error Trigger)
  - Trigger node: Error Trigger (catches workflow execution failures in real-time; no extra HTTP settings)
  - Authentication: uses n8n internal event system (no external auth)
- Schedule (disabled in this copy)
  - Node name: Schedule — Every 5 Minutes (disabled)
  - Intended frequency: every 5 minutes (sticky note and node name indicate 5-minute cadence)
  - Implementation detail: scheduleTrigger node with "minutes" interval rule
- Webhook (disabled in this copy)
  - Node name: Webhook — Bad Output Report (disabled)
  - HTTP method: POST
  - Path: `/bad-output-report` (node parameter: path: "bad-output-monitor", parameters2.path: "bad-output-report")
  - Response mode: `lastNode`
  - Authentication: not specified (open webhook expected to be called by internal workflows)

---

## 2. Node-Level Documentation

### Workflow Architecture
Trigger (Error Trigger / Schedule / Webhook)
→ Format / Validate (Set / Code)
→ Filter / Analyze (If / Code)
→ Notify (Slack nodes)
→ Output (Slack channel messages)

Node flow paths:
- Error Trigger → Format Error Details (Set) → Filter Non-Critical Errors (If) → Slack — Error Alert
- Schedule (every 5m, disabled) → Fetch Recent Executions (HTTP Request, disabled) → Analyze Executions for Anomalies (Code, disabled) → Anomaly Found? (If, disabled) → Slack Anomaly Alert OR Slack Health OK (both disabled)
- Webhook /bad-output-report (disabled) → Validate Bad Output Payload (Code, disabled) → Slack Bad Output Alert (disabled)

### Node Breakdown

#### Error Trigger - n8n-nodes-base.errorTrigger
**Purpose:** Entry point for real-time failure detection; fires when any workflow execution errors.

**Configuration:**
- Type: Error Trigger
- No additional parameters shown

**Logic & Business Rules:**
- Listens for workflow execution error events emitted by n8n.
- All caught events are passed to the Set node to produce a standardized error payload.

**Data Transformations:**
- None in this node; raw event forwarded.

**Error Handling:**
- This node is itself the error source; downstream nodes handle filtering and notification.

---

#### Format Error Details - n8n-nodes-base.set
**Purpose:** Normalize and extract key fields from the Error Trigger payload to a predictable JSON shape for filtering and notification.

**Configuration:**
- Node name: Format Error Details
- Assignments (key fields created):
  - workflowName: `={{ $json.workflow.name }}`
  - workflowId: `={{ $json.workflow.id }}`
  - executionId: `={{ $json.execution.id }}`
  - errorMessage: `={{ $json.execution.error.message }}`
  - errorStack: `={{ $json.execution.error.stack ?? 'N/A' }}`
  - timestamp: `={{ $now.toISO() }}`
  - severity: `CRITICAL`

**Logic & Business Rules:**
- Uses templated expressions to map nested event fields into top-level keys.
- Defaulting: `errorStack` uses `?? 'N/A'` to avoid null/undefined values.
- Sets a static severity "CRITICAL" for execution failures.

**Data Transformations:**
- Expression-based extraction (example):
  - `workflowName` = value of `$json.workflow.name`
  - `timestamp` uses `$now.toISO()` (current ISO timestamp)

**Error Handling:**
- Not applicable inside Set node; it simply creates fields used by downstream If and Slack nodes.

---

#### Filter Non-Critical Errors - n8n-nodes-base.if
**Purpose:** Filter out non-actionable errors to reduce alert noise (e.g., errors deliberately labelled IGNORE).

**Configuration:**
- Node name: Filter Non-Critical Errors
- Condition: string notContains
  - Left: `={{ $json.errorMessage }}`
  - Right: `IGNORE`
  - Operator: `notContains`
  - Case sensitive: true

**Logic & Business Rules:**
- Passes only events where the error message does NOT contain the literal string "IGNORE".
- This allows workflows to intentionally include "IGNORE" in error messages to suppress alerts (noise suppression rule).

**Data Transformations:**
- No transformation; conditional routing only.

**Error Handling:**
- When condition is false, items are dropped (not routed to Slack).

---

#### Slack — Error Alert - n8n-nodes-base.slack
**Purpose:** Post formatted critical error alerts to a Slack user (direct message) or channel.

**Configuration:**
- Node name: Slack — Error Alert
- Select: user
- User: U0AKSKBPV8C (cachedResultName: rheymart)
- Message text (uses n8n expression templating):
  - Contains workflow name, workflow id, execution id, error message, timestamp, severity and stack trace in a code block.
  - Example template fragment:
    ```
    *Workflow:* {{ $json.workflowName }}
    *Execution ID:* {{ $json.executionId }}
    *Error:* {{ $json.errorMessage }}
    *Stack Trace:*
    ```{{ $json.errorStack }}```
    ```
- Options: includeLinkToWorkflow = false, mrkdwn = true
- Credentials: Slack API credential named "Slack Interns App"

**Logic & Business Rules:**
- Only invoked when Filter Non-Critical Errors condition passes.
- Intended recipient is a specific user (likely on-call person).

**Data Transformations:**
- Uses placeholders referencing fields created by the Set node.

**API/Rate Limits:**
- Uses Slack API via configured credentials. No retry, backoff, or rate-limit handling configured in this node. Respect Slack app rate limits in production.

**Error Handling:**
- Standard n8n node behavior: if Slack API returns an error, workflow execution will fail (unless workflow-level error handling implemented elsewhere).

---

#### Schedule — Every 5 Minutes - n8n-nodes-base.scheduleTrigger (disabled)
**Purpose:** Periodic audit trigger to fetch recent executions and detect anomalies/downtime.

**Configuration:**
- Node name: Schedule — Every 5 Minutes
- Rule: interval: minutes (intended every 5 minutes via node name/sticky note)
- Status: disabled in this exported workflow

**Logic & Business Rules:**
- Intended to kick off a 1-hour lookback analysis every 5 minutes.
- Disabled here; enable to use the periodic checks.

---

#### Fetch Recent Executions - n8n-nodes-base.httpRequest (disabled)
**Purpose:** Query n8n internal API for recent executions to evaluate failure rates and uptime of critical workflows.

**Configuration:**
- Method: GET (implicit via URL)
- URL: `={{ $env.N8N_HOST }}/api/v1/executions`
- Query parameters:
  - `limit=50`
  - `includeData=false`
- Headers:
  - `X-N8N-API-KEY`: (JWT string present in node; treat as sensitive)
    - Value in export: `=eyJhbGci...` (a JWT — do not commit to public repos)
- sendQuery: true, sendHeaders: true

**Logic & Business Rules:**
- Requests up to 50 recent executions, excluding full execution data to reduce payload size.
- Relies on N8N_HOST env var and a API key header.
- Disabled in this export.

**API/Rate Limits:**
- Calls local n8n API. Respect API usage quotas. No retry/backoff logic configured.

**Error Handling:**
- No explicit retry; HTTP errors will cause the workflow branch to fail.

---

#### Analyze Executions for Anomalies - n8n-nodes-base.code (JavaScript) (disabled)
**Purpose:** Compute failure-rate and detect workflow downtime based on recent execution list.

**Configuration:**
- Node name: Analyze Executions for Anomalies
- Type: Code (JavaScript), returns a single structured JSON object

**Core logic (extracted):**
```javascript
const executions = $input.first().json.data || [];
const now = new Date();
const oneHourAgo = new Date(now - 60 * 60 * 1000);
const fifteenMinutesAgo = new Date(now - 15 * 60 * 1000);

// Failure rate over last 1 hour
const recentExecutions = executions.filter(e => new Date(e.startedAt) > oneHourAgo);
const failedRecent = recentExecutions.filter(e => e.status === 'error');
const failureRate = recentExecutions.length > 0
  ? (failedRecent.length / recentExecutions.length) * 100
  : 0;
const highFailureRate = failureRate > 20;

// Downtime detection for critical workflows
const criticalWorkflows = (process.env.CRITICAL_WORKFLOWS || '').split(',').map(s => s.trim()).filter(Boolean);
const recentWorkflowNames = executions
  .filter(e => new Date(e.startedAt) > fifteenMinutesAgo)
  .map(e => e.workflowData?.name || '');
const downtimeWorkflows = criticalWorkflows.filter(name => !recentWorkflowNames.includes(name));
const hasDowntime = downtimeWorkflows.length > 0;

const anomalyDetected = highFailureRate || hasDowntime;

return [{
  json: {
    anomalyDetected,
    highFailureRate,
    failureRate: failureRate.toFixed(1),
    totalRecent: recentExecutions.length,
    failedCount: failedRecent.length,
    hasDowntime,
    downtimeWorkflows,
    timestamp: now.toISOString()
  }
}];
```

**Logic & Business Rules:**
- Failure rate computed from executions started within last 1 hour.
- High failure threshold: > 20% (hard-coded).
- Downtime: any "critical" workflow (from env var CRITICAL_WORKFLOWS) not seen in last 15 minutes is considered down.
- Emits structured flags and metrics used by downstream If node and Slack messages.

**Data Transformations:**
- Aggregation, filtering by timestamp, status comparison, environment-driven list parsing (`CRITICAL_WORKFLOWS`).

**Error Handling:**
- No explicit try/catch in code node: runtime exceptions will fail node. Consider wrapping logic in try/catch and returning an error object for graceful handling.

---

#### Anomaly Found? - n8n-nodes-base.if (disabled)
**Purpose:** Route the schedule-driven branch to anomaly alert (true) or routine health OK (false).

**Configuration:**
- Condition:
  - Left: `={{ $json.anomalyDetected }}`
  - Right: `true`
  - Operator: boolean equal
  - Case sensitive / typeValidation: strict

**Logic & Business Rules:**
- If anomalyDetected === true → Slack — Anomaly Alert
- Else → Slack — Health OK

---

#### Slack — Anomaly Alert (disabled)
**Purpose:** Post a summary message to #n8n-health when the scheduled analysis finds an anomaly.

**Configuration:**
- Select: channel
- Channel: `#n8n-health`
- Message body includes:
  - Time, Failure Rate (last 1hr), counts, whether highFailureRate or downtime, and list of affected workflows when downtime exists
- mrkdwn: true
- Credentials: Slack Interns App

**Logic & Business Rules:**
- Only used when scheduled analysis detects an anomaly.

**Data Transformations:**
- Uses fields emitted by the Analyze Executions for Anomalies code node.

---

#### Slack — Health OK (disabled)
**Purpose:** Optional routine health message when no anomaly is found.

**Configuration:**
- Select: channel: `#n8n-health`
- Message includes execution count and failure rate summary.

**Logic & Business Rules:**
- Sent when analysis passes (no anomaly). Disabled in this export.

---

#### Webhook — Bad Output Report (disabled) - n8n-nodes-base.webhook
**Purpose:** Accept POST reports from other workflows that detect bad outputs or data-quality issues.

**Configuration:**
- path parameter: `bad-output-monitor` (top-level path), parameters2.path: `bad-output-report`
- httpMethod: POST
- responseMode: `lastNode`
- Disabled in this export

**Logic & Business Rules:**
- Intended callers: other workflows should POST a diagnostic payload with fields like workflowName, issue, details, missingFields, outputIsNull, outputIsEmpty.
- Response mode `lastNode` returns output of last connected node (Slack node result) to the caller.

---

#### Validate Bad Output Payload - n8n-nodes-base.code (JavaScript) (disabled)
**Purpose:** Normalize and score incoming bad-output reports and classify severity for alerting.

**Configuration / Core logic (extracted):**
```javascript
const payload = $input.first().json.body || $input.first().json;

const workflowName = payload.workflowName || 'Unknown Workflow';
const issue = payload.issue || 'Unexpected data detected';
const details = payload.details || '{}';
const timestamp = new Date().toISOString();

// anomaly scoring
const isNullOutput = payload.outputIsNull === true;
const isEmptyArray = payload.outputIsEmpty === true;
const isMissingFields = Array.isArray(payload.missingFields) && payload.missingFields.length > 0;

const severity = isNullOutput ? '=4 CRITICAL' : isMissingFields ? '=á WARNING' : '=à ANOMALY';

return [{
  json: {
    workflowName,
    issue,
    details: typeof details === 'object' ? JSON.stringify(details, null, 2) : details,
    severity,
    isNullOutput,
    isEmptyArray,
    isMissingFields,
    missingFields: payload.missingFields || [],
    timestamp
  }
}];
```

**Logic & Business Rules:**
- Severity classification:
  - CRITICAL if `outputIsNull` true
  - WARNING if `missingFields` present
  - ANOMALY otherwise
- Converts details to formatted JSON string when object provided.

**Data Transformations:**
- Normalizes incoming shape and computes boolean flags used in Slack templating.

**Error Handling:**
- No explicit error handling; assumes input conforms to expected shape.

---

#### Slack — Bad Output Alert (disabled)
**Purpose:** Notify a specific Slack user (cached as "rheymart") about data-quality issues coming through the webhook.

**Configuration:**
- Select: user U0AKSKBPV8C (rheymart)
- Message includes workflow, issue, severity, boolean flags, missing fields list, time, and formatted details code block.
- mrkdwn: true
- Credentials: Slack Interns App

**Logic & Business Rules:**
- Only invoked after Validate Bad Output Payload.

---

#### Sticky Note(s)
The workflow contains three sticky notes with documentation and purpose:
- Sticky Note (large)
  - Content excerpt:
    ```
    Automation Failure Monitoring

    Purpose: Monitors failures and anomalies in automation workflow
    Error Trigger:
    Catches all workflow execution failures in real-time

    Note:
    Serves only the Published workflows.
    ```
- Sticky Note1
  - Content excerpt:
    ```
    Automation Failure Monitoring
    Purpose: Monitors failures and anomalies in automation services

    Error Trigger - Catches all workflow execution failures in real-time

    Schedule (every 5 min) - Checks failure rate & downtime of critical workflows

    Webhook (/bad-output-report) - Receives bad data reports from your other workflows
    ```
- Sticky Note2: "### Trigger"
- Sticky Note3: "### Notificiation / Output"

These provide author intent and quick references for operators.

---

## 3. Data Flow & Mapping

### Input Data Structure
**Source:** Error Trigger event payload OR HTTP API response (for schedule analysis) OR webhook POST body.

**Expected Format (Error Trigger example):**
```json
{
  "workflow": {
    "id": "string",
    "name": "string"
  },
  "execution": {
    "id": "string",
    "error": {
      "message": "string",
      "stack": "string"
    }
  }
}
```

**Required Fields:**
- `workflow.name`: human-readable workflow name (used in messages)
  - Validation: non-empty string
- `workflow.id`: workflow identifier
  - Validation: non-empty string
- `execution.id`: execution identifier
  - Validation: non-empty string
- `execution.error.message`: error message string (used for filtering and alert content)
  - Validation: non-empty string

**Optional Fields:**
- `execution.error.stack`: full stack trace (defaults to 'N/A' if missing)
- For schedule branch: HTTP response `data` array where each element contains `startedAt`, `status`, and `workflowData.name`.
- For webhook branch: fields such as `workflowName`, `issue`, `details`, `missingFields` (array), `outputIsNull` (boolean), `outputIsEmpty` (boolean)

---

### Output Data Structure
**Destination:** Slack (user direct message or channel), and potential webhook response to the caller when webhook is used.

**Format (alert payload created by Set/Code nodes):**
```json
{
  "workflowName": "string",
  "workflowId": "string",
  "executionId": "string",
  "errorMessage": "string",
  "errorStack": "string",
  "timestamp": "ISO8601 string",
  "severity": "string"
}
```

For schedule analysis:
```json
{
  "anomalyDetected": true|false,
  "highFailureRate": true|false,
  "failureRate": "numeric string (one decimal)",
  "totalRecent": integer,
  "failedCount": integer,
  "hasDowntime": true|false,
  "downtimeWorkflows": ["name1","name2"],
  "timestamp": "ISO8601 string"
}
```

For bad-output webhook:
```json
{
  "workflowName": "string",
  "issue": "string",
  "details": "string (JSON formatted)",
  "severity": "string",
  "isNullOutput": true|false,
  "isEmptyArray": true|false,
  "isMissingFields": true|false,
  "missingFields": ["field1","field2"],
  "timestamp": "ISO8601 string"
}
```

---

### Field Mapping Table

| Source Field | Transformation | Destination Field | Notes |
|--------------|----------------|-------------------|-------|
| `$json.workflow.name` | Direct pass-through | `workflowName` | from Error Trigger via Set node |
| `$json.workflow.id` | Direct pass-through | `workflowId` | |
| `$json.execution.id` | Direct pass-through | `executionId` | |
| `$json.execution.error.message` | Direct pass-through | `errorMessage` | Used by If filter to ignore messages containing "IGNORE" |
| `$json.execution.error.stack` | `?? 'N/A'` default | `errorStack` | Ensures stack presence |
| `$now.toISO()` | Current timestamp | `timestamp` | ISO string timestamp |
| HTTP GET `/api/v1/executions` response -> `data` | JS code aggregates by startedAt/status | failureRate, totalRecent, failedCount, anomalyDetected, downtimeWorkflows | See Analyze Executions for Anomalies code |
| webhook body `payload.details` | If object, stringify with 2-space indentation | `details` | used in Slack code block |

**Transformation Details:**
- Set node expressions:
```text
workflowName = {{ $json.workflow.name }}
workflowId   = {{ $json.workflow.id }}
executionId  = {{ $json.execution.id }}
errorMessage = {{ $json.execution.error.message }}
errorStack   = {{ $json.execution.error.stack ?? 'N/A' }}
timestamp    = {{ $now.toISO() }}
severity     = "CRITICAL"
```

- Filter condition:
```text
IF $json.errorMessage notContains "IGNORE" (case-sensitive)
```

- Analyze Executions (key snippet shown earlier) — computes failureRate and downtimeWorkflows; highFailureRate threshold = 20%.

- Validate Bad Output Payload: creates severity via ternary logic:
```javascript
const severity = isNullOutput ? '=4 CRITICAL' : isMissingFields ? '=á WARNING' : '=à ANOMALY';
```

---

Additional operational notes
- Slack credential in nodes: `Slack Interns App` (credential id present in export).
- The HTTP Request node sends `X-N8N-API-KEY` header. The export contains a JWT-like value; treat as sensitive and rotate if this file is shared publicly.
- Several nodes (Schedule, Fetch Recent Executions, Analyze Executions for Anomalies, Anomaly alert branch, Webhook and Bad Output branch) are disabled in this exported workflow. To enable periodic monitoring or webhook reporting, enable those nodes and ensure CRITICAL_WORKFLOWS environment variable and API key are configured.
- Consider adding:
  - Retry/backoff on Slack and HTTP Request nodes
  - Centralized error-handling path or a logging sink (e.g., database) for historical alert records
  - Parameterization for thresholds (failure rate percentage, lookback windows) rather than hard-coding.

---