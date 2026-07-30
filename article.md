# Protection That Follows the Data — Automatically

I've spent fifteen years working in enterprise data, and if there's one problem I've seen at virtually every organization, it's this: nobody actually knows what sensitive data they have or where it lives.

That's not a knock on those teams. It's just the reality of how enterprise data warehouses grow. Years of ETL pipelines, migrations, and table sprawl leave behind schemas full of columns named `col_3`, `legacy_field`, `temp_b`. Somewhere in there are SSNs. Salaries. Email addresses. Probably. The security team wants to know for certain, but manually auditing thousands of tables across a mature data warehouse just isn't feasible.

And even the organizations that *have* done that work face a second problem that's just as persistent. One of the hallmarks of a mature data organization is giving teams their own space to explore — sandboxes, team schemas, shared workspaces. It's genuinely valuable. It's also, historically, one of the easier ways for governed data to end up somewhere it shouldn't be. A privileged analyst with legitimate access to the HR warehouse copies a table into the team space for some ad hoc analysis. Her teammates, who never completed PII training and were never supposed to have access to that data, can now see every name, SSN, and salary in plain text. Subverting the intended security model didn't require any malicious intent — just a CTAS and a few seconds.

There's a related scenario that keeps compliance teams up at night. Data subject to regulatory guidance — think DSAR, a customer's right to request deletion of their personal data — is only manageable if you actually know where all copies of that data live. Normal deletion procedures follow known paths. They don't follow data that got quietly copied into a team workspace six months ago by someone who needed it for a quick analysis and never cleaned it up.

I've watched organizations spend months, sometimes over a year, trying to solve these two problems with custom tooling, manual tagging processes, and governance frameworks that were outdated before they were finished. It was genuinely hard. The scale of the problem made it feel almost intractable.

At some point recently I had one of those moments where you realize — wait, these things can actually all hook together now. Auto-classification finds your sensitive data. The classification profile maps those detections to a user-defined tag. That tag carries masking policies. And that tag propagates when data moves. It's a genuinely elegant chain, and I don't think enough people have noticed yet that it works end-to-end.

What used to be a months-long governance project is now a handful of SQL statements.

This walkthrough shows exactly how to set it up.

---

## The Architecture

Three databases, clean separation of concerns:

| Database | Purpose |
|---|---|
| `GOVERNANCE_DB` | Tags, masking policies, classification profile |
| `ENTERPRISE_DB` | The protected HR source |
| `TEAMSPACE_DB` | Alice's shared exploration space |

Three roles:

| Role | Who | What they see |
|---|---|---|
| `DATA_GOVERNOR` | Governance admin | Sets up controls — not exempt from them |
| `PRIVILEGED_ANALYST` | Alice, PII-trained | Unmasked data |
| `ANALYST` | Alice's teammates | Masked PII columns |

The key design decision: `DATA_GOVERNOR` is **not** in the masking policy bypass. The governance admin sets up the controls but doesn't have special data access. Separation of duties.

---

## Step 1: Governance Infrastructure

```sql
USE ROLE SYSADMIN;

CREATE DATABASE governance_db;
CREATE SCHEMA governance_db.tags;
CREATE SCHEMA governance_db.masking_policies;
CREATE SCHEMA governance_db.classify;
```

### Create the `pii` tag

```sql
CREATE TAG governance_db.tags.pii
  ALLOWED_VALUES 'high', 'medium', 'low'
  COMMENT = 'PII sensitivity tier';
```

### Create masking policies

Only `PRIVILEGED_ANALYST` — users who have completed PII training — see unmasked values. Everyone else, including the governance admin, is masked.

```sql
CREATE MASKING POLICY governance_db.masking_policies.pii_string_mask
  AS (val STRING) RETURNS STRING ->
  CASE WHEN CURRENT_ROLE() = 'PRIVILEGED_ANALYST' THEN val ELSE '***REDACTED***' END;

CREATE MASKING POLICY governance_db.masking_policies.pii_number_mask
  AS (val NUMBER) RETURNS NUMBER ->
  CASE WHEN CURRENT_ROLE() = 'PRIVILEGED_ANALYST' THEN val ELSE -1 END;
```

