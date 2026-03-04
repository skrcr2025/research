Comprehensive Migration Factory Design
PowerCenter On-Premise → API-Driven Airflow/Spark/Kubernetes Platform
Document Version: 1.0
Scope: Enterprise-scale migration of thousands of PowerCenter objects to a declarative, API-driven data platform

1. Executive Summary
The Challenge
This migration translates an established Informatica PowerCenter on-premise estate — spanning multiple domains, repositories, and thousands of interconnected objects — into declarative JSON artifacts that drive a modern, API-controlled platform running Apache Airflow DAGs over Apache Spark on Kubernetes. The challenge is not merely technical translation; it is doing so systematically, at scale, and with guaranteed data correctness across an organization's entire data processing portfolio.
The Factory Concept
A migration factory is an industrialized, repeatable conversion process. Rather than migrating objects one-by-one through bespoke manual effort, the factory treats migration as a production line: raw source objects enter one end, validated target artifacts exit the other, with automated tooling, quality gates, and a defined human review process at each station. This yields:

Speed: Automated conversion of high-volume, standard objects
Consistency: Deterministic transformation rules applied uniformly
Quality: Mandatory validation gates before deployment
Visibility: Full tracking, reporting, and auditability
Scalability: Parallel wave teams working simultaneously

Key Principles

Automate high-confidence conversions; gate low-confidence ones for human review
An Intermediate Representation (IR) decouples source parsing from target generation
Source and target run in parallel until full data validation is complete
No object is deployed to production without passing all quality gates
The factory itself is a software product — versioned, tested, and maintained

Expected Outcomes

Migration of thousands of PowerCenter objects with a target ≥65% automation rate on conversion effort
Zero undiscovered data discrepancies in production post-cutover (enforced by data reconciliation gates)
Full audit trail from source object to deployed artifact
Reusable factory infrastructure applicable to future platform migrations


2. Migration Strategy
2.1 Overall Approach: Phased Wave Migration
A phased, wave-based migration is selected over big-bang for these reasons:
FactorBig BangWave Migration (Selected)RiskExtremely highDistributed and controlledTooling maturityAssumed from day 1Refined iterativelyBusiness continuityDisruptedMaintained through parallel runRollbackFull rollback onlyPer-wave rollback possibleLearning curveNo opportunityIncorporated between wavesStakeholder confidenceLowBuilt progressively
The migration runs in parallel operation throughout: PowerCenter remains the system of record until each wave's objects are fully validated and signed off on the target platform.
2.2 Prioritization Criteria
Objects are prioritized for migration waves based on a weighted scoring model:
CopyPriority Score = (Business Criticality × 0.35) + (Automation Feasibility × 0.30)
               + (Dependency Readiness × 0.20) + (Risk Profile × 0.15)
Business Criticality Dimensions:

Revenue/compliance impact of the pipeline
Stakeholder sponsorship and engagement level
Frequency of execution (high-frequency = higher value demonstrated quickly)

Automation Feasibility (Complexity Tier, see Section 5.4):

Tier 1 objects first (maximize early factory throughput)
Tier 4 objects last (maximum manual effort, benefit from mature tooling)

Dependency Readiness:

Objects whose dependencies (connections, shared components) are already migrated score higher
Cross-domain dependency objects score lower until predecessors complete

Risk Profile:

Low-risk objects (non-critical, dev/test) migrate early to validate approach
High-risk critical production objects migrate mid-to-late when factory is proven

2.3 Object Complexity Tiers
The factory classifies every object using an automated Complexity Scoring Engine:
CopyComplexity Score = Σ(transformation_weights) + workflow_factor + parameterization_factor
                 + external_dependency_factor + reusability_factor
Transformation Weights:
Transformation TypeWeightFilter, Expression, Union, Sorter1Joiner, Aggregator, Router, Normalizer2Lookup (connected), Rank, Sequence Generator3Stored Procedure, XML Parser, Java4External Procedure, Custom, Unconnected Lookup5
Additional Factors:
FactorAdditional PointsDecision task present+3Each Worklet embedded+2Each Mapplet used+1Parameter files with 10+ params+3Pre/post session commands+2Cross-domain dependencies+4CDC/real-time pattern+5
Resulting Tiers:
TierScoreDescriptionAutomation RateApprox. % of Estate1 – Simple0–10Linear, standard transforms, simple connections~80%~35%2 – Moderate11–25Multi-source, lookups, branching, worklets~55%~35%3 – Complex26–50Custom logic, stored procs, complex orchestration~25%~20%4 – Very Complex51+Custom plug-ins, CDC, external procedures~5%~10%
2.4 Wave/Batch Planning
Copy┌────────────────────────────────────────────────────────────────────────┐
│  WAVE STRUCTURE                                                         │
├────────────┬──────────────────────────────────────────────────────────┤
│ Wave 0     │  Foundation: Infrastructure, tooling, templates, pilot    │
│ (Month 1-3)│  Scope: 10-20 objects (hand-picked simple objects)       │
├────────────┼──────────────────────────────────────────────────────────┤
│ Wave 1     │  Pilot Production: Tier 1 objects, non-critical domains  │
│ (Month 3-5)│  Scope: 100-200 Tier 1 objects                          │
├────────────┼──────────────────────────────────────────────────────────┤
│ Waves 2-N  │  Production Migration: Tier 1 bulk, then Tier 2          │
│ (Month 5+) │  Scope: 200-400 objects per wave, 3-4 week cadence       │
├────────────┼──────────────────────────────────────────────────────────┤
│ Final Waves│  Complex Objects: Tier 3 and Tier 4                      │
│ (Late phase)│  Scope: 50-100 objects per wave with heavy manual effort│
├────────────┼──────────────────────────────────────────────────────────┤
│ Cutover    │  Final validation, parallel run sign-off, decommission   │
└────────────┴──────────────────────────────────────────────────────────┘
Within each wave, the internal migration order is always:

Datastores (connection definitions) — no dependencies
Datasets (source/target definitions) — depend on datastores
Shared library artifacts (Mapplet-derived code modules, Worklet-derived sub-pipeline templates)
Pipeline definitions — depend on all of the above


3. Factory Architecture
3.1 High-Level Architecture
Copy╔══════════════════════════════════════════════════════════════════════════════╗
║                        MIGRATION FACTORY PLATFORM                           ║
╠══════════════╦═══════════════╦═══════════════╦═════════════════════════════╣
║  DISCOVERY   ║  CONVERSION   ║  VALIDATION   ║  DEPLOYMENT                 ║
║  ENGINE      ║  ENGINE       ║  ENGINE       ║  ENGINE                     ║
║              ║               ║               ║                             ║
║ ┌──────────┐ ║ ┌───────────┐ ║ ┌───────────┐ ║ ┌─────────────────────────┐ ║
║ │Repository│ ║ │Connection │ ║ │Schema     │ ║ │Control Plane API Client │ ║
║ │Scanner   │ ║ │Converter  │ ║ │Validator  │ ║ └─────────────────────────┘ ║
║ └──────────┘ ║ └───────────┘ ║ └───────────┘ ║                             ║
║ ┌──────────┐ ║ ┌───────────┐ ║ ┌───────────┐ ║ ┌─────────────────────────┐ ║
║ │Dependency│ ║ │Dataset    │ ║ │Logic      │ ║ │Git Version Controller   │ ║
║ │Mapper    │ ║ │Converter  │ ║ │Validator  │ ║ └─────────────────────────┘ ║
║ └──────────┘ ║ └───────────┘ ║ └───────────┘ ║                             ║
║ ┌──────────┐ ║ ┌───────────┐ ║ ┌───────────┐ ║ ┌─────────────────────────┐ ║
║ │Complexity│ ║ │Mapping/   │ ║ │Data       │ ║ │Deployment Orchestrator  │ ║
║ │Scorer    │ ║ │Pipeline   │ ║ │Reconciler │ ║ └─────────────────────────┘ ║
║ └──────────┘ ║ │Converter  │ ║ └───────────┘ ║                             ║
║ ┌──────────┐ ║ └───────────┘ ║ ┌───────────┐ ║ ┌─────────────────────────┐ ║
║ │Inventory │ ║ ┌───────────┐ ║ │Test       │ ║ │Documentation Generator  │ ║
║ │Manager   │ ║ │Expression │ ║ │Runner     │ ║ └─────────────────────────┘ ║
║ └──────────┘ ║ │Translator │ ║ └───────────┘ ║                             ║
╠══════════════╩═══════════════╩═══════════════╩═════════════════════════════╣
║                    SHARED PLATFORM INFRASTRUCTURE                           ║
║  ┌─────────────┐  ┌──────────────┐  ┌──────────┐  ┌──────────────────────┐ ║
║  │ Migration   │  │  Git/Source  │  │ CI/CD    │  │  Tracking Dashboard  │ ║
║  │ Catalog DB  │  │  Control     │  │ Pipeline │  │  (Grafana/Tableau)   │ ║
║  │ (PostgreSQL)│  │  (GitLab)    │  │ (GitLab  │  │                      │ ║
║  └─────────────┘  └──────────────┘  │  CI)     │  └──────────────────────┘ ║
║                                      └──────────┘                           ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  EXTERNAL INTEGRATIONS                                                       ║
║  ┌────────────────┐    ┌─────────────────────┐    ┌──────────────────────┐  ║
║  │  PowerCenter   │    │  Target Control      │    │  Test Data           │  ║
║  │  Repository    │◄──►│  Plane APIs          │    │  Environments        │  ║
║  │  (Source)      │    │  (Target)            │    │  (Source + Target)   │  ║
║  └────────────────┘    └─────────────────────┘    └──────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════════════╝
3.2 Core Components
3.2.1 Discovery Engine
Responsible for extracting all metadata from PowerCenter repositories and building a complete, queryable inventory.
Sub-components:

