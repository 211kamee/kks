# KKS | KaMee's Krazy Server

## Notices
- For any communication, use Issues tab on this repo
- Need to work on backend architecture's specifc
- <a href="https://kks.kamee.top" target="_blank">Link</a> is not yet relevent but will be updated.

# Current Goal | AI-Driven L1/L2 Support Automation

An AI-powered support system that integrates with ServiceNow to analyze
Incidents/Service Requests, propose approved solutions, execute approved
actions, validate results, and update ServiceNow automatically.

## 0. Tech Stack (short version)

- `client/` — Flutter app: Dashboard + Admin Panel (kill switch, resource/queue management, internal cost/usage view).
- `server/host/` — Rust: API gateway, split auth (SSO for humans, mTLS/JWT for services), policy engine, execution engine, SQLx + Postgres (with pgvector).
- `server/agent/` — Python: AI orchestrator + a single shared MCP server exposing approved solutions as tools.
- Valkey Streams ties the stages (received → proposed → approved → executed → validated) together, single VM, no containers for now.

See [!docs/INFRA.md](!docs/INFRA.md) for the full breakdown and remaining
open questions.

## 1. Core Workflow

``` text
ServiceNow
    ↓
Incident / Service Request
    ↓
AI analyzes
    ↓
Find approved solution
    ↓
Proposed Resolution
    ↓
Human Approval
    ↓
Execution Engine
    ↓
Execute approved actions
    ↓
Validation
    ↓
Resolve / Escalate
```

The core principle is:

**Proposal → Approval → Execution → Validation**

The AI does not receive unrestricted execution access. The execution
layer controls which actions can actually be performed.

## 2. Example

A user creates this Incident in ServiceNow:

``` text
INC10234
VPN is not connecting
Windows 11
Priority: P2
```

### Step 1 --- ServiceNow triggers the AI system

ServiceNow detects the new Incident and sends the relevant information
to the AI backend through an HTTP request.

``` json
{
  "number": "INC10234",
  "description": "VPN is not connecting",
  "priority": "P2",
  "os": "Windows 11"
}
```

### Step 2 --- AI analyzes the Incident

The AI searches the approved Knowledge Base and previous resolved
incidents.

It finds:

``` text
KB001284
Confidence: 94%

Solution:
1. Restart VPN service
2. Clear VPN cache
3. Reconnect
```

### Step 3 --- AI creates a proposal

The dashboard displays:

``` text
INC10234

Proposed Resolution
Confidence: 94%

Actions:
1. Restart VPN service
2. Clear VPN cache
3. Reconnect

[ APPROVE ] [ REJECT ]
```

### Step 4 --- Human approval

An L1/L2 engineer clicks **APPROVE**.

The approval is recorded for auditing.

``` text
Status: APPROVED
Approved by: L2 Engineer
```

### Step 5 --- Execution

The Execution Engine performs only the approved actions.

``` text
Restart VPN service
        ↓
Clear VPN cache
        ↓
Reconnect
```

### Step 6 --- Validation

The AI checks whether the expected result occurred.

``` text
VPN connection: SUCCESS
```

### Step 7 --- ServiceNow is updated

The backend updates the Incident:

``` text
INC10234
State: Resolved
Resolution: VPN connection restored
```

If validation fails, the system does not falsely resolve the ticket. It
can retry according to policy or escalate to L1/L2.

## 3. Operating Modes

### Mode 1 --- Assist

``` text
AI proposes
    ↓
Human approves
    ↓
AI executes
```

Best for the initial deployment.

### Mode 2 --- Supervised Automation

``` text
AI proposes
    ↓
Policy automatically approves low-risk actions
    ↓
AI executes
```

Example: restarting a non-critical service.

### Mode 3 --- Autonomous

``` text
AI selects approved solution
    ↓
Executes
    ↓
Validates
    ↓
Resolves
```

Used only after sufficient confidence, testing, and governance are
established.

## 4. High-Level Architecture

``` text
                 ┌─────────────────┐
                 │    ServiceNow   │
                 │ Incidents / SRs │
                 └────────┬────────┘
                          │
                    HTTP / Event
                          ↓
                 ┌─────────────────┐
                 │   AI Backend    │
                 │                 │
                 │ Analysis        │
                 │ RAG / KB        │
                 │ Reasoning       │
                 │ Policy Engine   │
                 └────────┬────────┘
                          │
                    Proposal
                          ↓
                 ┌─────────────────┐
                 │    Dashboard    │
                 │                 │
                 │ Approve/Reject  │
                 │ AI Sessions     │
                 │ Live Activity   │
                 └────────┬────────┘
                          │
                       Approved
                          ↓
                 ┌─────────────────┐
                 │ Execution Engine│
                 └────────┬────────┘
                          │
             ┌────────────┼────────────┐
             ↓            ↓            ↓
          Servers       APIs        Endpoints
             │            │            │
             └────────────┼────────────┘
                          ↓
                     Validation
                          ↓
                 ┌─────────────────┐
                 │    ServiceNow   │
                 │ Update / Resolve│
                 └─────────────────┘
```

## 5. Main Components

### ServiceNow

The source of Incidents and Service Requests.

It can trigger the AI platform when relevant records are created or
updated.

Possible integration mechanisms include:

-   Business Rules
-   Flow Designer
-   REST integrations
-   ServiceNow APIs

### AI Backend

Responsible for:

-   Receiving ServiceNow events
-   Understanding the Incident
-   Searching the Knowledge Base
-   Finding relevant solutions
-   Generating proposals
-   Assigning confidence
-   Producing execution plans

### Knowledge Base

Contains approved solutions, runbooks, previous resolutions, and
troubleshooting procedures.

### Policy Engine

Determines whether an action requires:

-   Human approval
-   Automatic approval
-   Escalation

Example:

``` text
Restart application service → Low risk → Auto-approved

Delete production database → Critical → Never auto-approved
```

### Execution Engine

The only component allowed to perform operational actions.

It receives an approved action plan and executes it through controlled
integrations.

### Validation Engine

Checks whether the expected outcome occurred.

``` text
Expected:
VPN connection successful

Actual:
VPN connection successful

Result:
PASS
```

### Dashboard

Provides a real-time view of:

``` text
Pending AI Approval
AI Executing
AI Validation
Human L1
Human L2
Escalated

AI Sessions
Incident Status
Execution History
Approval History
```

## 6. Example AI Session

``` text
Session: S-92832
Incident: INC10234

Status:
Executing

Proposal:
KB001284

Approved Actions:
✓ Restart VPN service
✓ Clear VPN cache
✓ Reconnect

Validation:
PASS

Final Result:
Resolved
```

## 7. Safety and Governance

Every automated action should have:

-   Explicit permissions
-   Action allowlists
-   Risk classification
-   Approval requirements
-   Audit logs
-   Execution logs
-   Validation criteria
-   Rollback/recovery procedures

The system should never allow an AI model to freely execute arbitrary
commands.

## 8. Goal

The long-term goal is to gradually automate L1/L2 support:

``` text
Human-heavy
    ↓
AI-assisted
    ↓
Supervised automation
    ↓
Controlled autonomous support
```

while maintaining **security, auditability, explainability, and human
control**.
