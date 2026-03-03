## ADDED Requirements

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

### Requirement: Admin role SHALL default to least privilege
The system SHALL configure admin LOGIN role attributes with least-privilege defaults and SHALL avoid superuser privilege unless explicitly documented by a separate approved requirement.

#### Scenario: Superuser is not implicitly granted
- **WHEN** admin role attributes are inspected
- **THEN** the role is not configured as superuser by default

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