Repository Scanner: Connects to each PowerCenter repository via pmrep CLI, infacmd, or direct repository database queries. Exports all objects as XML and parses them into structured records.
Dependency Mapper: Constructs a directed acyclic graph (DAG) of all object dependencies — sessions depend on mappings and connections; workflows depend on sessions and worklets; mappings depend on source/target definitions. Uses NetworkX (Python) internally.
Complexity Scorer: Applies the scoring rubric (Section 2.3) to each workflow and mapping, assigns complexity tiers.
Inventory Manager: Persists all extracted metadata into the Migration Catalog Database with full lineage and status tracking.

3.2.2 Conversion Engine
The heart of the factory. Transforms parsed PowerCenter objects into valid target JSON artifacts through deterministic rules and templates.
Sub-components:

Connection Converter: Maps PowerCenter connection objects to Datastore JSON definitions.
Dataset Converter: Maps PowerCenter Source/Target definitions and Session-level connection bindings to Dataset JSON definitions.
Mapping/Pipeline Converter: Converts the transformation DAG within each PC mapping into the corresponding pipeline task transformation specification.
Workflow/Pipeline Converter: Converts the workflow orchestration graph (tasks, links, decisions, worklets) into the pipeline orchestration JSON structure.
Expression Translator: Converts PowerCenter expression language to Spark SQL expressions. This is the most technically complex sub-component (see Section 5.3).

The Intermediate Representation (IR):
The conversion engine works through an explicit Intermediate Representation layer. Raw PC XML is never directly transformed to target JSON. Instead:
CopyPowerCenter XML → [Parser] → Normalized IR → [Rule Engine] → Target JSON Artifacts
The IR is language-agnostic, stored in the Migration Catalog DB, and enables:

Validation at the semantic level (not just XML or JSON syntax)
Incremental processing (re-run only changed portions)
Manual editing of the IR when auto-conversion fails
Clear separation of concerns for tooling development

IR Structure (simplified):
jsonCopy{
  "ir_id": "uuid",
  "source_object_type": "pc_workflow",
  "source_domain": "domain_name",
  "source_repository": "repo_name",
  "source_folder": "folder_name",
  "source_object_name": "WF_CUSTOMER_LOAD",
  "complexity_tier": 2,
  "complexity_score": 18,
  "conversion_status": "auto_converted | needs_review | blocked | manual_complete",
  "target_artifacts": {
    "pipeline_id": "pipeline_customer_load",
    "referenced_datastores": ["ds_oracle_crm", "ds_oracle_dw"],
    "referenced_datasets": ["ds_src_customers", "ds_tgt_customer_dim"]
  },
  "issues": [
    {
      "issue_id": "uuid",
      "severity": "warning | error | info",
      "category": "unsupported_transformation | expression | connection",
      "description": "Unconnected lookup LKP_POSTAL_CODE requires manual broadcast join implementation",
      "resolution_status": "open | resolved | deferred",
      "resolution_notes": ""
    }
  ],
  "manual_review_required": true,
  "reviewer_assigned": "engineer_id",
  "review_completed_at": null
}
3.2.3 Validation Engine
Ensures every generated artifact meets quality standards before deployment.
Sub-components:

Schema Validator: Validates generated JSON against the target platform's JSON schema definitions.
Logic Validator: Checks that all referenced datastores and datasets exist and are deployed, transformation chains are complete, no orphaned references.
Data Reconciler: Executes migrated pipelines in a test environment and compares outputs between PowerCenter and the target platform.
Test Runner: Orchestrates unit and integration test suites against generated artifacts.

3.2.4 Deployment Engine
Manages the publication of validated artifacts to the target Control Plane.
Sub-components:

Control Plane API Client: Wrapper around the target platform's REST APIs for creating/updating datastores, datasets, and pipelines. Handles authentication, retries, rate limiting.
Git Version Controller: Commits all generated artifacts to a structured Git repository before deployment.
Deployment Orchestrator: Enforces deployment order (datastores first, then datasets, then pipelines), manages environment promotion (dev → staging → production).
Documentation Generator: Auto-generates pipeline documentation from the IR and deployed artifacts.

3.3 Migration Catalog Database Schema (Core Tables)
sqlCopy-- Master inventory of all source objects
migration_objects (
  object_id UUID PRIMARY KEY,
  domain VARCHAR, repository VARCHAR, folder VARCHAR,
  object_type VARCHAR,  -- workflow, mapping, session, connection, etc.
  object_name VARCHAR,
  complexity_tier INT,
  complexity_score INT,
  migration_status VARCHAR,  -- discovered, converting, review, validated, deployed, verified, closed
  wave_assignment INT,
  priority_score DECIMAL,
  created_at TIMESTAMP, updated_at TIMESTAMP
)

-- Dependency graph
object_dependencies (
  dependent_id UUID REFERENCES migration_objects,
  dependency_id UUID REFERENCES migration_objects,
  dependency_type VARCHAR  -- uses_connection, uses_mapping, calls_worklet, etc.
)

-- Generated artifacts
migration_artifacts (
  artifact_id UUID PRIMARY KEY,
  object_id UUID REFERENCES migration_objects,
  artifact_type VARCHAR,  -- datastore, dataset, pipeline
  artifact_name VARCHAR,
  git_commit_sha VARCHAR,
  target_environment VARCHAR,
  deployment_status VARCHAR,
  deployed_at TIMESTAMP,
  artifact_json JSONB
)

-- Issues and resolutions
migration_issues (
  issue_id UUID PRIMARY KEY,
  object_id UUID REFERENCES migration_objects,
  severity VARCHAR,
  category VARCHAR,
  description TEXT,
  resolution_status VARCHAR,
  resolution_notes TEXT,
  assigned_to VARCHAR,
  created_at TIMESTAMP, resolved_at TIMESTAMP
)

-- Test results
test_results (
  result_id UUID PRIMARY KEY,
  object_id UUID REFERENCES migration_objects,
  test_type VARCHAR,  -- schema_validation, logic_validation, data_reconciliation, uat
  test_status VARCHAR,  -- pass, fail, skipped
  details JSONB,
  executed_at TIMESTAMP, executed_by VARCHAR
)

-- Wave planning
waves (
  wave_id INT PRIMARY KEY,
  wave_name VARCHAR,
  planned_start DATE, planned_end DATE,
  actual_start DATE, actual_end DATE,
  wave_status VARCHAR,
  scope_count INT, completed_count INT
)

wave_assignments (
  wave_id INT REFERENCES waves,
  object_id UUID REFERENCES migration_objects,
  priority_within_wave INT
)
3.4 Git Repository Structure
Copymigration-factory/
├── tools/                          # Factory tooling (Python packages)
│   ├── discovery/                  # Repository scanning and inventory
│   │   ├── pc_extractor.py         # pmrep/infacmd wrapper
│   │   ├── xml_parser.py           # PowerCenter XML parser
│   │   ├── dependency_analyzer.py  # Dependency graph builder
│   │   └── complexity_scorer.py    # Complexity scoring engine
│   ├── converter/                  # Conversion engine
│   │   ├── connection_converter.py
│   │   ├── dataset_converter.py
│   │   ├── mapping_converter.py
│   │   ├── workflow_converter.py
│   │   └── expression_translator.py  # PC expression → Spark SQL
│   ├── validator/                  # Validation engine
│   │   ├── schema_validator.py
│   │   ├── logic_validator.py
│   │   └── data_reconciler.py
│   └── deployer/                   # Deployment engine
│       ├── control_plane_client.py
│       ├── deployment_orchestrator.py
│       └── doc_generator.py
├── schemas/                        # JSON schemas for target artifacts
│   ├── datastore_schema.json
│   ├── dataset_schema.json
│   └── pipeline_schema.json
├── templates/                      # Jinja2 templates for common patterns
│   ├── datastores/
│   │   ├── jdbc_oracle.json.j2
│   │   ├── jdbc_sqlserver.json.j2
│   │   ├── s3.json.j2
│   │   └── ...
│   ├── datasets/
│   │   ├── table_dataset.json.j2
│   │   ├── file_dataset.json.j2
│   │   └── ...
│   └── pipelines/
│       ├── linear_pipeline.json.j2
│       ├── branching_pipeline.json.j2
│       └── ...
├── rules/                          # Conversion rule definitions (YAML)
│   ├── transformation_rules.yaml   # PC transform → Spark mapping
│   ├── expression_rules.yaml       # Expression function mapping
│   ├── connection_type_rules.yaml  # Connection type mapping
│   └── workflow_task_rules.yaml    # Workflow task type mapping
├── artifacts/                      # Generated JSON artifacts (Git-versioned)
│   ├── dev/
│   │   ├── datastores/
│   │   ├── datasets/
│   │   └── pipelines/
│   ├── staging/
│   └── production/
├── tests/                          # Test suites
│   ├── unit/                       # Tool unit tests
│   ├── integration/                # End-to-end conversion tests
│   └── reconciliation/             # Data reconciliation test harness
├── ci/                             # CI/CD pipeline definitions
│   └── .gitlab-ci.yml
└── docs/
    ├── runbooks/                   # Operational runbooks
    ├── standards/                  # Migration standards guide
    ├── decisions/                  # Architecture decision records (ADRs)
    └── generated/                  # Auto-generated pipeline docs