### Attach masking policies to the `pii` tag

One `ALTER TAG` binds both policies. Snowflake applies the right one per column data type automatically.

```sql
ALTER TAG governance_db.tags.pii
  SET MASKING POLICY governance_db.masking_policies.pii_string_mask,
      MASKING POLICY governance_db.masking_policies.pii_number_mask;
```

---

## Step 2: Data Infrastructure

```sql
USE ROLE SYSADMIN;

CREATE DATABASE enterprise_db COMMENT = 'Enterprise data warehouse — governed source';
CREATE SCHEMA enterprise_db.hr;

CREATE DATABASE teamspace_db COMMENT = 'Shared exploration space — copy destination';
CREATE SCHEMA teamspace_db.shared;

CREATE TABLE enterprise_db.hr.employees (
  employee_id   NUMBER,
  first_name    VARCHAR,
  last_name     VARCHAR,
  email         VARCHAR,
  phone         VARCHAR,
  ssn           VARCHAR,
  salary        NUMBER
);

INSERT INTO enterprise_db.hr.employees VALUES
  (1, 'Sarah',   'Chen',    'sarah.chen@acme.com',    '415-555-0101', '523-45-6789', 142000),
  (2, 'Marcus',  'Webb',    'marcus.webb@acme.com',   '312-555-0182', '614-77-3210', 98500),
  (3, 'Priya',   'Sharma',  'priya.sharma@acme.com',  '646-555-0147', '731-22-8841', 117000),
  (4, 'James',   'Okafor',  'james.okafor@acme.com',  '404-555-0193', '829-56-1234', 155000),
  (5, 'Elena',   'Vasquez', 'elena.vasquez@acme.com', '206-555-0166', '943-11-5678', 89000);
```

---

## Step 3: Roles

```sql
USE ROLE USERADMIN;
CREATE ROLE data_governor;
CREATE ROLE privileged_analyst;
CREATE ROLE analyst;
GRANT ROLE data_governor TO ROLE SYSADMIN;
GRANT ROLE privileged_analyst TO ROLE SYSADMIN;
GRANT ROLE analyst TO ROLE SYSADMIN;

USE ROLE SYSADMIN;
GRANT USAGE ON WAREHOUSE demo_wh TO ROLE data_governor;
GRANT USAGE ON WAREHOUSE demo_wh TO ROLE privileged_analyst;
GRANT USAGE ON WAREHOUSE demo_wh TO ROLE analyst;

-- data_governor: read access to both databases
GRANT USAGE ON DATABASE enterprise_db TO ROLE data_governor;
GRANT USAGE ON ALL SCHEMAS IN DATABASE enterprise_db TO ROLE data_governor;
GRANT SELECT ON ALL TABLES IN DATABASE enterprise_db TO ROLE data_governor;
GRANT USAGE ON DATABASE teamspace_db TO ROLE data_governor;
GRANT USAGE ON ALL SCHEMAS IN DATABASE teamspace_db TO ROLE data_governor;
GRANT SELECT ON ALL TABLES IN DATABASE teamspace_db TO ROLE data_governor;

-- privileged_analyst: read EDW + CREATE TABLE in teamspace to run the CTAS
GRANT USAGE ON DATABASE enterprise_db TO ROLE privileged_analyst;
GRANT USAGE ON ALL SCHEMAS IN DATABASE enterprise_db TO ROLE privileged_analyst;
GRANT SELECT ON ALL TABLES IN DATABASE enterprise_db TO ROLE privileged_analyst;
GRANT USAGE ON DATABASE teamspace_db TO ROLE privileged_analyst;
GRANT USAGE ON ALL SCHEMAS IN DATABASE teamspace_db TO ROLE privileged_analyst;
GRANT CREATE TABLE ON SCHEMA teamspace_db.shared TO ROLE privileged_analyst;

-- analyst: read both databases, no CREATE TABLE
GRANT USAGE ON DATABASE enterprise_db TO ROLE analyst;
GRANT USAGE ON ALL SCHEMAS IN DATABASE enterprise_db TO ROLE analyst;
GRANT SELECT ON ALL TABLES IN DATABASE enterprise_db TO ROLE analyst;
GRANT USAGE ON DATABASE teamspace_db TO ROLE analyst;
GRANT USAGE ON ALL SCHEMAS IN DATABASE teamspace_db TO ROLE analyst;

-- future grant requires ACCOUNTADMIN (MANAGE GRANTS privilege)
USE ROLE ACCOUNTADMIN;
GRANT SELECT ON FUTURE TABLES IN SCHEMA teamspace_db.shared TO ROLE analyst;
```

