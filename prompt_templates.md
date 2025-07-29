Below are ready-to-drop **prompt templates** and **tool schemas** for your Partner Lifecycle / Integration Copilot in **Foundry AIP Agent Studio**. They’re organized so you can copy/paste and adapt quickly.

---

## 1. Agent Definition Snapshot

**Agent Name:** `PartnerLifecycleCopilot`  
**Primary Users:** Business Owners, Asset Integration Manager, Model Integrators  
**Core Capabilities:**

1. Answer lifecycle/status questions (Onboarding, Integration, Offboarding).
    
2. Surface KPIs, risks, gates, tasks.
    
3. Trigger ontology Actions (approve gate, close case, escalate risk).
    
4. Generate reports/summaries with citations to ontology objects.
    

---

## 2. Prompt Stack

### 2.1 System Prompt (Core Rules)

```text
You are the PartnerLifecycleCopilot operating inside Palantir Foundry.
Your mission is to help Airbus business owners and integrators manage partner onboarding, run/integration, and offboarding.

GROUND RULES
1. Always ground answers in the Foundry Ontology and KPI datasets. If data is missing, say so and propose the next step.
2. Cite object IDs or dataset row keys when making factual claims: e.g., (IntegrationCase: IC-2025-034).
3. Only change data by calling the provided Actions tools. Never “assume” a change happened.
4. Be concise but complete: give direct answers first, then offer drill-down or related insights.
5. Respect role-based access: if a tool returns “forbidden,” inform the user you lack permission.
6. Use safe language; do not provide export-controlled or restricted info unless user has access.
7. Prefer structured outputs (bullets, numbered lists, short tables) over long prose.

OUTPUT FORMAT
- Begin with the direct answer.
- Follow with “Details” or “Next Steps” if relevant.
- Include citations (object IDs, gate names, etc.) at the end of paragraphs.
```

---

### 2.2 Retrieval Prompt Template (for ontology/KPI queries)

Use this when forming a query tool call (planner message):

```text
User request: {{user_query}}

Goal: retrieve minimal but sufficient objects to answer.
Entities of interest: IntegrationCase, IntegrationTask, ApprovalGate, RiskIssue, KPI rows.

Write a structured query description:
- object types
- filters / ids
- properties needed
- joins/links needed

Return JSON for the tool call.
```

---

### 2.3 Answer Draft Template

```text
**Answer:** {{succinct_answer}}

**Details:**
- {{supporting_data_bullets}}

{{optional_table}}

**Next Steps:**
- {{suggested_actions}}

Citations: {{object_ids_and_dataset_refs}}
```

---

### 2.4 Action Invocation Template

```text
User intends to perform: {{action_name}}

Check preconditions:
- Is status/decision already final?
- Does user have permission?
- Are required inputs present?

If OK, call tool {{tool_name}} with payload:
{
  "targetObjectId": "...",
  "inputs": {...}
}

If missing info, ask clarifying question.
```

---

### 2.5 Error / Fallback Template

```text
I could not complete that because: {{error_message}}.
Here’s what we can do next:
1. {{option1}}
2. {{option2}}
```

---

### 2.6 Report Generation Template (LLM doc writer)

```text
Create a concise {{report_type}} for {{partner/model/case}}, covering:
- Current status & phase
- KPI snapshot (tasks_done_pct, gate_pass_rate, schedule_variance_days)
- Open risks (severity, mitigation owner)
- Upcoming milestones/gates (due dates)
- Required actions/decisions

Format: Markdown with headings.
Include IDs in parentheses for traceability.
```

---

## 3. Tool Schemas

> **Notation:** Adapt to AIP Logic’s JSON schema for tools (similar to OpenAI function calling). Replace dataset/objectSet names as per your Foundry installation.

### 3.1 `queryOntology`

**Purpose:** Query objects/links (generic ontology fetch).

```json
{
  "name": "queryOntology",
  "description": "Fetch ontology objects and their links from Foundry by type and filters.",
  "parameters": {
    "type": "object",
    "properties": {
      "objectType": { "type": "string", "description": "e.g. da.IntegrationCase" },
      "filters": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "property": { "type": "string" },
            "operator": { "type": "string", "enum": ["=", "in", "contains", ">", "<"] },
            "value": {}
          },
          "required": ["property", "operator", "value"]
        }
      },
      "properties": {
        "type": "array",
        "items": { "type": "string" },
        "description": "Fields to return"
      },
      "expandLinks": {
        "type": "array",
        "items": { "type": "string" },
        "description": "Link names to expand"
      },
      "limit": { "type": "integer", "default": 100 }
    },
    "required": ["objectType"]
  }
}
```

**Example Call:**

```json
{
  "name": "queryOntology",
  "arguments": {
    "objectType": "da.IntegrationCase",
    "filters": [
      { "property": "partnerRef", "operator": "=", "value": "da.Partner_GE" },
      { "property": "status", "operator": "in", "value": ["InProgress", "Blocked"] }
    ],
    "properties": ["caseId", "phase", "status", "plannedEnd", "modelRef"],
    "expandLinks": ["hasRisk", "hasGate"]
  }
}
```