3.5 Technology Stack
LayerTechnologyPurposeSource ExtractionPython + subprocess (pmrep/infacmd)PC repository extractionXML Parsinglxml, ElementTreeParse PowerCenter XML exportsDependency AnalysisnetworkxObject dependency graphIR StoragePostgreSQL + psycopg2/SQLAlchemyMigration catalogTemplate GenerationJinja2Parameterized JSON artifact generationSchema ValidationjsonschemaValidate generated JSONSQL/Expression Parsingsqlglot, sqlparseParse and transform SQL/expressionsControl Plane Integrationhttpx / requestsREST API callsVersion ControlGitLab / GitHubArtifact and code versioningCI/CDGitLab CIAutomated pipelineTest Data Validationgreat_expectationsData reconciliation frameworkDashboardGrafana + PostgreSQLMigration progress visibilityIssue TrackingJIRAIssue and resolution trackingCLI Toolingclick / typerFactory CLI toolsProgress ReportingrichConsole output formattingLoggingstructlogStructured loggingConfigurationpydanticSettings and model validation

4. Migration Process Workflows
4.1 Phase 1: Discovery and Assessment
Duration: 4-8 weeks (Wave 0 / Pre-Wave 1)
Copy┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 1: DISCOVERY & ASSESSMENT                                     │
│                                                                       │
│  INPUTS:            ACTIVITIES:              OUTPUTS:                │
│  • PC Domain/Repo   • Connect to repos       • Complete object       │
│    credentials      • Export all objects       inventory (catalog)   │
│  • Domain list      • Parse XML exports      • Dependency graph      │
│  • Folder scope     • Build dependency       • Complexity scores     │
│                       graph                 • Wave plan              │
│                     • Score complexity      • Migration readiness    │
│                     • Identify issues         assessment             │
│                     • Map dependencies      • Issue backlog          │
│                     • Plan waves            • Executive dashboard    │
└─────────────────────────────────────────────────────────────────────┘
Detailed Activities:
1.1 Repository Extraction
bashCopy# Example extraction using pmrep
pmrep connect -r <repository> -d <domain> -n <user> -x <password>
pmrep listobjects -o workflow -f <folder> > workflow_list.txt
pmrep exportobjects -o workflow -f <folder> -u workflow_list.txt \
  -m <output_file>.xml

Extract all domains, repositories, and folders systematically
Export: Connections, Source Definitions, Target Definitions, Transformations, Mapplets, Mappings, Sessions, Worklets, Workflows
Capture parameter file locations and content
Document environment configurations

1.2 XML Parsing and Normalization

Parse each exported XML file using the factory's XML Parser
Normalize into the Intermediate Representation schema
Persist all objects to the Migration Catalog DB
Validate completeness: cross-reference all object references

1.3 Dependency Graph Construction

Build a complete dependency DAG using NetworkX
Identify:

Circular dependencies (should not exist in PC, but validate)
Cross-folder dependencies
Cross-repository dependencies (high complexity flag)
Orphaned objects (source/target defs with no sessions)
Shared objects (mapplets/worklets used by multiple workflows)



1.4 Automated Assessment Reports:

Inventory Summary: Total counts by object type, domain, repository, folder
Complexity Distribution: % in each tier, estimated migration effort
Dependency Heat Map: Objects with most downstream dependents
Feature Usage Report: Count of each transformation type, unusual features, custom plug-ins
Risk Assessment: Tier 4 objects, cross-domain deps, custom code
Connection Inventory: All connection types and counts
Parameter Analysis: Parameter files, variable usage patterns
Estimated Migration Duration: By wave, by team size

Quality Gate 1 — Discovery Complete:

 All domains and repositories extracted
 Zero unresolved reference errors in dependency graph
 All objects assigned complexity tiers
 Wave plan drafted and reviewed
 Assessment report delivered to stakeholders


4.2 Phase 2: Conversion and Transformation
Duration: Ongoing — per wave, typically 2-3 weeks per batch of objects
Copy┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 2: CONVERSION & TRANSFORMATION                                │
│                                                                       │
│  INPUTS:            ACTIVITIES:              OUTPUTS:                │
│  • Normalized IR    • Auto-convert Tier 1   • Datastore JSON        │
│    (from catalog)   • Partial-convert        artifacts               │
│  • Transformation     Tier 2/3             • Dataset JSON           │
│    rules            • Flag Tier 4 for        artifacts               │
│  • JSON templates     manual               • Pipeline JSON           │
│  • Connection map   • Human review queue     artifacts               │
│  • Parameter map    • Issue resolution      • Conversion log         │
│                     • Expression            • Issue resolutions      │
│                       translation           • Review documentation   │
└─────────────────────────────────────────────────────────────────────┘
Detailed Activities:
2.1 Automated Conversion Pipeline
The conversion engine processes each object type in order:
CopyStep 1: Retrieve IR from catalog (filtered by wave, status=discovered)
Step 2: Route by complexity tier:
  Tier 1 → Full auto-conversion pipeline
  Tier 2 → Partial auto-conversion + issue queue
  Tier 3 → Template scaffolding + manual review queue
  Tier 4 → Issue queue with manual assignment
Step 3: For each convertible object:
  a) Select appropriate template
  b) Apply transformation rules
  c) Translate expressions
  d) Resolve parameter references
  e) Resolve connection/dataset references
  f) Render target JSON artifact
  g) Log any conversion warnings/errors
  h) Update catalog with conversion_status
Step 4: Commit generated artifacts to Git (dev branch)
Step 5: Generate conversion summary report
2.2 Human Review Process
Objects requiring review enter a structured queue:
CopyReview Queue → Assigned to Migration Engineer → Review Checklist:
  □ Verify expression translations are semantically equivalent
  □ Confirm connection/dataset bindings are correct
  □ Review orchestration logic (decisions, branching)
  □ Validate parameter/variable mappings
  □ Document any deviations or assumptions
  □ Mark review_status = complete or escalate
→ Peer Review (for Tier 3/4 by Senior Engineer)
→ Update IR with manual corrections
→ Regenerate target artifacts
→ Update catalog status
2.3 Issue Resolution Workflow
CopyIssue Detected → Categorized by severity:
  BLOCKER: Cannot proceed without resolution
    → Assigned to Architecture team
    → Decision documented in ADR
    → Resolution implemented in tool or manual artifact
  
  ERROR: Conversion incomplete, requires manual fix
    → Assigned to Migration Engineer
    → Fix applied to IR
    → Artifact regenerated
  
  WARNING: Converted with assumption, needs review
    → Added to review queue
    → Reviewer confirms or corrects assumption
Quality Gate 2 — Conversion Complete (per batch):

 All objects in batch have conversion_status = auto_converted or manual_complete
 Zero open BLOCKER issues
 All ERROR issues resolved
 All WARNING issues reviewed and accepted or corrected
 All generated JSON artifacts committed to Git (dev branch)
 Conversion summary reviewed by Technical Lead


4.3 Phase 3: Testing and Validation
Duration: 1-2 weeks per wave batch
Copy┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 3: TESTING & VALIDATION                                       │
│                                                                       │
│  INPUTS:            ACTIVITIES:              OUTPUTS:                │
│  • Committed        • JSON schema            • Schema validation     │
│    JSON artifacts     validation               report                │
│  • Test data        • Logic validation       • Logic validation      │
│    (dev environ.)   • Deploy to dev            report                │
│  • Source PC          environment            • DAG execution logs    │
│    test runs        • Execute pipelines      • Data reconciliation   │
│                     • Data reconciliation      report                │
│                     • Performance            • Performance           │
│                       baseline                 baseline report       │
│                     • Issue remediation      • Test sign-off         │
└─────────────────────────────────────────────────────────────────────┘
Detailed Activities:
3.1 Static Validation (Pre-Deployment)