---

## Step 4: Auto-Classification

This is where it gets interesting. Instead of manually tagging every column, we let Snowflake's ML models do the detection — and configure a **tag map** that automatically applies our `pii` tag based on what's found.

### Create the classification profile

The tag map is the bridge between Snowflake's system-detected categories and our user-defined `pii` tag. Direct masking on system tags isn't supported — the tag map is the required connection.

```sql
USE ROLE SYSADMIN;

CREATE SNOWFLAKE.DATA_PRIVACY.CLASSIFICATION_PROFILE
  governance_db.classify.pii_profile(
    {
      'minimum_object_age_for_classification_days': 0,
      'auto_tag': true,
      'tag_map': {
        'column_tag_map': [
          {
            'tag_name': 'governance_db.tags.pii',
            'tag_value': 'high',
            'semantic_categories': ['NAME', 'EMAIL', 'NATIONAL_IDENTIFIER']
          },
          {
            'tag_name': 'governance_db.tags.pii',
            'tag_value': 'medium',
            'semantic_categories': ['PHONE_NUMBER', 'SALARY']
          }
        ]
      }
    }
  );
```

### Run classification

For a single table, `SYSTEM$CLASSIFY` runs immediately and returns results inline — which makes it useful for reviewing what Snowflake detected before committing to anything. If you want to inspect the recommendations before tags are applied, set `auto_tag: false` in the profile, run `SYSTEM$CLASSIFY`, review the output via `SYSTEM$GET_CLASSIFICATION_RESULT`, then apply tags manually or flip `auto_tag` back to true.

In this walkthrough we're running with `auto_tag: true`, so tags are applied in the same call:

```sql
CALL SYSTEM$CLASSIFY('enterprise_db.hr.employees', 'governance_db.classify.pii_profile');
```

For a full schema, use `SYSTEM$CLASSIFY_SCHEMA` — it schedules classification across all tables in parallel, so there can be some lag before all results are available:

```sql
CALL SYSTEM$CLASSIFY_SCHEMA('enterprise_db.hr', {'auto_tag': true});
```

For ongoing, automatic classification at the database level, you can associate the profile with the database directly and Snowflake will classify new and modified tables on a background schedule:

```sql
ALTER DATABASE enterprise_db SET CLASSIFICATION_PROFILE = 'governance_db.classify.pii_profile';
```

In a production environment that last option is the most scalable — classify once, keep classifying as the warehouse grows.

Snowflake scans column names and sample values, applies system tags (`SNOWFLAKE.CORE.SEMANTIC_CATEGORY`, `SNOWFLAKE.CORE.PRIVACY_CATEGORY`), and — because of the tag map — simultaneously stamps our `pii` tag with the right sensitivity tier.

The result on `enterprise_db.hr.employees`:

| Column | Detected As | `pii` Tag |
|---|---|---|
| `employee_id` | — | not tagged |
| `first_name` | NAME | `high` |
| `last_name` | NAME | `high` |
| `email` | EMAIL | `high` |
| `phone` | PHONE_NUMBER | `medium` |
| `ssn` | NATIONAL_IDENTIFIER (US_SSN) | `high` |
| `salary` | SALARY | `medium` |

---

## Step 5: Verify Masking at the Source

Use `USE SECONDARY ROLES NONE` to ensure the test is clean — only the primary role is active.

As `ANALYST` — PII columns masked, `employee_id` visible:

```sql
USE ROLE ANALYST;
USE SECONDARY ROLES NONE;
SELECT * FROM enterprise_db.hr.employees;
```

