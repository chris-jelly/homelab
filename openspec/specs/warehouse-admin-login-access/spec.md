## Requirements

### Requirement: Warehouse admin LOGIN role SHALL be declaratively managed
The system SHALL define a dedicated warehouse admin LOGIN role per environment for human pgAdmin operations using CNPG declarative role management.

#### Scenario: Admin LOGIN role exists after reconciliation
- **WHEN** warehouse CNPG cluster manifests are applied
- **THEN** an admin LOGIN role exists and can authenticate with managed credentials

#### Scenario: Admin role is distinct from application principals
- **WHEN** role assignments are reviewed
- **THEN** admin LOGIN role is separate from Airflow and Streamlit application read principals

### Requirement: Admin credentials SHALL be sourced from Azure Key Vault
The system SHALL source warehouse admin LOGIN password material from Azure Key Vault via External Secrets synchronization to Kubernetes and SHALL reference that synchronized secret from CNPG role configuration.

#### Scenario: Key Vault is source of truth for admin secret
- **WHEN** External Secrets resources reconcile
- **THEN** Kubernetes admin credential secret value is populated from Azure Key Vault rather than committed plaintext manifests

#### Scenario: CNPG role references synchronized secret
- **WHEN** CNPG managed role configuration is evaluated
- **THEN** admin LOGIN role password source points to the Kubernetes secret maintained by External Secrets

### Requirement: Admin role SHALL declare explicit privilege posture
The system SHALL configure admin LOGIN role attributes with an explicit and documented privilege posture, and any elevation (including superuser) SHALL be tracked as an approved, reviewed configuration change.

#### Scenario: Superuser is only granted by explicit declaration
- **WHEN** admin role attributes are inspected
- **THEN** superuser is enabled only when explicitly declared in managed role configuration

#### Scenario: Elevated privilege requires explicit decision
- **WHEN** operational need for broader privileges is identified
- **THEN** privilege elevation is tracked as an explicit, reviewed change rather than applied ad hoc

### Requirement: Admin access SHALL be validated for pgAdmin connectivity
The system SHALL provide a documented validation path confirming the admin LOGIN role can connect through the standard warehouse connection endpoint and perform approved administrative operations.

#### Scenario: Connection test succeeds
- **WHEN** operators authenticate with the admin LOGIN role in pgAdmin
- **THEN** connection to the warehouse read/write endpoint succeeds for the target environment

#### Scenario: Validation remains within approved boundaries
- **WHEN** admin validation is executed
- **THEN** only approved administrative actions are validated and application read-only policies remain unaffected