Schema validation: Every JSON artifact validated against platform JSON schema
Reference validation: All datastore/dataset references resolvable
Cycle detection: No circular task dependencies in pipeline JSON
Parameter validation: All declared parameters have defaults or are supplied

3.2 Dynamic Validation (Post-Deployment to Dev)

Deploy to dev environment via Control Plane API
Verify Airflow DAG is generated correctly (DAG structure matches expected)
Verify DAG tasks map to correct Spark job configurations
Execute with synthetic/sanitized test data

3.3 Data Reconciliation Testing
This is the most critical validation: proving that migrated pipelines produce the same output as their PowerCenter equivalents.
CopyReconciliation Framework:

1. Run PowerCenter pipeline on test data → capture output dataset
2. Run target platform pipeline on same input → capture output dataset
3. Compare:
   a) Row count (must match exactly, or within defined tolerance)
   b) Column-level aggregates: SUM, COUNT, MAX, MIN, COUNT DISTINCT
      for all numeric/date columns
   c) Key column value comparison for sample rows (random 5% sample)
   d) NULL count comparison per column
   e) Duplicate detection

4. Tolerance Thresholds:
   - Row count: Exact match required (0% tolerance)
   - Aggregate values: ±0.001% for floating point
   - Date comparisons: Format-normalized, exact match required
   - Text comparisons: Trim-normalized, case-normalized

5. Reporting:
   - PASS: All checks within tolerance
   - PASS WITH NOTES: Minor known differences documented
   - FAIL: Investigation required
3.4 Performance Baseline
For each migrated pipeline:

Record PowerCenter execution time (from historical logs)
Record target platform execution time
Record data volume processed
Note: Target (Spark) may differ significantly; establish new baseline, flag regressions

Quality Gate 3 — Testing Complete (per batch):

 100% of artifacts pass schema validation
 100% of artifacts pass logic/reference validation
 100% of pipelines successfully deployed to dev environment
 100% of data reconciliation tests executed
 ≥95% of pipelines achieve PASS status on reconciliation
 All FAIL results investigated and root-caused
 Remaining FAILs escalated with documented plan
 Performance baselines documented


4.4 Phase 4: Deployment
Duration: 1 week per wave (staging + production deployment)
Copy┌─────────────────────────────────────────────────────────────────────┐
│  PHASE 4: DEPLOYMENT                                                 │
│                                                                       │
│  INPUTS:            ACTIVITIES:              OUTPUTS:                │
│  • Validated        • Promote artifacts      • Staging deployment    │
│    artifacts          to staging branch      • Staging test results  │
│  • QG3 sign-off     • Deploy to staging     • Production deployment  │
│  • Change request   • Staging validation     • Post-deploy verify    │
│                     • Change approval        • Deployment record     │
│                     • Deploy to production  • Stakeholder sign-off   │
│                     • Post-deploy checks    • Wave closure report    │
└─────────────────────────────────────────────────────────────────────┘
Deployment Pipeline (CI/CD):
CopyDeveloper pushes to feature branch
    │
    ▼
CI: Lint + Schema Validation (automated)
    │
    ▼
CI: Unit Tests for tool code (automated)
    │
    ▼
Merge Request → Peer Review
    │
    ▼
Merge to dev branch
    │
    ▼
CI: Deploy to DEV environment via Control Plane API (automated)
    │
    ▼
CI: Smoke test — DAG generated, pipeline executable (automated)
    │
    ▼
CI: Data reconciliation tests (automated)
    │
    ▼
Quality Gate 3 review (manual sign-off)
    │
    ▼
Merge to staging branch
    │
    ▼
CI: Deploy to STAGING environment (automated)
    │
    ▼
CI: Integration tests + reconciliation (automated)
    │
    ▼
UAT sign-off by Business Analyst (manual)
    │
    ▼
Change Request approved (manual)
    │
    ▼
Merge to main (production) branch
    │
    ▼
CI: Deploy to PRODUCTION environment (automated, with manual approval gate)
    │
    ▼
Production smoke test (automated)
    │
    ▼
Post-deploy verification (manual)
    │
    ▼
Wave Closure Report (manual)
Quality Gate 4 — Production Deployment:

 Staging validation complete and signed off
 UAT completed by business stakeholders
 Change request approved per change management process
 Production deployment successfully executed
 All deployed pipelines generating correct DAGs
 First production run completed successfully (or scheduled)
 Monitoring and alerting configured for new pipelines


4.5 Phase 5: Verification and Wave Closure
Duration: 1-2 weeks post-deployment, then ongoing during parallel run
Activities:

Monitor first N production runs of migrated pipelines
Compare outputs against PowerCenter executions (production data reconciliation)
Track and resolve post-deployment issues
Conduct wave retrospective
Feed lessons learned into next wave's tooling improvements
Upon full sign-off: update PowerCenter schedule to disabled (keep runnable for emergency)
Upon final wave closure: plan PowerCenter decommission


5. Technical Mapping and Transformation Rules
5.1 PowerCenter Object → Target Platform Object Mapping
CopyPowerCenter Object          Target Platform Object
────────────────────        ───────────────────────────────────────────
Connection Object       →   Datastore JSON
  (Relational, File,
   FTP, SAP, etc.)

Source Definition       →   Dataset JSON (input)
Target Definition       →   Dataset JSON (output)
Session (connections)   →   Dataset references within Pipeline task

Mapping               →   Pipeline task transformation specification
  (transformation DAG)     (Spark transformation logic)

Mapplet               →   Reusable Spark code module
  (reusable transforms)    (referenced via pipeline template)

Session               →   Pipeline Task (binding mapping + connections)

Worklet               →   Sub-pipeline / TaskGroup (referenced inline)

Workflow              →   Pipeline JSON (top-level orchestration)
  (orchestration graph)

Parameter File        →   Pipeline parameters + Airflow variables
                          + Secrets Manager entries

Folder                →   Namespace/project within target platform
5.2 PowerCenter Transformation Type → Spark Equivalent
PowerCenter TransformationSpark ImplementationAutomation LevelSource Qualifierspark.read.jdbc() / spark.read.format() with filter/SQL override appliedHighExpressionDataFrame.withColumn() / selectExpr()HighFilterDataFrame.filter() / .where()HighAggregatorDataFrame.groupBy().agg()HighJoinerDataFrame.join() with join type mappingHighRouterMultiple DataFrame.filter() branches, split and routeHighUnionDataFrame.union() / unionByName()HighSorterDataFrame.orderBy()HighRankWindow function: rank().over(Window.partitionBy().orderBy())HighSequence Generatormonotonically_increasing_id() or row_number().over(Window)MediumLookup (connected, cached)Broadcast join: DataFrame.join(broadcast(lookup_df))MediumLookup (connected, uncached)Regular join with lookup datasetMediumLookup (unconnected)Pre-staged broadcast join or PySpark UDFLowNormalizerDataFrame.explode() or stack() for multiple column normalizationMediumUpdate StrategySpark write mode + MERGE INTO / Delta Lake upsertMediumXML Source/Targetspark.read.format("xml") with schemaMediumStored ProcedureAirflow JdbcOperator / PythonOperator calling DB procLowCommand TaskAirflow BashOperator / PythonOperatorMediumCustom TransformationRewrite as PySpark UDF — manualVery LowExternal ProcedureCustom Spark operator or Airflow operator — manualVery LowJava TransformationPort Java logic to PySpark UDF — manualVery Low
Target Definition Write Mode Mapping:
PowerCenter Target Load TypeSpark Write ModeNotesNormal (Insert)mode("append")Standard appendTruncate-Insertmode("overwrite")Full table replaceUpdateDelta Lake MERGE or JDBC UPDATERequires key columnsDeleteDelta Lake MERGE with DELETE clauseRequires key columnsUpdate-else-Insert (Upsert)Delta Lake MERGE with INSERT+UPDATEMost complexBulkmode("overwrite") with bulk JDBC optionsPerformance tuning
5.3 Expression Language Conversion
The Expression Translator processes PowerCenter expressions through a multi-pass approach:
Pass 1: Tokenization and Parsing

Tokenize the PC expression using a custom lexer
Build an Abstract Syntax Tree (AST) for the expression
Identify all function calls, operators, port references, variable references