```
EMPLOYEE_ID  FIRST_NAME     LAST_NAME      EMAIL          PHONE          SSN            SALARY
1            ***REDACTED*** ***REDACTED*** ***REDACTED*** ***REDACTED*** ***REDACTED*** -1
2            ***REDACTED*** ***REDACTED*** ***REDACTED*** ***REDACTED*** ***REDACTED*** -1
...
```

As `PRIVILEGED_ANALYST` — full visibility:

```sql
USE ROLE PRIVILEGED_ANALYST;
USE SECONDARY ROLES NONE;
SELECT * FROM enterprise_db.hr.employees;
```

```
EMPLOYEE_ID  FIRST_NAME  LAST_NAME  EMAIL                    PHONE         SSN           SALARY
1            Sarah       Chen       sarah.chen@acme.com      415-555-0101  523-45-6789   142000
2            Marcus      Webb       marcus.webb@acme.com     312-555-0182  614-77-3210   98500
...
```

---

## Step 6: Protection Follows the Data

Now for the key test. Alice copies the table to the team space.

**Before the copy**, configure the `pii` tag to propagate on data movement:

```sql
USE ROLE SYSADMIN;
ALTER TAG governance_db.tags.pii SET PROPAGATE = ON_DATA_MOVEMENT;
```

> `ON_DATA_MOVEMENT` fires at copy time — CTAS, INSERT, MERGE, COPY INTO. It's a one-shot mechanism: set it before the copy happens.

Alice runs the CTAS:

```sql
USE ROLE PRIVILEGED_ANALYST;
USE SECONDARY ROLES NONE;
CREATE TABLE teamspace_db.shared.employees_copy
  AS SELECT * FROM enterprise_db.hr.employees;
```

The `pii` tag propagates from every tagged source column to the corresponding destination column. Alice didn't do anything special — propagation is automatic.

### Verify masking is enforced in the copy

Alice's teammates query the copy:

```sql
USE ROLE ANALYST;
USE SECONDARY ROLES NONE;
SELECT * FROM teamspace_db.shared.employees_copy;
```

```
EMPLOYEE_ID  FIRST_NAME     LAST_NAME      EMAIL          PHONE          SSN            SALARY
1            ***REDACTED*** ***REDACTED*** ***REDACTED*** ***REDACTED*** ***REDACTED*** -1
2            ***REDACTED*** ***REDACTED*** ***REDACTED*** ***REDACTED*** ***REDACTED*** -1
...
```

Same masking. Different table. Different database. Didn't matter.

---

## Step 7: Audit

`POLICY_REFERENCES` shows every column currently protected by a given masking policy across all objects:

```sql
SELECT *
FROM TABLE(
  governance_db.information_schema.policy_references(
    policy_name => 'governance_db.masking_policies.pii_string_mask'
  )
);
```

`TAG_REFERENCES_ALL_COLUMNS` confirms the `pii` tag is present on both tables:

```sql
-- source
SELECT * FROM TABLE(
  enterprise_db.information_schema.tag_references_all_columns(
    'enterprise_db.hr.employees', 'table'
  )
) WHERE tag_name = 'PII';

-- copy
SELECT * FROM TABLE(
  teamspace_db.information_schema.tag_references_all_columns(
    'teamspace_db.shared.employees_copy', 'table'
  )
) WHERE tag_name = 'PII';
```

---

## Summary

One tag. Two masking policies. A classification profile with a tag map. That's the entire governance layer.

When Alice copied the data, she didn't bypass any controls. She couldn't. The `pii` tag traveled with the columns, and the masking policy attached to that tag fired automatically in the new location — regardless of who ran the copy or where it landed.

The chain:

```
SYSTEM$CLASSIFY detects PII
  → tag map applies pii tag to sensitive columns
    → masking policy on pii tag fires at query time
      → Alice runs CTAS
        → pii tag propagates via ON_DATA_MOVEMENT
          → masking fires in the copy
```

Auto-classify once. The rest is automatic.

---

## Teardown

```sql
USE ROLE SYSADMIN;
DROP DATABASE teamspace_db;
DROP DATABASE enterprise_db;
DROP DATABASE governance_db;

USE ROLE USERADMIN;
DROP ROLE analyst;
DROP ROLE privileged_analyst;
DROP ROLE data_governor;
```
