**Goal:** Give Model Integrators (Airbus side) a single ontology “object family” that captures their work (tasks, validations, handoffs, risks) and automatically feeds business-owner views (status, KPIs, blockers, dates).

**Goal:** Give Model Integrators (Airbus side) a single ontology “object family” that captures their work (tasks, validations, handoffs, risks) and automatically feeds business-owner views (status, KPIs, blockers, dates).

---

## 1. What Business Owners Need to See (Competency Questions)

Design the object so you can answer, at any time:

- “Which partner/models are in integration right now and what stage are they in?”
    
- “What’s blocking deployment to MPVAL/Prod? Who owns the blocker?”
    
- “Which gates are passed/failed? Where’s the evidence?”
    
- “Are we on track vs. SLA dates? What are the key risks?”
    
- “What tools/environments were touched and by whom?”
    

---

## 2. Top-Level Object: **`IntegrationCase`**

Think of **IntegrationCase** as THE record of a model (or set of assets) moving through Airbus integration. One per model per environment (or per handoff wave—your call).

**Core properties (examples)**

- `caseId` (string, unique)
    
- `modelRef` (link → ModelWork/ModelVersion)
    
- `partnerRef` (link → Partner)
    
- `envTarget` (enum: MPVAL, PROD, etc.)
    
- `phase` (enum: Prep, Validation, Deployment, Post-Deployment)
    
- `status` (enum: NotStarted, InProgress, Blocked, Complete)
    
- `plannedStart`, `plannedEnd`, `actualStart`, `actualEnd` (dates)
    
- `slaDays` (int)
    
- `currentOwner` (link → Role/Person)
    
- `riskLevel` (enum: Low/Med/High)
    
- `lastUpdated` (timestamp)
    

**Key links**

- `hasTask` → IntegrationTask (1→many)
    
- `hasGate` → ApprovalGate (0→many)
    
- `hasValidation` → ValidationResult (0→many)
    
- `hasArtifact` → Artifact (0→many)
    
- `hasRisk` → RiskIssue (0→many)
    
- `usesTool` → Tool (many↔many)
    
- `touchesEnvironment` → Environment (many↔many)
    

---

## 3. Supporting Objects

|Object|Purpose|Key Props|Links|
|---|---|---|---|
|**IntegrationTask**|Atomic work item for the team|taskId, title, type, status, dueDate, owner|belongsTo → IntegrationCase; producesArtifact → Artifact; blockedBy → IntegrationTask|
|**ValidationResult**|Technical/functional validation outcome|resultId, type (perf, security), passFail|forTask → IntegrationTask; referencesMetrics → MetricSet; evidence → Artifact|
|**ApprovalGate**|Decision point (QG, internal board)|gateName, decision, decisionDate, approver|tiedTo → IntegrationCase; supportedBy → ValidationResult|
|**RiskIssue**|Risk, issue, or blocker tracking|riskId, category, severity, description, status, mitigation|raisedFor → IntegrationCase; owner → Role/Person|
|**Artifact**|Any evidence/document/script/output|artifactId, type (script, log, doc), uri|generatedBy → Task/ValidationResult|
|**KPI Snapshot** _(optional)_|Cached business KPIs per case|scheduleVariance, tasksDonePct, gatesPassed|summarizes → IntegrationCase|

> You can also reuse **Dataset, Environment, Tool, Role, Policy** objects from the broader ontology.

---

## 4. Relationship Pattern (text “diagram”)

```
Partner ──(delivers)──> ModelWork/ModelVersion
                           │
                           │ (integrated-via)
                           ▼
                     IntegrationCase ── hasTask ──> IntegrationTask
                           │   │                    └─ producesArtifact → Artifact
                           │   ├─ hasValidation ──> ValidationResult ── references → MetricSet
                           │   ├─ hasGate ────────> ApprovalGate ── supportedBy → ValidationResult
                           │   ├─ hasRisk ────────> RiskIssue
                           │   ├─ usesTool ───────> Tool
                           │   └─ touchesEnvironment → Environment
                           │
                           └─ currentOwner / RACI → Role/Person
```

---

## 5. Example Foundry Ontology Snippet (JSON-ish)