Pass 2: Function Mapping
Apply the function conversion rules:
PowerCenter FunctionSpark SQL / PySpark EquivalentNotesIIF(cond, true, false)CASE WHEN cond THEN true ELSE false ENDDECODE(val, s1,r1, s2,r2, default)CASE WHEN val=s1 THEN r1 WHEN val=s2 THEN r2 ELSE default ENDIN(val, v1, v2, ...)val IN (v1, v2, ...)ISNULL(val)val IS NULLNVL(val, default)COALESCE(val, default)NVL2(val, not_null_res, null_res)CASE WHEN val IS NOT NULL THEN not_null_res ELSE null_res ENDTO_DATE(str, fmt)to_date(str, fmt)Format string conversion neededTO_CHAR(date, fmt)date_format(date, fmt)Format string conversion neededSYSDATEcurrent_timestamp()TRUNC(num)floor(num)TRUNC(date, fmt)date_trunc(fmt, date)ADD_TO_DATE(date, 'DD', n)date_add(date, n)Unit-dependent mappingDATEDIFF(date1, date2, 'DD')datediff(date1, date2)SUBSTR(str, start, len)substring(str, start, len)1-based indexing in bothINSTR(str, search)instr(str, search)LENGTH(str)length(str)LTRIM(str)ltrim(str)RTRIM(str)rtrim(str)UPPER(str)upper(str)LOWER(str)lower(str)CONCAT(s1, s2)concat(s1, s2)LPAD(str, len, pad)lpad(str, len, pad)RPAD(str, len, pad)rpad(str, len, pad)ROUND(num, scale)round(num, scale)CEIL(num)ceil(num)FLOOR(num)floor(num)ABS(num)abs(num)MOD(num, divisor)mod(num, divisor)POWER(base, exp)power(base, exp)SQRT(num)sqrt(num)REG_EXTRACT(str, pattern, idx)regexp_extract(str, pattern, idx)REG_REPLACE(str, pattern, repl)regexp_replace(str, pattern, repl)ERROR(msg)Custom raise_error() UDFABORT(msg)Custom raise_abort() UDFSetVariable(var, val)XCom push (Airflow) or pipeline output variableComplexGetVariable(var)XCom pull (Airflow) or pipeline input variableComplex
Pass 3: Date Format Conversion
PowerCenter uses a mix of Oracle-style and custom date format strings. These require a dedicated conversion map:
PC FormatSpark FormatMM/DD/YYYYMM/dd/yyyyDD-MON-YYYYdd-MMM-yyyyYYYY-MM-DD HH24:MI:SSyyyy-MM-dd HH:mm:ssMON → MMM (3-letter month)Apply substitution
Pass 4: Port Reference Resolution

PC expressions reference input/output ports by name
These map to DataFrame column names in Spark
Column name sanitization: remove spaces, special characters
Resolve port aliases and renamed columns through the transformation chain

Pass 5: Variable Reference Resolution

$$ParameterName → pipeline parameter reference
$WorkflowVariable → Airflow XCom or pipeline state variable
$MappingVariable → Spark accumulator or output variable (requires special handling)

Unresolvable Expressions:

Custom user-defined functions (PC-side): Flagged for manual UDF implementation in PySpark
Cross-mapping lookups: Flagged for broadcast join implementation
GetDate() variants with complex timezone logic: Flagged for review

5.4 Workflow Orchestration Mapping
PowerCenter Workflow TaskAirflow / Pipeline EquivalentAutomation LevelStart TaskDAG start (implicit)HighEnd TaskDAG end (implicit)HighSession TaskSpark pipeline taskHighWorkletTaskGroup or called sub-pipelineMediumDecision TaskBranchPythonOperator with condition expressionMediumCommand TaskBashOperator or PythonOperatorMediumTimer TaskAirflow scheduling / TimeDeltaSensorMediumEvent Wait TaskCustom Sensor operatorLowEvent Raise TaskTrigger mechanism / XCom signalLowEmail TaskEmailOperatorHighAssignment TaskPythonOperator setting XCom variableMedium
Link Condition Mapping:
PC Link ConditionAirflow EquivalentSuccess (default)TriggerRule.ALL_SUCCESSFailureTriggerRule.ALL_FAILEDSkipTriggerRule.ALL_SKIPPEDTop of LoopLoop construct in DAGConditional expressionBranchPythonOperator
5.5 Connection Type → Datastore JSON Mapping
Relational Connections:
jsonCopy{
  "datastore": {
    "name": "oracle_production_crm",
    "type": "jdbc",
    "description": "Oracle CRM Production Database",
    "connection": {
      "driver_class": "oracle.jdbc.OracleDriver",
      "url": "jdbc:oracle:thin:@${ORACLE_CRM_HOST}:${ORACLE_CRM_PORT}:${ORACLE_CRM_SID}",
      "username": "${secret:oracle_crm_user}",
      "password": "${secret:oracle_crm_password}",
      "fetch_size": 10000,
      "num_partitions": 8
    },
    "tags": ["source", "crm", "oracle"]
  }
}
Connection Type Mapping Table:
PowerCenter Connection TypeDatastore TypeNotesOracleJDBC (oracle driver)SQL ServerJDBC (mssql driver)DB2JDBC (db2 driver)TeradataJDBC (teradata driver)MySQLJDBC (mysql driver)PostgreSQLJDBC (postgresql driver)Flat File (local)Local filesystem datasetLikely needs redesign for KubernetesFlat File (NFS/SFTP)SFTP datastore or cloud storageAmazon S3S3 datastoreHDFSHDFS datastoreFTPCustom FTP datastore or pre-stage to cloudMay need workaroundSAPCustom SAP connector or JDBC via BAPIManual effortSalesforceCustom REST connectorManual effortMQ/JMSCustom messaging connectorMay be out of scopeWeb ServicesHTTP datastore or Airflow HTTP operator
5.6 Handling Unsupported Features
FeatureCategoryRecommended HandlingExternal Procedure TransformationNo direct equivalentImplement as custom Airflow operator or PythonOperator calling external APICustom (C/C++) TransformationNo direct equivalentRewrite business logic in PySpark; validate equivalenceJava TransformationNo direct equivalentPort Java code to PySpark UDFPowerCenter Data Quality pluginPlatform-specificReplace with target platform's DQ capability or custom Spark DQ rulesReal-time / JMS sourceStreaming patternSeparate streaming architecture required (Spark Structured Streaming or Kafka) — out of scope for batch migrationPersistent lookup cacheNo direct equivalentImplement as cached broadcast DataFrame with cache invalidation strategySession recovery / restartNo direct equivalentImplement checkpoint/restart pattern in pipeline JSON; use Airflow task retryReject file handlingNo direct equivalentImplement Spark bad record handling (PERMISSIVE mode + corrupt record column), write rejects to designated datasetFTP file pre/post processingNo direct equivalentAirflow SSHOperator or custom FTP operator as pre/post taskWorkflow variable increment loopsComplexImplement as parameterized Airflow DAG with dynamic task mappingSAP connectionsComplexRequires SAP connector library evaluation on Kubernetes; may require PyRFC or BAPI calls

6. Automation and Tooling
6.1 Automation Strategy
The factory's automation strategy is based on the "automate the repeatable, review the ambiguous, manually implement the unique" principle.
Automation Tier Framework:
CopyAUTOMATE (No human intervention):
  ✓ All Tier 1 connection → datastore conversions
  ✓ All simple source/target → dataset conversions
  ✓ Tier 1 workflow → pipeline JSON generation
  ✓ JSON schema validation
  ✓ Control Plane API deployment (with approval gates)
  ✓ Reconciliation test execution and reporting
  ✓ Git commits, CI/CD pipeline execution
  ✓ Documentation generation

ASSIST (Human reviews, tool does heavy lifting):
  ✓ Tier 2 pipeline conversion (tool generates scaffold, human fills gaps)
  ✓ Expression translation with warnings (tool converts, human validates)
  ✓ Decision task logic (tool generates template, human validates condition)
  ✓ Worklet/mapplet conversion (tool generates module structure, human verifies)

MANUAL (Tool creates issue ticket with context; human implements):
  ✗ Tier 4 custom transformations
  ✗ External procedures
  ✗ Complex parameterization with dynamic SQL
  ✗ Cross-domain dependency resolution
  ✗ CDC/streaming patterns
6.2 Factory CLI Tool
The factory is accessed through a unified CLI:
bashCopy# Discovery commands
factory discover --domain DOMAIN1 --all-repos --output catalog
factory assess --wave 1 --report html
factory plan --wave 1 --priority-model standard

# Conversion commands
factory convert --wave 1 --tier 1 --dry-run
factory convert --wave 1 --tier 1 --execute
factory review list --status needs_review --wave 1
factory review complete --object-id <uuid> --reviewer <id>

# Validation commands
factory validate schema --wave 1
factory validate logic --wave 1
factory test reconcile --wave 1 --environment dev

# Deployment commands
factory deploy --wave 1 --environment dev
factory deploy --wave 1 --environment staging
factory deploy --wave 1 --environment production --require-approval

# Reporting commands
factory report status --wave 1
factory report progress --all-waves
factory report issues --severity error --open

# Management commands
factory wave create --name "Wave_2" --start-date 2025-02-01
factory wave assign --wave 2 --tier 2 --domain DOMAIN1
factory issue list --wave 1 --open
factory issue resolve --issue-id <uuid> --notes "..."
6.3 Expression Translator Tool
The Expression Translator deserves special attention as it is the most technically sophisticated component:
pythonCopy# Conceptual interface
from factory.converter.expression_translator import ExpressionTranslator