---

### 3.2 `queryKPI`

**Purpose:** Pull precomputed KPIs from a curated dataset.

```json
{
  "name": "queryKPI",
  "description": "Read KPI rows for IntegrationCases or portfolio-level aggregates.",
  "parameters": {
    "type": "object",
    "properties": {
      "dataset": { "type": "string", "description": "e.g. da_kpi.IntegrationCaseKPIs" },
      "filters": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "column": { "type": "string" },
            "operator": { "type": "string", "enum": ["=", "in", ">", "<"] },
            "value": {}
          },
          "required": ["column", "operator", "value"]
        }
      },
      "columns": { "type": "array", "items": { "type": "string" } },
      "limit": { "type": "integer", "default": 200 }
    },
    "required": ["dataset"]
  }
}
```

---

### 3.3 `runAction`

**Purpose:** Execute ontology Actions (write-backs).

```json
{
  "name": "runAction",
  "description": "Invoke a Foundry ontology Action to update or create objects.",
  "parameters": {
    "type": "object",
    "properties": {
      "actionId": { "type": "string", "description": "e.g., da.CloseCase" },
      "targetObjectId": { "type": "string" },
      "inputs": { "type": "object", "additionalProperties": true }
    },
    "required": ["actionId", "targetObjectId"]
  }
}
```

**Example Call:**

```json
{
  "name": "runAction",
  "arguments": {
    "actionId": "da.ApproveGate",
    "targetObjectId": "da.ApprovalGate_QG2_2025_06_20",
    "inputs": {
      "decision": "Approved",
      "comment": "Performance metrics passed all thresholds"
    }
  }
}
```

---

### 3.4 `notifyChannel` (optional)

Send Slack/Email for escalations.

```json
{
  "name": "notifyChannel",
  "description": "Send a notification to Slack/Email group.",
  "parameters": {
    "type": "object",
    "properties": {
      "channel": { "type": "string", "description": "Slack channel or email group" },
      "subject": { "type": "string" },
      "message": { "type": "string" }
    },
    "required": ["channel", "message"]
  }
}
```

---

### 3.5 `generateDocument`

Use an LLM transform to create PDFs/Word/Markdown.

```json
{
  "name": "generateDocument",
  "description": "Generate a formatted report (markdown/pdf) based on supplied structured data and template.",
  "parameters": {
    "type": "object",
    "properties": {
      "templateId": { "type": "string" },
      "data": { "type": "object", "additionalProperties": true },
      "format": { "type": "string", "enum": ["markdown", "pdf", "docx"] }
    },
    "required": ["templateId", "data"]
  }
}
```

---

## 4. Few-Shot Examples

### Q1: Status of a Case

**User:** “What’s the status of integration case IC-2025-034 and are there any high risks?”

**Planner:**

- Call `queryOntology` on `da.IntegrationCase` filter by caseId.
    
- Expand `hasRisk`.
    
- If needed, call `queryKPI` for metrics.
    

**Answer Skeleton:**

```
Integration case IC-2025-034 is InProgress, phase=Validation, planned to end 2025-08-12.
Open High Risks: 2 (RiskIssue: R-883, R-901). Tasks done: 65%, Gate pass rate: 50%. (IntegrationCase: IC-2025-034)
Next steps: Resolve R-883 (network access), schedule QG2 by Aug 1.
```

---

### Q2: Approve a Gate

**User:** “Approve QG2 for case IC-2025-034.”

**Planner:**

- Check gate object: `queryOntology` ApprovalGate where belongsTo=case and gateName=QG2, decision pending.
    
- If found, call `runAction` → da.ApproveGate.
    

**If missing info:** “Can you confirm decision (Approved/Rejected) and comment?”

---

### Q3: Generate Monthly Portfolio Report

**User:** “Create a monthly status report for all partners in July.”

**Planner Steps:**

- `queryKPI` -> cases closed/open in July, aggregated metrics.
    
- `queryOntology` -> open risks, major milestones.
    
- `generateDocument` -> template “MonthlyPortfolioReport”.
    

---

## 5. Guardrails / Safety

- Always verify user intent before running actions (“Are you sure you want to close this case?”).
    
- Use Foundry RBAC to ensure the agent only sees permitted objects.
    
- Log all action calls (Foundry does this natively) for audit.
    

---

## 6. Implementation Checklist

1. **Register tools** in AIP Logic (or Agent Studio) using schemas above.
    
2. **Create prompt templates** in Agent Studio (system, planner, answer).
    
3. **Hook tools** to underlying datasets/ontology query endpoints.
    
4. **Test Few-Shots** (cover questions above + negative cases).
    
5. **Add Post-Processing** to ensure citations and formatting.
    
6. **Roll out** to pilot users; collect feedback; refine.
    

---

Need:

- A packaged JSON for Agent Studio import?
    
- Prompt tuning with your real object IDs / dataset names?
    
- AIP Logic code snippets for tool handlers?
    

Just say “send the import bundle” and I’ll structure it for direct drop-in.