```json
{
  "objectTypes": [
    {
      "name": "da.IntegrationCase",
      "properties": [
        {"name": "caseId", "type": "string", "required": true},
        {"name": "phase", "type": "string", "enum": ["Prep", "Validation", "Deployment", "Post-Deployment"]},
        {"name": "status", "type": "string", "enum": ["NotStarted", "InProgress", "Blocked", "Complete"]},
        {"name": "envTarget", "type": "string", "enum": ["DEV", "MPVAL", "PROD"]},
        {"name": "plannedStart", "type": "date"},
        {"name": "plannedEnd", "type": "date"},
        {"name": "actualStart", "type": "date"},
        {"name": "actualEnd", "type": "date"},
        {"name": "slaDays", "type": "integer"},
        {"name": "riskLevel", "type": "string", "enum": ["Low", "Medium", "High"]},
        {"name": "lastUpdated", "type": "timestamp"}
      ],
      "links": [
        {"name": "modelRef", "type": "da.ModelVersion", "cardinality": "ONE"},
        {"name": "partnerRef", "type": "da.Partner", "cardinality": "ONE"},
        {"name": "currentOwner", "type": "da.Role", "cardinality": "ONE"},
        {"name": "hasTask", "type": "da.IntegrationTask", "cardinality": "MANY"},
        {"name": "hasValidation", "type": "da.ValidationResult", "cardinality": "MANY"},
        {"name": "hasGate", "type": "da.ApprovalGate", "cardinality": "MANY"},
        {"name": "hasRisk", "type": "da.RiskIssue", "cardinality": "MANY"},
        {"name": "hasArtifact", "type": "da.Artifact", "cardinality": "MANY"},
        {"name": "usesTool", "type": "da.Tool", "cardinality": "MANY"},
        {"name": "touchesEnvironment", "type": "da.Environment", "cardinality": "MANY"}
      ]
    },
    {
      "name": "da.IntegrationTask",
      "properties": [
        {"name": "taskId", "type": "string", "required": true},
        {"name": "title", "type": "string"},
        {"name": "type", "type": "string", "enum": ["Access", "Validation", "Deployment", "Documentation", "Other"]},
        {"name": "status", "type": "string", "enum": ["ToDo", "InProgress", "Blocked", "Done"]},
        {"name": "dueDate", "type": "date"},
        {"name": "owner", "type": "string"}
      ],
      "links": [
        {"name": "belongsTo", "type": "da.IntegrationCase", "cardinality": "ONE"},
        {"name": "producesArtifact", "type": "da.Artifact", "cardinality": "MANY"},
        {"name": "blockedBy", "type": "da.IntegrationTask", "cardinality": "MANY"}
      ]
    }
    /* ...define ValidationResult, RiskIssue, ApprovalGate, Artifact… */
  ]
}
```

(Replace `owner` string with link to `da.Role` or `da.Person` if you prefer linked objects.)

---

## 6. KPIs & Derived Fields

From these objects, compute business-friendly KPIs:

- **% Tasks Complete**: `doneTasks / totalTasks`
    
- **Schedule Variance (days)**: `actualEnd - plannedEnd`
    
- **Gate Pass Rate**: `approvedGates / totalGates`
    
- **Risk Heatmap**: count of open risks by severity
    
- **Time in Status**: transition logs on IntegrationCase.status
    

> Store these in a **KPI Snapshot object** or compute via Foundry transforms for dashboard performance.

---

## 7. Dashboards / Views for Business Owners

- **Case Overview Grid:** One row per IntegrationCase with status, phase, SLA burn-down, blockers.
    
- **Timeline / Gantt:** planned vs actual for each IntegrationCase.
    
- **Gate Progress Bar:** colored icons per gate (Passed, Pending, Blocked).
    
- **Risk & Issue Board:** severity, owners, mitigation due dates.
    
- **Tool/Environment Footprint:** which Airbus assets are touched (useful for impact/compliance reporting).
    

> **Visualization Opportunity:** Sankey (ModelVersion → Tasks → Gates), RACI Heatmap for integration-only steps, Risk bubble chart.

---

## 8. Governance & Automation Hooks

- **Validation Rules:**
    
    - Every IntegrationCase must link to a ModelVersion and Partner.
        
    - A case in “Deployment” must have at least one Passed ValidationResult.
        
    - Tasks past dueDate auto-flag `riskLevel=Medium` or create RiskIssue.
        
- **Actions / Write-Backs:**
    
    - “Close Case” action sets status=Complete and triggers artifact archiving.
        
    - “Escalate Risk” action notifies Business Owner if severity high for > X days.
        

---

## 9. Implementation Steps

1. **Confirm minimal property set** with integrators & owners.
    