translator = ExpressionTranslator(
    function_rules="rules/expression_rules.yaml",
    date_format_map="rules/date_format_rules.yaml",
    port_registry=port_registry,  # maps port names to column names
    variable_registry=variable_registry  # maps PC variables to pipeline params
)

result = translator.translate(
    expression="IIF(ISNULL(customer_id), NVL(alt_id, -1), customer_id)",
    context=transformation_context
)

# result.spark_expression = "CASE WHEN customer_id IS NULL THEN COALESCE(alt_id, -1) ELSE customer_id END"
# result.confidence = 0.98  # confidence score (1.0 = fully automated)
# result.warnings = []      # any warnings about the translation
# result.requires_review = False
Translation Confidence Scoring:

1.0: Fully deterministic translation, all functions known
0.8-0.99: Minor assumptions made, recommend review
0.5-0.79: Significant assumptions, require review
<0.5: Cannot translate reliably, flag for manual implementation

6.4 CI/CD Pipeline Design
yamlCopy# .gitlab-ci.yml (conceptual)
stages:
  - validate
  - test
  - deploy-dev
  - test-dev
  - deploy-staging
  - test-staging
  - approve-production
  - deploy-production
  - verify-production

validate-schema:
  stage: validate
  script:
    - factory validate schema --wave $WAVE --environment $CI_ENVIRONMENT
  rules:
    - if: $CI_COMMIT_BRANCH == "dev"

unit-tests:
  stage: test
  script:
    - pytest tests/unit/ -v --junit-xml=report.xml
  artifacts:
    reports:
      junit: report.xml

deploy-dev:
  stage: deploy-dev
  script:
    - factory deploy --wave $WAVE --environment dev
  environment: development
  needs: ["validate-schema", "unit-tests"]

reconcile-dev:
  stage: test-dev
  script:
    - factory test reconcile --wave $WAVE --environment dev --threshold 0.95
  needs: ["deploy-dev"]

deploy-staging:
  stage: deploy-staging
  script:
    - factory deploy --wave $WAVE --environment staging
  when: manual
  environment: staging
  needs: ["reconcile-dev"]

uat-staging:
  stage: test-staging
  script:
    - factory test reconcile --wave $WAVE --environment staging --threshold 1.0
  needs: ["deploy-staging"]

approve-prod:
  stage: approve-production
  when: manual
  allow_failure: false
  script:
    - echo "Production deployment approved"
  needs: ["uat-staging"]

deploy-prod:
  stage: deploy-production
  script:
    - factory deploy --wave $WAVE --environment production
  environment: production
  needs: ["approve-prod"]

verify-prod:
  stage: verify-production
  script:
    - factory test smoke --wave $WAVE --environment production
  needs: ["deploy-prod"]
6.5 Reusable Component Library
For Mapplets (reusable transformation groups):

Each unique Mapplet is converted once into a reusable PySpark module
Stored in a shared library within the factory Git repository
Versioned and referenced by pipeline configurations
Tested independently before being used in pipelines

For Worklets (reusable workflow subgraphs):

Each Worklet is converted into a pipeline template
Instantiated inline within parent pipelines
Or published as a standalone sub-pipeline and called by reference


7. Quality Assurance and Testing Strategy
7.1 Quality Gates Summary
CopyQG1: Discovery Complete
  └─ Inventory 100% complete, dependency graph validated, waves planned

QG2: Conversion Complete (per batch)
  └─ All objects converted, zero blockers, all reviews complete

QG3: Technical Validation Complete (per batch)
  └─ Schema valid, logic valid, dev deployment successful,
     ≥95% data reconciliation pass rate

QG4: Staging Validation Complete
  └─ Staging deployment successful, UAT passed, business sign-off

QG5: Production Deployment Complete
  └─ Production deployment successful, first run successful, monitoring active

QG6: Wave Closure
  └─ Production parallel run validated, business sign-off,
     PowerCenter schedules disabled for wave objects
7.2 Testing Types
Tier 1: Static Testing (automated, pre-deployment)

JSON Schema Validation: Every artifact validated against platform schema
Reference Integrity Check: All cross-references resolvable
Naming Convention Check: Object names follow standards
Parameter Completeness: All required parameters defined

Tier 2: Structural Testing (automated, post-deployment to dev)

DAG Generation: Confirm Airflow DAG is generated correctly
DAG Structure: Task count, dependencies match expected from conversion
Spark Job Configuration: Verify job parameters, resource allocation
Pipeline Execution Dry-Run: Execute with empty input (structural validation)

Tier 3: Functional Testing (semi-automated)

Data Transformation Logic: Execute with test data, validate transformation rules
Expression Validation: Unit-test complex expressions with known input/output pairs
Decision Logic: Test branching conditions with positive and negative test cases
Error Handling: Test with intentionally bad data; verify error behavior

Tier 4: Data Reconciliation Testing (automated execution, manual review)

Row count matching
Aggregate validation
Sample record comparison
NULL distribution comparison
See Section 4.3 for framework details

Tier 5: User Acceptance Testing (manual, business stakeholders)

Business validation of migrated pipeline outputs
Comparison against expected business results
End-user sign-off per wave

Tier 6: Performance Testing (manual, on representative datasets)

Execution time comparison (PC vs target)
Resource utilization on Kubernetes
Scalability under production data volumes
Identify performance regressions requiring optimization

7.3 Data Reconciliation Framework
Copy┌─────────────────────────────────────────────────────────────────┐
│  DATA RECONCILIATION FRAMEWORK                                   │
│                                                                   │
│  Source PC Run:                Target Platform Run:             │
│  ┌──────────┐                  ┌──────────────────────┐         │
│  │PowerCenter│  ─── same ───►  │ Target Platform       │        │
│  │Pipeline  │     input data   │ Pipeline (Spark)      │        │
│  └────┬─────┘                  └───────────┬──────────┘         │
│       │                                     │                    │
│       ▼                                     ▼                    │
│  PC Output Dataset              Target Output Dataset           │
│       │                                     │                    │
│       └──────────────────┬──────────────────┘                   │
│                           ▼                                      │
│                  ┌─────────────────┐                            │
│                  │ Reconciliation  │                            │
│                  │ Engine          │                            │
│                  │                 │                            │
│                  │ 1. Row counts   │                            │
│                  │ 2. Aggregates   │                            │
│                  │ 3. Sample rows  │                            │
│                  │ 4. NULL counts  │                            │
│                  │ 5. Key columns  │                            │
│                  └────────┬────────┘                            │
│                           ▼                                      │
│                  Reconciliation Report                           │
│                  PASS / FAIL / PASS WITH NOTES                  │
└─────────────────────────────────────────────────────────────────┘
Reconciliation Test Data Strategy:

Use sanitized/anonymized copies of production data for sensitive pipelines
Use production data (read-only) for non-sensitive pipelines in staging
Volume: minimum 30 days of production-representative data
Edge cases: include known edge-case data sets (NULLs, special characters, boundary values)

7.4 Acceptance Criteria
CriterionThresholdEscalation PathSchema Validation Pass Rate100%Blocker — fix before proceedingLogic Validation Pass Rate100%Blocker — fix before proceedingDev Deployment Success Rate100%Blocker — fix before proceedingData Reconciliation Pass Rate≥95%Remaining 5% individually assessedReconciliation: Row Count Match100%Individual investigation requiredReconciliation: Aggregate MatchWithin ±0.001%Document known differencesUAT Sign-Off100% of in-scope objectsEscalate to business sponsorProduction First-Run Success≥98%Immediate incident responsePerformance vs. BaselineWithin 150% of PC time or betterOptimization sprint

8. Risk Management
8.1 Risk Register
Risk IDRisk DescriptionLikelihoodImpactRisk LevelMitigation StrategyR01Expression translation produces semantically incorrect results for complex expressionsMediumHighCriticalExhaustive expression unit tests; confidence scoring; mandatory human review for score <0.8R02PowerCenter custom transformations have no equivalent on target platformHighHighCriticalInventory all custom transforms in Wave 0; design rewrite strategy per transform type; involve platform architectsR03Undocumented business logic embedded in PowerCenter SQL overridesMediumHighHighInclude SQL override extraction in discovery; flag for business analyst reviewR04Cross-domain dependencies not fully resolved before migrationMediumHighHighMandatory dependency graph validation; enforce migration order in wave planningR05Performance of migrated Spark pipelines significantly worse than PowerCenterHighMediumHighPerformance testing in Wave 1 pilot; Spark tuning guidelines; performance gate in QAR06Data reconciliation failures reveal logic differences in migrated pipelinesMediumHighHighMandatory reconciliation before production; parallel run period; root-cause all failuresR07Target platform API changes break factory deployment toolingLowHighMediumAPI version pinning; monitor platform release notes; factory tool regression testsR08Key personnel (PC SMEs) unavailable during migrationMediumHighHighKnowledge documentation in Wave 0; cross-training; document all PC-specific decisionsR09Scale of automation tooling insufficient for thousands of objectsLowHighMediumLoad-test factory tools in Wave 0/1; design for parallel processing from startR10Business stakeholders unavailable for UAT sign-offHighMediumHighBuild UAT into project plan; automated reconciliation reduces UAT burden; formal sign-off SLAsR11PowerCenter parameter files contain environment-specific secretsHighMediumMediumSecrets management strategy defined in Wave 0; use platform secrets manager from day 1R12Some PowerCenter features are out of scope for target platformMediumMediumMediumFeature gap analysis in Wave 0; formal decision for each gap; stakeholder alignmentR13Regression in production after migration (not caught in testing)LowVery HighHighMandatory parallel run period; production reconciliation; gradual traffic cutoverR14Factory tooling bugs produce systematically incorrect artifacts across many objectsLowVery HighCriticalUnit test the factory tools themselves; peer review of tool code; manual spot-check of auto-converted artifacts
8.2 Mitigation Strategies in Detail
R01 / R14 — Tool Quality:
The factory tools themselves are treated as production software:

