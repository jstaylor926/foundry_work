Standardize 
One place to see every partner, model, and integration case.
Automated dashboards and KPIs for business owners
Actionable workflows (approve gates, close risks, revoke access)
AIP powered copilots
Faster, safer partner activation and deactivation

**Goal:** Give Model Integrators (Airbus side) a single ontology “object family” that captures their work (tasks, validations, handoffs, risks) and automatically feeds business-owner views (status, KPIs, blockers, dates).

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
    
## 3. Supporting Objects

|Object|Purpose|Key Props|Links|
|---|---|---|---|
|**IntegrationTask**|Atomic work item for the team|taskId, title, type, status, dueDate, owner|belongsTo → IntegrationCase; producesArtifact → Artifact; blockedBy → IntegrationTask|
|**ValidationResult**|Technical/functional validation outcome|resultId, type (perf, security), passFail|forTask → IntegrationTask; referencesMetrics → MetricSet; evidence → Artifact|
|**ApprovalGate**|Decision point (QG, internal board)|gateName, decision, decisionDate, approver|tiedTo → IntegrationCase; supportedBy → ValidationResult|
|**RiskIssue**|Risk, issue, or blocker tracking|riskId, category, severity, description, status, mitigation|raisedFor → IntegrationCase; owner → Role/Person|
|**Artifact**|Any evidence/document/script/output|artifactId, type (script, log, doc), uri|generatedBy → Task/ValidationResult|
|**KPI Snapshot** _(optional)_|Cached business KPIs per case|scheduleVariance, tasksDonePct, gatesPassed|summarizes → IntegrationCase|

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

# Ontology: The Core Model

Partner
Phase
Step
Role / Person
Tool
Environment and AccessObject
IntegrationCase
IntegrationTask
ApprovalGate
ValidationResult
RiskIssue
Artifact
ModelWork / ModelVersion / Experiment / Dataset / MetricSet


### Key Relationships (Examples)

- `Partner undergoesPhase Phase`
    
- `Phase hasStep Step` (ordered by `precedes`)
    
- `Step performedBy Role` and `Step requiresTool Tool`
    
- `IntegrationCase hasTask IntegrationTask`
    
- `IntegrationCase hasRisk RiskIssue`, `hasGate ApprovalGate`, `hasValidation ValidationResult`
    
- `ModelWork hasVersion ModelVersion` → `ModelVersion hasExperiment Experiment` → `Experiment producedMetrics MetricSet`
    

## Foundry Architecture & Components

|Layer|Foundry Feature|What We Do|
|---|---|---|
|**Data/Model**|**Ontology Object Sets**|Define Partner, Phase, Step, IntegrationCase, etc., with properties & links.|
|**Compute/Logic**|Code Workbook, Pipeline Builder|KPI transforms, validation rules, event logging, automated actions.|
|**UI/Consumption**|Workshop Dashboards & Views|Business-owner landing pages, detail views, risk boards.|
|**Automation**|Ontology Actions, Scheduled Transforms|Close cases, approve gates, escalate risks, recompute KPIs.|
|**AIP**|Agent Studio, AIP Logic, LLM Transform|Chatbots, auto-report generation, natural-language queries, proactive alerts.|



## Workflow & Actions

We embed **Actions** directly in the ontology so users can update state without leaving the dashboard.

**Examples:**

- **CloseCase** – set status=Complete, actualEnd; triggers KPI refresh.
    
- **MarkTaskDone / MarkTaskBlocked** – task updates with timestamps.
    
- **ApproveGate** – record decision, date, comments; recalc KPIs.
    
- **EscalateRisk** – notify business owners via Slack/Email, log escalation time.
    
- **BlacklistIP / RevokeAccess** – write back to AccessObject and trigger downstream scripts.
    

Actions can have:

- **Visibility rules** (only show when appropriate).
    
- **Input forms** (date, comment).
    
- **Handlers** (Python/Logic to send notifications or execute tech steps).
    
- **Post-actions** (kick off transforms, alerts).
    


Once you’ve formalized your Ontology in Foundry, here’s how that drives the way your data is actually stored, surfaced, and used:

---

## 1. Ontology → Object Sets (Underlying Datasets)

* **Object Type ⇒ Object Set**
  Every Object Type you define (e.g. `IntegrationCase`, `IntegrationTask`, `IntegrationGate`, `RequestValidation`) becomes its own *Object Set*—a managed dataset under the covers.
* **Schema Enforcement**
  Foundry automatically enforces your property definitions (types, required‑ness, enums) on every record ingested into that Object Set.

---

## 2. Link Types ⇒ Link Tables

* **Link Type ⇒ Relationship Table**
  Each Link Type (e.g. `caseRef` on `IntegrationTask`, or `modelRef` on `RequestValidation`) materializes as an internal link table that preserves the “from → to” relationships you’ve defined.
* **Queryable Graph**
  Those link tables let you traverse your graph in both UI (Ontology Explorer) and code (e.g. PySpark or SQL) just like joins, but with the safety and discoverability of your ontology metadata.

---

## 3. Ingestion Pipelines Populate Object Sets

* When you run a code‑free or code‑based pipeline to ingest a CSV, a database extract, or even a streaming source, you map source fields to your Ontology properties.
* Behind the scenes, Foundry writes those records into the Object Set’s underlying columnar storage (Parquet, Delta, or similar), indexed for fast lookups and lineage tracking.

---

## 4. You Can Treat Object Sets as First‑Class Datasets

* **In Code**: In a Python or Spark workbook, you simply import the Object Set (e.g. `foundry.read("yourns.IntegrationTask")`) and work with it like any other DataFrame.
* **In Transforms**: When building downstream transforms or feature pipelines, you can reference `yourns.IntegrationCase` directly—Foundry will resolve it to the physical dataset.
* **In Dashboards**: BI tools and Foundry’s built‑in dashboarding will see each Object Set as a table you can visualize, filter, and aggregate.

---

## 5. Automatic Lineage & Governed Access

* **Lineage**: Every time you ingest or transform data into your Object Sets, Foundry records the full upstream lineage. You can trace exactly which raw file or transform step produced each `IntegrationGate` record.
* **Access Control**: You govern at the Ontology level—so you can say “only DataOps can write to `IntegrationCase`,” or “everyone can read `IntegrationTask`,” and Foundry enforces it on the underlying datasets.

---

## 6. Scaling & Evolution

* **New Object Sets on Demand**: Whenever you add another Object Type or Link Type, you immediately get a new dataset ready to ingest into—no manual table creation is needed.
* **Versioning**: You can version your Ontology (v1 → v2) and Foundry will manage migrations for you, ensuring backward compatibility or smooth schema evolution.

---

### In Practice

1. **You define** your Ontology schema in the Builder (object types, properties, links).
2. **Foundry auto‑creates** the backing datasets and relationship tables.
3. **You ingest** data via pipelines, which write into those datasets.
4. **You consume** those datasets everywhere—code, dashboards, lineage views—all governed by the Ontology’s rules.

This model means you never have to separately create or manage your tables: your Ontology *is* your data model, and Foundry takes care of spinning up, enforcing, and governing the actual datasets.