2. Build Ontology Types in Foundry; link to existing datasets (Tickets, QG logs, MOCA reports).
    
3. Backfill historical data to test KPIs.
    
4. Create curated Views and Dashboards.
    
5. Iterate: add Automation & Validation once adoption starts.
    

---

**Next:** If you want, I can:

- Draft the remaining object definitions (ValidationResult, RiskIssue, ApprovalGate) in Foundry JSON/YAML.
    
- Provide example transforms to compute KPIs.
    
- Sketch the dashboard wireframes.
    

Below are two deliverables:

1. **Example Foundry transforms to compute KPIs** (PySpark + SQL patterns you can drop into Code Workbook or Pipeline Builder).
    
2. **Dashboard wireframes** for business owners (overview + drill-down pages).
    

---

## 1. KPI Transforms (Examples)

Assumptions:

- Ontology object sets already materialize to datasets like:
    
    - `da.IntegrationCase` → columns: `caseId`, `plannedStart`, `plannedEnd`, `actualStart`, `actualEnd`, `status`, `phase`, `envTarget`, `partnerRef`, `modelRef`, `lastUpdated`
        
    - `da.IntegrationTask` → `taskId`, `belongsTo` (caseId), `status`, `dueDate`, `owner`, `blockedBy`
        
    - `da.ApprovalGate` → `gateId`, `belongsTo` (caseId), `decision`, `decisionDate`
        
    - `da.RiskIssue` → `riskId`, `belongsTo` (caseId), `severity`, `status`, `openedOn`, `closedOn`
        
- Status enums: Tasks = {ToDo, InProgress, Blocked, Done}; Gates = {Approved/Rejected/Pending}; Risks = {Open/Closed}.
    
- Time zone assumptions handled upstream.
    

### 1.1 PySpark Transform (Code Workbook)

```python
# transform_kpis.py
from pyspark.sql import functions as F

cases = objectSet("da.IntegrationCase").read_dataframe()
tasks = objectSet("da.IntegrationTask").read_dataframe()
gates = objectSet("da.ApprovalGate").read_dataframe()
risks = objectSet("da.RiskIssue").read_dataframe()

# --- Task KPIs per case ---
task_agg = (tasks
    .groupBy("belongsTo")
    .agg(
        F.count("*").alias("tasks_total"),
        F.sum(F.when(F.col("status") == "Done", 1).otherwise(0)).alias("tasks_done"),
        F.sum(F.when(F.col("status") == "Blocked", 1).otherwise(0)).alias("tasks_blocked")
    )
    .withColumn("tasks_done_pct", F.when(F.col("tasks_total") > 0,
                                         F.col("tasks_done") / F.col("tasks_total")).otherwise(F.lit(0.0)))
)

# --- Gate KPIs ---
gate_agg = (gates
    .groupBy("belongsTo")
    .agg(
        F.count("*").alias("gates_total"),
        F.sum(F.when(F.col("decision") == "Approved", 1).otherwise(0)).alias("gates_passed")
    )
    .withColumn("gate_pass_rate",
                F.when(F.col("gates_total") > 0, F.col("gates_passed") / F.col("gates_total")).otherwise(F.lit(None)))
)

# --- Risk KPIs ---
risk_agg = (risks
    .groupBy("belongsTo")
    .agg(
        F.sum(F.when(F.col("status") == "Open", 1).otherwise(0)).alias("risks_open"),
        F.sum(F.when((F.col("status") == "Open") & (F.col("severity") == "High"), 1).otherwise(0)).alias("risks_open_high"),
        F.sum(F.when((F.col("status") == "Open") & (F.col("severity") == "Medium"), 1).otherwise(0)).alias("risks_open_med"),
        F.sum(F.when((F.col("status") == "Open") & (F.col("severity") == "Low"), 1).otherwise(0)).alias("risks_open_low")
    )
)

# --- Schedule Variance ---
cases_with_dates = (cases
    .withColumn("schedule_variance_days",
                F.datediff(F.col("actualEnd"), F.col("plannedEnd")))
    .withColumn("elapsed_days",
                F.datediff(F.coalesce(F.col("actualEnd"), F.current_date()), F.col("plannedStart")))
    .withColumn("planned_duration_days",
                F.datediff(F.col("plannedEnd"), F.col("plannedStart")))
)

# --- Combine all ---
kpis = (cases_with_dates.alias("c")
    .join(task_agg.alias("t"), F.col("c.caseId") == F.col("t.belongsTo"), "left")
    .join(gate_agg.alias("g"), F.col("c.caseId") == F.col("g.belongsTo"), "left")
    .join(risk_agg.alias("r"), F.col("c.caseId") == F.col("r.belongsTo"), "left")
    .select(
        "c.caseId", "c.partnerRef", "c.modelRef", "c.envTarget", "c.phase", "c.status",
        "c.plannedStart", "c.plannedEnd", "c.actualStart", "c.actualEnd",
        "schedule_variance_days", "elapsed_days", "planned_duration_days",
        "tasks_total", "tasks_done", "tasks_blocked", "tasks_done_pct",
        "gates_total", "gates_passed", "gate_pass_rate",
        "risks_open", "risks_open_high", "risks_open_med", "risks_open_low",
        "c.lastUpdated"
    )
)

# Write out as a KPI snapshot dataset
kpis.write.format("delta").mode("overwrite").saveAsTable("da_kpi.IntegrationCaseKPIs")
```