All conversion logic has unit tests with known input/output pairs
Expression translator has a dedicated test suite of 200+ common PC expressions
Tool changes go through code review and CI/CD before deployment
10% random sample of auto-converted Tier 1 objects manually verified each wave

R05 — Performance:

Wave 0 pilot includes 5 Tier 1 and 3 Tier 2 pipelines, measuring Spark performance baseline
Spark tuning runbook developed and maintained
Performance SLA defined before production migration: target ≤150% of PowerCenter execution time or equal/better data SLA satisfaction
Object-level performance monitoring post-deployment

R13 — Production Regression:

Mandatory parallel run period per wave: both PC and target run simultaneously
Automated daily reconciliation during parallel run
Hard rule: PC schedules not disabled until 10+ successful parallel runs with full reconciliation pass

8.3 Rollback Procedures
Per-Object Rollback:
Copy1. Identify failing pipeline in production
2. Disable pipeline in target platform via Control Plane API
3. Re-enable PowerCenter schedule for that workflow
4. Log rollback event in migration catalog
5. Investigate root cause
6. Fix artifact and re-migrate object
Per-Wave Rollback:
Copy1. Declare wave rollback (requires approval from Migration Lead + Business Sponsor)
2. Disable all wave objects in target platform
3. Re-enable all PowerCenter schedules for wave objects
4. Conduct wave retrospective
5. Root-cause all failures
6. Remediate tooling and/or artifacts
7. Re-execute wave from conversion phase
Factory-Level Rollback (emergency):
Copy1. Never permanently disable PowerCenter until final decommission approval
2. PowerCenter remains fully operational throughout migration
3. Target platform is additive — does not replace PC until wave sign-off
4. Emergency procedure: bulk-disable target platform pipelines, PC resumes automatically
8.4 Contingency Planning
ScenarioContingency PlanAutomation rate significantly lower than projectedIncrease manual migration engineer headcount; adjust wave scope and timelineFactory tooling requires major reworkPause production migration waves; sprint to fix tooling; re-validate pilot objectsTarget platform has unexpected feature gapsEscalate to platform vendor; design workarounds; formally defer affected objects to later wavePowerCenter decommission date is fixed/mandatedTriage objects by business criticality; accept technical debt for low-value complex objects; negotiate scopeData reconciliation fails for >20% of objects in a waveHalt wave; conduct deep-dive analysis; determine if systemic tool issue or object-specific issues; remediate accordingly

9. Governance and Organization
9.1 Team Structure
Copy┌─────────────────────────────────────────────────────────────────────┐
│  MIGRATION FACTORY ORGANIZATION                                       │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  PROGRAM STEERING COMMITTEE                                  │    │
│  │  Executive Sponsor | IT Director | Business Sponsor          │    │
│  └───────────────────────────┬─────────────────────────────────┘    │
│                               │                                      │
│  ┌───────────────────────────▼─────────────────────────────────┐    │
│  │  MIGRATION PROGRAM OFFICE                                    │    │
│  │  Program Manager | Change Manager | Communications Lead      │    │
│  └────┬───────────────────┬──────────────────────┬────────────┘    │
│       │                   │                      │                   │
│  ┌────▼────┐         ┌────▼────┐           ┌────▼─────────┐        │
│  │FACTORY  │         │MIGRATION│           │QUALITY       │        │
│  │TEAM     │         │TEAM(S)  │           │ASSURANCE     │        │
│  │         │         │         │           │TEAM          │        │
│  │Lead     │         │Wave Lead│           │QA Lead       │        │
│  │Architect│         │PC SME(s)│           │QA Engineers  │        │
│  │Tool Dev │         │Platform │           │Data          │        │
│  │(3-5)    │         │SME(s)   │           │Reconciliation│        │
│  │DevOps   │         │Mig. Eng │           │Specialists   │        │
│  │         │         │(3-5 per │           │              │        │
│  │         │         │ wave    │           │              │        │
│  │         │         │ team)   │           │              │        │
│  └─────────┘         └─────────┘           └──────────────┘        │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  BUSINESS VALIDATION TEAM                                     │   │
│  │  Business Analysts (per domain) | Data Owners | UAT Leads     │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
Key Roles and Responsibilities:
RoleResponsibilitiesFTE EstimateMigration Program ManagerProgram oversight, stakeholder management, risk escalation, reporting1Migration Factory Lead / ArchitectFactory design, technical standards, escalation of architectural decisions1PowerCenter SMESource system expertise, knowledge documentation, complex object guidance2-3Target Platform SMETarget platform expertise, JSON schema guidance, Control Plane API expertise2-3Factory Tool DevelopersBuild and maintain factory tooling3-5DevOps / CI-CD EngineerCI/CD pipeline, environment management, Control Plane API integration1-2Wave Team LeadPer-wave migration ownership, daily team standup, issue escalation1 per wave teamMigration EngineersObject conversion, review, manual implementation3-5 per wave teamQA EngineersValidation execution, reconciliation analysis, quality gate enforcement2-3Data Reconciliation SpecialistReconciliation framework development and test analysis1-2Business AnalystBusiness validation, UAT coordination, requirements for complex objects2-3Change ManagerChange management, stakeholder communications, training1
RACI Matrix (Summary):
ActivityFactory LeadTool DevMig. EngineerQABusinessPMTool developmentARCC-IWave planningA-CCCRObject conversionC-RI-IManual reviewA-RC-IValidationC-CR-IUATI-ICRAProduction deploymentAICC-RWave sign-offA-CCRR
R=Responsible, A=Accountable, C=Consulted, I=Informed
9.2 Standards and Decision Governance
Architecture Decision Records (ADRs):
Every significant technical decision is documented as an ADR:

Decision context and problem statement
Options considered
Decision made and rationale
Consequences and trade-offs
Date, decision makers, review date

ADRs are stored in the Git repository (/docs/decisions/) and are immutable once approved. Revisions create new ADRs referencing the original.
Transformation Rule Governance:

All transformation rules are defined in YAML files in the rules/ directory
Changes to rules require PR review by Factory Lead + PowerCenter SME + Platform SME
Rule changes are versioned; existing artifacts indicate which rule version was used
Regression test suite validates rule changes don't break previously converted objects

Naming Convention Standards:
CopyDatastores:   ds_{source_system}_{environment}         → ds_oracle_crm_prod
Datasets:     dset_{schema}_{table_name}_{direction}   → dset_crm_customers_src
Pipelines:    pl_{domain}_{functional_area}_{name}     → pl_crm_customer_load
9.3 Communication Plan
AudienceFrequencyFormatContentOwnerSteering CommitteeMonthlyExecutive dashboardStatus, risks, milestones, decisions neededPMIT LeadershipBi-weeklyStatus reportWave progress, issues, upcoming milestonesPMMigration TeamsDailyStandupDay's objectives, blockers, decisionsWave LeadAll StakeholdersWeeklyEmail/intranet updateProgress, upcoming milestones, changesPMBusiness OwnersPer waveWave briefingWhat's migrating, validation needs, sign-off scheduleBATechnical TeamContinuousWiki/ConfluenceStandards, ADRs, runbooks, lessons learnedFactory Lead
9.4 Change Management Approach
Communication Streams:

IT Operations: How pipeline monitoring, alerting, and on-call processes change
Business Users: How to access new platform monitoring; any changes to SLA/scheduling
Data Teams: New development workflow; how to request pipeline changes post-migration
Support Teams: New troubleshooting procedures; updated runbooks

Training Plan:

Platform training for all migration engineers (mandatory before Wave 1)
"Migrated Pipeline" orientation for business users per wave
Operations runbook training for support teams before production cutover


10. Implementation Roadmap
10.1 Phased Roadmap
CopyMONTH:   1    2    3    4    5    6    7    8    9   10   11   12   13+
         │    │    │    │    │    │    │    │    │    │    │    │    │
PHASE 0  ████████████████████│
FOUNDATION│                  │
         │                  │
PHASE 1  │    ████████████████████│
PILOT    │         │              │
         │         │              │
