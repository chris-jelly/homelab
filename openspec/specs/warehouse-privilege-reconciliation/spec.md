## Requirements

### Requirement: Warehouse read roles SHALL be declaratively managed
The system SHALL define warehouse read-access principals declaratively in CNPG role management for each environment. The role model SHALL include consumer access for both Airflow and Streamlit and SHALL avoid shared human/application login credentials.

#### Scenario: Consumer principals exist for Airflow and Streamlit
- **WHEN** warehouse CNPG cluster manifests are reconciled
- **THEN** declarative role definitions include read-access principals for Airflow and Streamlit consumers

#### Scenario: Consumer credentials are isolated
- **WHEN** consumer role configuration is reviewed
- **THEN** Airflow and Streamlit do not share the same LOGIN secret material

### Requirement: Privilege reconciler SHALL converge existing object grants
The system SHALL run an idempotent privilege reconciler that grants read access across existing warehouse objects in approved schemas. The reconciler SHALL grant database CONNECT, schema USAGE, and object-level read privileges required for query execution.

#### Scenario: Existing schemas and tables become readable
- **WHEN** the reconciler runs after schemas and tables already exist
- **THEN** the configured read principals can query approved schemas without manual grant commands

#### Scenario: Reconciler rerun is safe
- **WHEN** the reconciler executes multiple times with no schema changes
- **THEN** executions complete without privilege duplication errors and effective access remains unchanged

### Requirement: Privilege reconciler SHALL configure future-object access
The system SHALL configure Postgres default privileges for the designated warehouse object-creator role so newly created DAG-managed objects in approved schemas inherit read access for consumer principals.

#### Scenario: New table created after bootstrap remains readable
- **WHEN** DAG workflows create a new table in an approved schema after initial bootstrap
- **THEN** Airflow and Streamlit read principals can SELECT from the table without additional manual grants

#### Scenario: Default privileges are tied to designated creator role
- **WHEN** default privilege SQL is applied
- **THEN** it targets the environment's designated object-creator role used by DAG-managed DDL

### Requirement: Reconciler SHALL include Streamlit in warehouse read policy
The system SHALL explicitly include Streamlit as a managed warehouse read consumer and SHALL validate Streamlit read-path access using the same privilege convergence model as Airflow.

#### Scenario: Streamlit is covered by reconciliation scope
- **WHEN** reconciler configuration is evaluated
- **THEN** Streamlit read principal is included in grant and default-privilege convergence logic

#### Scenario: Streamlit reads newly created DAG objects
- **WHEN** a new approved-schema object is created by DAG workflows
- **THEN** Streamlit can read the object after reconciliation convergence without manual SQL intervention

### Requirement: Reconciler failures SHALL be observable
The system SHALL expose reconciler execution status so repeated failures are detectable during operations and bootstrap validation.

#### Scenario: Failed reconciliation is visible
- **WHEN** the reconciler run fails due to SQL or connectivity errors
- **THEN** workload status and logs provide actionable failure signals for operators

#### Scenario: Bootstrap validation checks reconciliation health
- **WHEN** a cluster is provisioned from scratch
- **THEN** runbook validation includes a successful reconciler execution and verification of read access for consumer roles