> Swap `.saveAsTable` for your Foundry dataset write API if you’re not on SparkSQL/Delta.

### 1.2 SQL Transform (Pipeline Builder)

```sql
-- transform_kpis.sql

WITH task_agg AS (
  SELECT belongsTo AS caseId,
         COUNT(*) AS tasks_total,
         SUM(CASE WHEN status = 'Done' THEN 1 ELSE 0 END) AS tasks_done,
         SUM(CASE WHEN status = 'Blocked' THEN 1 ELSE 0 END) AS tasks_blocked
  FROM da.IntegrationTask
  GROUP BY belongsTo
),
gate_agg AS (
  SELECT belongsTo AS caseId,
         COUNT(*) AS gates_total,
         SUM(CASE WHEN decision = 'Approved' THEN 1 ELSE 0 END) AS gates_passed
  FROM da.ApprovalGate
  GROUP BY belongsTo
),
risk_agg AS (
  SELECT belongsTo AS caseId,
         SUM(CASE WHEN status = 'Open' AND severity = 'High' THEN 1 ELSE 0 END) AS risks_open_high,
         SUM(CASE WHEN status = 'Open' AND severity = 'Medium' THEN 1 ELSE 0 END) AS risks_open_med,
         SUM(CASE WHEN status = 'Open' AND severity = 'Low' THEN 1 ELSE 0 END) AS risks_open_low,
         SUM(CASE WHEN status = 'Open' THEN 1 ELSE 0 END) AS risks_open
  FROM da.RiskIssue
  GROUP BY belongsTo
)
SELECT
  c.caseId,
  c.partnerRef,
  c.modelRef,
  c.envTarget,
  c.phase,
  c.status,
  c.plannedStart,
  c.plannedEnd,
  c.actualStart,
  c.actualEnd,
  DATEDIFF(c.actualEnd, c.plannedEnd) AS schedule_variance_days,
  DATEDIFF(COALESCE(c.actualEnd, CURRENT_DATE), c.plannedStart) AS elapsed_days,
  DATEDIFF(c.plannedEnd, c.plannedStart) AS planned_duration_days,
  t.tasks_total,
  t.tasks_done,
  t.tasks_blocked,
  CASE WHEN t.tasks_total > 0 THEN t.tasks_done / t.tasks_total ELSE 0 END AS tasks_done_pct,
  g.gates_total,
  g.gates_passed,
  CASE WHEN g.gates_total > 0 THEN g.gates_passed / g.gates_total ELSE NULL END AS gate_pass_rate,
  r.risks_open,
  r.risks_open_high,
  r.risks_open_med,
  r.risks_open_low,
  c.lastUpdated
FROM da.IntegrationCase c
LEFT JOIN task_agg t ON c.caseId = t.caseId
LEFT JOIN gate_agg g ON c.caseId = g.caseId
LEFT JOIN risk_agg r ON c.caseId = r.caseId;
```

> Configure this SQL file as a transform to produce a dataset like `da_kpi.IntegrationCaseKPIs`.

### 1.3 Optional: “Time in Status” transform

Track status changes with an events log (if you store history). Example PySpark:

```python
# Assuming a status history dataset: caseId, status, start_ts, end_ts
status_hist = spark.table("da.IntegrationCaseStatusHistory")

time_in_status = (status_hist
    .withColumn("duration_hours",
                (F.unix_timestamp("end_ts") - F.unix_timestamp("start_ts")) / 3600.0)
    .groupBy("caseId", "status")
    .agg(F.sum("duration_hours").alias("hours_in_status"))
)

time_in_status.write.format("delta").mode("overwrite").saveAsTable("da_kpi.TimeInStatus")
```

