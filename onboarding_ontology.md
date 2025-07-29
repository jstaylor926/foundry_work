Core objects (entity types)
Links (relationships)
Key properties



Below is a **proposed Foundry Ontology design** for the full partner lifecycle (Onboarding → Enablement/Run → Offboarding). I’ve “solidified” the core **objects (entity types)**, their **links (relationships)**, and key **properties**, using only what’s in scope for this project. Citations are included where lifecycle facts come directly from your manual.

---

## 1. Scope Check (what we’re modeling)

- Four lifecycle stages: **Onboarding, Enablement, Run Activity/Mode, Offboarding**.
    
- Provisioning steps (portal declaration, private space creation, authorization, access grant).
    
- Offboarding includes **IP blacklisting, portal removal, private space deletion**.
    

Everything else (training, certification, model dev, governance) hangs off those phases.

---

## 2. Ontology Layering Strategy

**Layer 1 – Process Ontology**  
Phases, Steps, Milestones, Roles, RACI, Decisions.

**Layer 2 – Technical Enablement Ontology**  
Spaces, Access Objects (IPs, groups), Tools, Datasets, Works/Models, Artifacts.

**Layer 3 – Compliance & Evidence**  
Policies, Approvals, Validation Reports, Checklists/Forms.

This separation keeps objects reusable and lets you drive docs, dashboards, and workflows from the same source.

---

## 3. Core Entity Types (Foundry “Object Types”)

| Entity Type                              | Purpose (short)                                 | Key Properties (examples)                                       | Key Links (→ Target)                                                                           |
| ---------------------------------------- | ----------------------------------------------- | --------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| **Partner**                              | External org in DA                              | legalName, entityId, status, contractRefs                       | undergoesPhase → Phase; delivers → Work; hasContact → Person                                   |
| **Phase** (enum-like entity)             | Onboarding / Enablement / Run / Offboarding     | name, startDate, endDate                                        | hasStep → Step; precedes → Phase                                                               |
| **Step**                                 | Atomic activity in a phase                      | stepCode, description, entryCriteria, exitCriteria, SLA, status | performedBy → Role; requiresTool → Tool; producesArtifact → Artifact; precedes → Step          |
| **Milestone**                            | Decision/approval point                         | name, dueDate, decision, outcome                                | belongsTo → Phase/Project; supportedBy → ValidationReport                                      |
| **Role**                                 | Business/tech role                              | name, org (Airbus/Partner), description                         | responsibleFor / accountableFor / consultedOn / informedAbout → Step (or via RACI link object) |
| **RACIAssignment** _(optional junction)_ | Explicit R/A/C/I mapping                        | responsibilityType (R/A/C/I)                                    | role → Role; step → Step                                                                       |
| **Project**                              | Aggregate of Works under one business objective | codeName, description, valueKPIs                                | includesWork → Work; ownedBy → BusinessOwner (Role)                                            |
| **Work**                                 | Deliverable set by a partner                    | workId, title, type (model, service), status                    | usesEnvironment → Environment; usesDataset → Dataset; producesArtifact → Artifact              |
| **Environment**                          | Foundry spaces etc.                             | envType (Private/Shared/Execution), namespace, orgId            | containsAccessObject → AccessObject; hostsWork → Work                                          |
| **AccessObject**                         | IP ranges, groups, permissions                  | type (IP, Group), value, validFrom/To                           | grantsAccessTo → Partner/User; belongsTo → Environment                                         |
| **Tool**                                 | DA/Foundry tools used in steps                  | name, purpose, url, dataHandled                                 | supportsStep → Step; usedIn → Experiment                                                       |
| **Dataset / FeatureSet**                 | Data inputs                                     | datasetId, sensitivityFlag, sourceSystem                        | governedBy → Policy; usedBy → Work/ModelVersion                                                |
| **ModelWork**                            | Model development “wrapper”                     | modelId, businessUseCase, status                                | hasVersion → ModelVersion; belongsTo → Work/Project                                            |
| **ModelVersion**                         | Versioned artifact                              | versionTag, createdOn, description                              | hasExperiment → Experiment; validatedBy → ValidationReport; deployedTo → DeploymentTarget      |
| **Experiment**                           | ML experiment run                               | experimentId, hyperparams, start/end                            | producedMetrics → MetricSet; producedArtifact → Artifact                                       |
| **MetricSet**                            | KPIs of an experiment                           | metrics (map), thresholdCompliance                              | ofExperiment → Experiment                                                                      |
| **ValidationReport**                     | Formal validation doc                           | reportId, validator, passFail, date                             | summarizesMetrics → MetricSet; requiredForGate → ApprovalGate                                  |
| **ApprovalGate**                         | QG / board decision                             | gateName, decision, decisionDate                                | referencesReport → ValidationReport; advancesVersion → ModelVersion                            |
| **DeploymentTarget**                     | MPVAL/PROD endpoints                            | env, endpointURL, deploymentDate                                | deploysVersion → ModelVersion                                                                  |
| **Policy/Control**                       | Data/security rule                              | policyId, type, description                                     | governs → Dataset/Work/Step                                                                    |
| **Artifact**                             | Any produced file/object                        | type (weights, doc), uri, checksum                              | generatedBy → Step/Experiment; supports → ValidationReport                                     |
| **ChecklistItem**                        | (Optional) tracklist items                      | text, mandatoryFlag, completed                                  | belongsTo → Step/Phase                                                                         |

