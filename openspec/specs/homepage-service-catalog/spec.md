## Requirements

### Requirement: Dashboard-worthy applications appear in Homepage
The cluster SHALL represent applications and operator-facing services in Homepage when they are intended to be launched or inspected from the dashboard.

#### Scenario: Application is intended for Homepage
- **WHEN** an application is introduced or reviewed for ongoing use
- **THEN** the change SHALL include an explicit decision to represent it in Homepage or intentionally exclude it from Homepage

#### Scenario: Backing workload is not dashboard-worthy
- **WHEN** a workload is only a helper, backing component, or implementation detail
- **THEN** the workload MAY be excluded from Homepage without creating a dashboard entry

### Requirement: Ingress-backed applications declare Homepage metadata on the ingress
Applications that belong in Homepage and expose an ingress SHALL declare Homepage presentation metadata on that ingress, regardless of whether the ingress is public or local-only.

#### Scenario: Public ingress-backed application
- **WHEN** an application belongs in Homepage and exposes a public ingress
- **THEN** the ingress SHALL carry the Homepage metadata required to discover and present the application correctly

#### Scenario: Local-only ingress-backed application
- **WHEN** an application belongs in Homepage and exposes a local-only ingress
- **THEN** the ingress SHALL carry the same Homepage metadata used for public ingress-backed applications

### Requirement: Non-ingress applications use curated Homepage entries
Applications that belong in Homepage but do not expose an ingress SHALL be represented through curated Homepage configuration.

#### Scenario: Application has no ingress
- **WHEN** an application belongs in Homepage and has no ingress
- **THEN** Homepage SHALL include a curated entry for that application instead of relying on ingress discovery

#### Scenario: Application gains an ingress later
- **WHEN** an application moves from curated representation to ingress-backed discovery
- **THEN** the manual Homepage entry SHALL be updated or removed to avoid duplicate dashboard entries

### Requirement: Homepage status checks rely on stable application identity
Applications represented in Homepage SHALL use stable Kubernetes identity metadata so Homepage status indicators can resolve the correct workloads.

#### Scenario: Application workload labels are normalized
- **WHEN** an application belongs in Homepage
- **THEN** its Kubernetes resources SHALL use standard application identity labels or an explicit selector that resolves the intended workload

#### Scenario: Multi-component application is represented by one entrypoint
- **WHEN** Homepage represents a multi-component system
- **THEN** the Homepage entry SHALL target the operator-facing entrypoint rather than every internal component

### Requirement: Homepage layout distinguishes applications from platform services
Homepage SHALL distinguish application launch surfaces from platform and operator services so tiles and widgets reflect user intent.

#### Scenario: Application entry is rendered
- **WHEN** an application appears in Homepage
- **THEN** it SHALL be grouped with other application entries rather than with platform operators

#### Scenario: Operator entry is rendered
- **WHEN** an operator or platform service appears in Homepage
- **THEN** it SHALL be grouped separately from application entries and SHALL not rely on a misleading application widget

### Requirement: New changes keep Homepage aligned with deployments
Changes that introduce or materially change Homepage-worthy applications SHALL update Homepage representation in the same change.

#### Scenario: New Homepage-worthy application is added
- **WHEN** a change adds an application that belongs in Homepage
- **THEN** the same change SHALL add the required ingress metadata or curated Homepage entry for that application

#### Scenario: Existing Homepage-worthy application changes access pattern
- **WHEN** a change adds, removes, or changes an ingress or entrypoint for an application already represented in Homepage
- **THEN** the same change SHALL update Homepage representation to match the new access pattern