---

## 2. Dashboard Wireframes

### 2.1 Dashboard 1 – **Portfolio Overview (Business Owner Landing Page)**

```
┌───────────────────────────────────────────────────────────────────────────┐
│  Header: “Integration Portfolio Dashboard”    [Filters: Partner, Env, Phase, Status] │
├───────────────────────────────────────────────────────────────────────────┤
│ KPI Cards (row):                                                           │
│  • Active Cases  • % On-Time  • Avg Gate Pass Rate  • Open High Risks     │
├───────────────────────────────────────────────────────────────────────────┤
│ Portfolio Timeline (Gantt)                                                 │
│  - Horizontal bars: each IntegrationCase; colors by status/phase          │
│  - Milestone markers for Approval Gates                                   │
├───────────────────────────────────────────────────────────────────────────┤
│ Cases Grid (sortable table):                                               │
│  caseId | Partner | Model | Env | Phase | Status | TasksDone% | GatePass% │
│         | Risks(H/M/L) | PlannedEnd | Variance(days) | Owner   | Drilldown │
├───────────────────────────────────────────────────────────────────────────┤
│ Risk Heatmap (matrix or bubble): Severity vs. Case                         │
│ - Hover → mitigation plan, owner, due date                                 │
├───────────────────────────────────────────────────────────────────────────┤
│ Footer: data refresh timestamp, link to ontology, contact emails           │
└───────────────────────────────────────────────────────────────────────────┘
```

**Visualizations:**

- KPI tiles (cards)
    
- Gantt chart
    
- Table with conditional formatting
    
- Risk bubble chart or heatmap
    

### 2.2 Dashboard 2 – **IntegrationCase Detail View**

```
┌────────────────────────────── caseId: IC-2025-034  ───────────────────────┐
│ Header: Partner, Model, Env, Phase, Status, Owner, SLA Remaining          │
├───────────────────────────────────────────────────────────────────────────┤
│ Timeline strip: Planned vs Actual dates, milestone icons                  │
├───────────────────────────────────────────────────────────────────────────┤
│ Section A: Tasks                                                           │
│  - Kanban board (ToDo/InProgress/Blocked/Done) or progress bar             │
│  - Table: taskId, title, owner, dueDate, status                            │
│  - “Blocked by” chain visualization                                        │
├───────────────────────────────────────────────────────────────────────────┤
│ Section B: Gates & Validations                                             │
│  - Gate progress tracker (QG1→QG2→Prod) with pass/fail badges              │
│  - Validation reports list (download links, metrics summary)               │
├───────────────────────────────────────────────────────────────────────────┤
│ Section C: Risks & Issues                                                  │
│  - List with severity, mitigation plan, owner, dueDate                     │
│  - Add/Escalate buttons (if interactive)                                   │
├───────────────────────────────────────────────────────────────────────────┤
│ Section D: Artifacts & Evidence                                            │
│  - Table: artifactId, type, link/URI                                       │
│  - Data lineage mini-graph (optional)                                      │
└───────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Dashboard 3 – **Risk Board / Executive Heatmap**

```
┌───────────────────────────── Risk & SLA Monitor ──────────────────────────┐
│ Filters: Severity, Owner, Partner, Phase                                   │
├───────────────────────────────────────────────────────────────────────────┤
│ Heatmap: Rows = Cases | Columns = Severity Levels (Low/Med/High)          │
│   - Cell value = count of open risks                                       │
├───────────────────────────────────────────────────────────────────────────┤
│ SLA Breach Table:                                                          │
│ caseId | Partner | PlannedEnd | ActualEnd | Variance | Owner | Next Step  │
├───────────────────────────────────────────────────────────────────────────┤
│ Trend Lines:                                                              │
│ - %On-Time over last 6 months                                              │
│ - Avg Gate Pass Rate over time                                             │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## Next Steps

- Confirm KPI definitions & enums; adjust transform code accordingly.
    
- Decide which dashboards to build first (likely Overview + Case Detail).
    
- I can convert these wireframes into Figma/PlantUML/Mermaid or Foundry Workshop mockups if helpful.
    
- Let me know if you want the **transforms in YAML (Pipeline Builder)**, or example **Action** definitions for writebacks/escalations.
    

Ready to iterate when you are.