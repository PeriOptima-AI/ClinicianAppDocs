``` mermaid
erDiagram
    ORGANIZATIONS ||--o{ DEPARTMENTS : has
    ORGANIZATIONS ||--o{ USERS : employs
    ORGANIZATIONS ||--o{ PATIENTS : owns
    ORGANIZATIONS ||--o{ DEVICES : owns
    ORGANIZATIONS ||--o{ ALERTS : tracks
    ORGANIZATIONS ||--o{ MESSAGES : contains
    ORGANIZATIONS ||--o{ APPOINTMENTS : schedules
    ORGANIZATIONS ||--o{ AUDIT_LOGS : records
    ORGANIZATIONS ||--o{ CONSENTS : governs
    ORGANIZATIONS ||--o{ USER_SHIFTS : defines
    ORGANIZATIONS ||--o{ ABAC_POLICIES : defines

    DEPARTMENTS ||--o{ USERS : groups
    DEPARTMENTS ||--o{ PATIENTS : treats

    USERS ||--o{ USER_ROLES : has
    ROLES ||--o{ USER_ROLES : assigned_to
    ROLES ||--o{ ROLE_PERMISSIONS : grants
    PERMISSIONS ||--o{ ROLE_PERMISSIONS : included_in

    PATIENTS ||--o{ ASSIGNMENTS : linked_to
    USERS ||--o{ ASSIGNMENTS : assigned_to

    PATIENTS ||--o{ ENCOUNTERS : has
    PATIENTS ||--o{ OBSERVATIONS : has
    USERS ||--o{ OBSERVATIONS : authored
    PATIENTS ||--o{ CARE_PLANS : guided_by

    DEVICES ||--o{ DEVICE_READINGS : produces
    PATIENTS ||--o{ DEVICE_READINGS : receives

    PATIENTS ||--o{ ALERTS : triggers
    USERS ||--o{ ALERTS : created_by
    USERS ||--o{ ALERTS : assigned_to

    USERS ||--o{ MESSAGES : sends
    PATIENTS ||--o{ MESSAGES : about

    USERS ||--o{ APPOINTMENTS : attends_as_clinician
    PATIENTS ||--o{ APPOINTMENTS : attends_as_patient

    USERS ||--o{ AUDIT_LOGS : actor

    USERS ||--o{ USER_ATTRIBUTES : has
    USERS ||--o{ BREAK_GLASS_EVENTS : initiates
    PATIENTS ||--o{ BREAK_GLASS_EVENTS : subject

    ORGANIZATIONS {
      uuid id PK
      text name
      text slug
      text status
      timestamptz created_at
    }
    DEPARTMENTS {
      uuid id PK
      uuid org_id FK
      text name
      timestamptz created_at
    }
    USERS {
      uuid id PK
      uuid org_id FK
      uuid department_id FK
      text email
      text full_name
      boolean is_active
      timestamptz created_at
    }
    ROLES {
      uuid id PK
      text name
      text scope
      text description
    }
    PERMISSIONS {
      uuid id PK
      text resource
      text action
      text name
      text description
    }
    USER_ROLES {
      uuid id PK
      uuid user_id FK
      uuid role_id FK
      uuid org_id FK
      uuid department_id
    }
    ROLE_PERMISSIONS {
      uuid id PK
      uuid role_id FK
      uuid permission_id FK
    }
    PATIENTS {
      uuid id PK
      uuid org_id FK
      uuid department_id FK
      text mrn
      text first_name
      text last_name
      date dob
      text sex
      text status
      timestamptz created_at
    }
    ASSIGNMENTS {
      uuid id PK
      uuid patient_id FK
      uuid user_id FK
      text role_context
      timestamptz start_at
      timestamptz end_at
    }
    ENCOUNTERS {
      uuid id PK
      uuid patient_id FK
      text facility
      timestamptz admission_at
      timestamptz discharge_at
      text type
    }
    DEVICES {
      uuid id PK
      uuid org_id FK
      uuid patient_id
      text type
      text manufacturer
      text serial
      text status
    }
    DEVICE_READINGS {
      uuid id PK
      uuid device_id FK
      uuid patient_id FK
      timestamptz captured_at
      text metric
      jsonb value_json
    }
    OBSERVATIONS {
      uuid id PK
      uuid patient_id FK
      uuid author_user_id FK
      timestamptz observed_at
      text type
      jsonb value_json
    }
    CARE_PLANS {
      uuid id PK
      uuid patient_id FK
      text status
      jsonb plan_json
      timestamptz updated_at
    }
    ALERTS {
      uuid id PK
      uuid org_id FK
      uuid patient_id FK
      text severity
      text type
      text status
      uuid created_by FK
      uuid assigned_to
      timestamptz created_at
    }
    MESSAGES {
      uuid id PK
      uuid org_id FK
      uuid thread_id
      uuid sender_user_id FK
      uuid patient_id
      text body
      timestamptz created_at
    }
    APPOINTMENTS {
      uuid id PK
      uuid org_id FK
      uuid patient_id FK
      uuid clinician_user_id FK
      timestamptz scheduled_start
      timestamptz scheduled_end
      text status
    }
    AUDIT_LOGS {
      uuid id PK
      uuid org_id FK
      uuid user_id FK
      text action
      text resource_type
      uuid resource_id
      boolean success
      inet ip
      timestamptz at
      jsonb extra_json
    }
    CONSENTS {
      uuid id PK
      uuid org_id FK
      uuid patient_id FK
      text consent_type
      boolean granted
      timestamptz granted_at
      timestamptz revoked_at
    }
    USER_SHIFTS {
      uuid id PK
      uuid user_id FK
      uuid org_id FK
      timestamptz starts_at
      timestamptz ends_at
      text timezone
    }
    USER_ATTRIBUTES {
      uuid id PK
      uuid user_id FK
      text key
      text value
    }
    BREAK_GLASS_EVENTS {
      uuid id PK
      uuid user_id FK
      uuid patient_id FK
      text reason
      timestamptz started_at
      timestamptz ended_at
      uuid approved_by
    }
    ABAC_POLICIES {
      uuid id PK
      uuid org_id FK
      text name
      text effect
      jsonb condition_json
      int priority
    }

    %% Notes:
    %% - Enums like status values (e.g., inpatient/discharged/monitoring) and UNIQUE/INDEX constraints
    %%   aren’t supported inline; document them here or in SQL.
    %% - Where you had "FK -> table.id", Mermaid only supports the FK tag, not the reference target.

```