> You can collapse some of these if Foundry object-count is a concern (e.g., merge MetricSet into ValidationReport), but the above is clean for automation.

---

## 4. Link Types (Relationships)

Define them explicitly in the ontology with cardinalities:

- **Partner _undergoesPhase_ Phase** (1→many)
    
- **Phase _hasStep_ Step** (1→many, ordered via step.order or explicit precedes)
    
- **Step _precedes_ Step** (many↔many, directed)
    
- **Step _performedBy_ Role** (many↔many)
    
- **RACIAssignment _role_ Role / _step_ Step** (many↔many)
    
- **Step _requiresTool_ Tool** (many↔many)
    
- **Step _producesArtifact_ Artifact** (many↔many)
    
- **Work _usesEnvironment_ Environment** (many↔many)
    
- **Environment _containsAccessObject_ AccessObject** (1→many)
    
- **AccessObject _grantsAccessTo_ Partner/User** (many↔many)
    
- **Work _usesDataset_ Dataset** (many↔many)
    
- **Dataset _governedBy_ Policy** (many↔many)
    
- **Work _hasModel_ ModelWork** (0→many)
    
- **ModelWork _hasVersion_ ModelVersion** (1→many)
    
- **ModelVersion _hasExperiment_ Experiment** (1→many)
    
- **Experiment _producedMetrics_ MetricSet** (1→1)
    
- **Experiment _producedArtifact_ Artifact** (many↔many)
    
- **ModelVersion _validatedBy_ ValidationReport** (0→many)
    
- **ValidationReport _requiredForGate_ ApprovalGate** (1→1)
    
- **ModelVersion _deployedTo_ DeploymentTarget** (0→many)
    
- **Milestone _belongsTo_ Phase/Project** (many↔1)
    
- **Milestone _supportedBy_ ValidationReport** (many↔many)
    

---

## 5. Property Patterns & Enums

- **status** fields: Draft / In Progress / Complete / Rejected.
    
- **responsibilityType** enum: R, A, C, I.
    
- **phase.name** enum: Onboarding, Enablement, Run, Offboarding. (Yes, Enablement & Run are distinct per manual.)
    
- **envType** enum: Private Space, Shared Space, Works Execution Space (per manual’s terminology).
    

---

## 6. Foundry Implementation Tips

1. **Start with a Spreadsheet/JSON seed** of entities & links; import into Ontology Builder.
    
2. Use **namespaces**: `da.Partner`, `da.Step`, etc., to avoid clashes.
    
3. Add **Actions/Writebacks** to change statuses (e.g., “ApproveGate”, “BlacklistIP”).
    
4. Build **Views**:
    
    - Process Dashboard (Phase → Steps → Status)
        
    - RACI Heatmap (pivot Role vs Step)
        
    - Tool Matrix (Step vs Tool)
        
5. **Validation Rules** (Foundry Code Workbook or SHACL-like checks):
    
    - Every Step must have at least one Role (Responsible).
        
    - Offboarding Steps must reference Access revocation actions.
        
    - Any Dataset with sensitivityFlag=true must link to a Policy.
        

---

## 7. Minimal vs. Full Model

If you need to ship fast:

- **Must-have**: Partner, Phase, Step, Role, Tool, Environment, AccessObject, Work, Dataset, Policy, Artifact.
    
- **Later**: ModelWork stack (ModelVersion, Experiment, Metrics), RACIAssignment as separate entity, ChecklistItem.
    

---

## 8. Next Actions

1. Confirm this object list & link map.
    
2. Decide which pieces live as **enum attributes vs. first-class entities**.
    
3. I can draft a **Foundry Ontology JSON export** or **YAML schema** next, plus PlantUML for auto-diagramming.
    
4. We’ll then map your existing datasets/tables to these objects and start populating.
    