PHASE 2  │         │    ████████████████████████████████████████
PROD MIG │         │         │
         │         │         │
PHASE 3  │         │         │              ████████████████████
COMPLEX  │         │         │                   │
OBJECTS  │         │         │                   │
         │         │         │                   │
CUTOVER  │         │         │                        ██████████
         │         │         │                             │
10.2 Phase 0: Foundation (Months 1-3)
Objectives: Build the factory, prove it works, establish standards, complete discovery.
MilestoneActivitiesDeliverablesTargetM0.1: Discovery CompleteExtract all repositories, build inventory, score complexity, map dependenciesComplete migration catalog, wave plan, assessment reportMonth 2M0.2: Factory Core BuiltBuild discovery, conversion, validation, and deployment enginesWorking factory CLI, unit tests, CI/CD pipelineMonth 2M0.3: Templates & RulesDefine all transformation rules, create JSON templates, build expression translatorRule YAML files, template library, expression test suiteMonth 3M0.4: Pilot CompleteMigrate 10-20 hand-picked Tier 1 and Tier 2 objects end-to-endValidated pilot artifacts, lessons learned, refined toolingMonth 3M0.5: Standards ApprovedNaming conventions, quality gates, ADRs for key decisionsStandards document, ADR library, quality gate checklistMonth 3
Success Criteria for Phase 0:

Complete object inventory (100% of repositories scanned)
Factory can convert and deploy a simple Tier 1 pipeline end-to-end automatically
Data reconciliation framework operational
Wave 1 plan approved by stakeholders

10.3 Phase 1: Pilot Production (Months 3-5)
Objectives: Validate factory at production scale with real business objects; build confidence.
MilestoneActivitiesDeliverablesTargetM1.1: Wave 1 ConvertedConvert 100-200 Tier 1 objectsValidated JSON artifacts in stagingMonth 4M1.2: Wave 1 ValidatedSchema validation, reconciliation tests passingQG3 sign-off reportMonth 4M1.3: Wave 1 DeployedProduction deployment of Wave 1 objectsProduction artifacts deployedMonth 5M1.4: Wave 1 ClosedParallel run complete, business sign-off, PC schedules disabledWave 1 closure reportMonth 5M1.5: Factory TunedLessons learned incorporated, automation rate measured, tooling improvedUpdated tooling, refined templates and rulesMonth 5
Success Criteria for Phase 1:

≥95% data reconciliation pass rate on Wave 1 objects
Automation rate ≥75% for Tier 1 objects
10+ successful production runs during parallel period before PC disable
Factory throughput sufficient for projected production migration rate

10.4 Phase 2: Production Migration (Months 5-12+)
Objectives: Full-scale migration of Tier 1 and Tier 2 objects across all waves.
Wave Cadence:

Wave duration: 3-4 weeks end-to-end (conversion → production sign-off)
Wave size: 200-400 Tier 1/2 objects per wave
Parallel wave teams: 2-3 teams working simultaneously on different domain/repository groups
Factory tools handle automated conversion; teams focus on review and validation

MilestoneObjects MigratedCumulative %Approximate TimingWave 2 Complete+300 Tier 1 objects~20%Month 7Wave 3 Complete+300 Tier 1/2 objects~40%Month 8Wave 4 Complete+250 Tier 2 objects~57%Month 10Wave 5 Complete+250 Tier 2 objects~73%Month 12All Tier 1/2 CompleteAll Tier 1 and Tier 2~70-80%Month 12
Ongoing Factory Improvements:

Each wave retrospective drives tooling improvements
Automation rate tracked; increases as tooling matures
Expression library grows as new patterns are discovered and added

10.5 Phase 3: Complex Object Migration (Months 10-16)
Objectives: Migrate Tier 3 and Tier 4 objects with significant manual effort.
MilestoneActivitiesDeliverablesTargetM3.1: Tier 3 AssessmentDeep-dive analysis of each Tier 3 object; design custom solutionsImplementation plans per complex objectMonth 10M3.2: Tier 3 MigrationCustom conversion with intensive review and testingAll Tier 3 objects deployedMonth 14M3.3: Tier 4 AssessmentArchitecture review of Tier 4 objects; formal decisionsADRs for each Tier 4 approachMonth 12M3.4: Tier 4 MigrationLargely manual implementation, intensive QAAll Tier 4 objects deployed (or formally deferred)Month 16
Tier 4 Handling Decision Matrix:
For each Tier 4 object, a formal decision is made:

Rewrite: Logic ported to PySpark — high effort, full parity
Workaround: Alternative implementation using available platform capabilities
Defer: Business case reassessed; may be retired or redesigned separately
Accept as-is: Platform extends to accommodate (vendor engagement if needed)

10.6 Cutover and Decommission (Months 15-18)
MilestoneActivitiesDeliverablesTargetM4.1: Full Cutover ReadinessAll objects migrated, all parallel runs validatedCutover readiness reportMonth 15M4.2: Final CutoverPC schedules globally disabled; target platform is system of recordCutover completion sign-offMonth 16M4.3: Hypercare PeriodIntensive monitoring, rapid response to issuesHypercare reportMonth 17M4.4: PC DecommissionPowerCenter license, infrastructure decommissionedDecommission completion sign-offMonth 18
10.7 KPIs and Success Metrics
Factory Performance Metrics:
KPIDescriptionTargetMeasurement FrequencyObjects Migrated per WeekThroughput of the factory≥50 (mature phase)WeeklyAutomation Rate% of objects converted without manual code≥65% overallPer waveFirst-Pass Quality Rate% of objects passing all QGs on first attempt≥80%Per waveData Reconciliation Pass Rate% of pipelines achieving PASS on reconciliation≥95%Per waveCycle TimeDays from wave start to production sign-off≤21 daysPer waveIssue Resolution TimeAverage days to resolve migration issues≤3 days (Medium), ≤1 day (High)WeeklyDefect Escape RatePost-production issues found per wave≤2% of migrated objectsPer wave
Program Metrics:
KPIDescriptionTargetTotal Migration Progress% of inventory migrated and signed off100% at cutoverWave Completion Rate% of waves completed on schedule≥90%Business Sign-Off Rate% of waves signed off by business on first submission≥85%Parallel Run DurationDays of parallel running before PC disable≤14 days per wavePC Decommission Date AdherenceActual vs. planned decommission dateWithin 2 months
Technical Quality Metrics:
KPIDescriptionTargetZero Production Reconciliation FailuresNo data differences found in production after cutover100%Performance SLA AdherenceMigrated pipelines completing within business SLA≥98%Platform Deployment Success Rate% of CI/CD deployments succeeding≥99%Factory Tool Test CoverageUnit test coverage of factory tooling≥80%

Appendix A: Transformation Rules Reference Card
CopyONE-PAGE QUICK REFERENCE FOR MIGRATION ENGINEERS
─────────────────────────────────────────────────
ALWAYS:
  ✓ Check dependency graph before starting any object
  ✓ Verify datastores and datasets exist before pipeline conversion
  ✓ Run schema validation before committing to Git
  ✓ Run reconciliation before promoting to staging
  ✓ Document every manual decision in the object's IR record

NEVER:
  ✗ Modify production artifacts directly (always through Git + CI/CD)
  ✗ Disable PowerCenter schedules without full wave sign-off
  ✗ Deploy to production without QG4 approval
  ✗ Assume an expression translation is correct without testing
  ✗ Mark an object "complete" with open ERROR-severity issues

ESCALATE IMMEDIATELY:
  ⚡ Any object with unresolvable cross-domain dependency
  ⚡ Any expression the translator cannot parse
  ⚡ Any reconciliation failure with >1% row count difference
  ⚡ Any custom/external procedure transformation
  ⚡ Any object with CDC or real-time patterns
  ⚡ Any systemic issue affecting >5 objects in same way
─────────────────────────────────────────────────

Appendix B: Estimated Timeline Summary (3,000 Object Estate)
CopyTier Distribution (estimated):
  Tier 1 – Simple:       1,050 objects (35%)
  Tier 2 – Moderate:     1,050 objects (35%)
  Tier 3 – Complex:        600 objects (20%)
  Tier 4 – Very Complex:   300 objects (10%)
  ─────────────────────────────────────────
  TOTAL:                 3,000 objects

Phase 0 (Foundation):                    3 months
Phase 1 (Pilot Production, Wave 1):      2 months
Phase 2 (Tier 1 bulk + Tier 2 partial): 7 months (Waves 2-5)
Phase 3 (Tier 2 remainder + Tier 3/4):  6 months (Waves 6-9)
Cutover and Decommission:                3 months
─────────────────────────────────────────────────────────
TOTAL PROGRAM DURATION:               ~18-21 months

Team Size:
  Phase 0:   ~8-10 people (heavy on factory developers)
  Phase 1-2: ~15-20 people (wave teams + factory support)
  Phase 3:   ~12-15 people (fewer Tier 1 conversions, more manual Tier 3/4)
  Cutover:   ~8-10 people
