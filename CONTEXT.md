# Medium Article: Tag-Based Masking Policies in Snowflake
## Project Context — pick up here in a new session

---

## Article concept

Write a Medium article demonstrating the full governance chain:
**auto-classification → tag propagation → tag-based masking → "protection follows the data"**

The narrative: an "exploration" user with privileged access copies sensitive data from an enterprise
warehouse into a shared team space. Because tag propagation is configured, masking follows the data
automatically — even users without privileged access in the team space see masked output.

Opens with automatic classification (create fake PII data, let Snowflake detect it), then builds
the governance layer, then proves it survives a CTAS copy.

---

## Snowflake account

- Connection: `bfrank_demo`
- Account: `SFPSCOGS-BFRANK`
- User: `BFRANK`
- Role: `SYSADMIN` (confirmed working)
- Warehouse: `DEMO_WH`
- Auth: PAT stored in secret `demo_account_token`
- NOTE: `cortex sql_execute` may have a stale keychain entry — use Python connector with
  `NEW_TOKEN="<demo_account_token>"` injection as fallback. Token confirmed valid.

---

## Docs read (all verified July 30, 2026)

- https://docs.snowflake.com/en/user-guide/tag-based-masking-policies
- https://docs.snowflake.com/en/user-guide/object-tagging/propagation
- https://docs.snowflake.com/en/user-guide/tutorials/sensitive-data-auto-classification
- https://docs.snowflake.com/en/user-guide/classify-intro
- https://docs.snowflake.com/en/user-guide/classify-auto
- https://docs.snowflake.com/en/user-guide/classify-native

---

## The accurate technical chain

```
CREATE TABLE with fake PII data
  → SYSTEM$CLASSIFY('db.sch.table', 'profile')   ← immediate, no 1-hour wait
      → SNOWFLAKE.CORE.SEMANTIC_CATEGORY applied (NAME, EMAIL, NATIONAL_IDENTIFIER, SALARY...)
      → SNOWFLAKE.CORE.PRIVACY_CATEGORY applied (IDENTIFIER, QUASI_IDENTIFIER, SENSITIVE)
      → user-defined `pii` tag applied via classification profile tag map
          → masking policy attached to `pii` tag fires on matching columns
              → explorer runs CTAS into team space
                  → pii tag propagates (PROPAGATE = ON_DATA_MOVEMENT)
                      → masking enforced in the copy
```

---

## Key technical facts from docs

**Classification:**
- Requires Enterprise Edition
- Two system tags applied: `SNOWFLAKE.CORE.SEMANTIC_CATEGORY` and `SNOWFLAKE.CORE.PRIVACY_CATEGORY`
- Privacy categories: IDENTIFIER, QUASI_IDENTIFIER, SENSITIVE
- Classification profile is an instance of `SNOWFLAKE.DATA_PRIVACY.CLASSIFICATION_PROFILE`
- `auto_tag: true` in profile causes tags to be applied automatically
- `SYSTEM$CLASSIFY('db.sch.table', 'db.sch.profile')` — runs classification immediately (no 1hr wait)
  Use this for the demo; the `ALTER DATABASE ... SET CLASSIFICATION_PROFILE` approach has a 1hr delay
- Results viewable via `SYSTEM$GET_CLASSIFICATION_RESULT('db.sch.table')`

**Tag maps:**
- Added to classification profile via `profile!SET_TAG_MAP(...)` or inline at creation
- Maps SEMANTIC_CATEGORY values → user-defined tags with specific values
- Example: NAME/NATIONAL_IDENTIFIER → `pii = 'high'`, EMAIL → `pii = 'medium'`, SALARY → `pii = 'medium'`

**OPEN QUESTION — system tag masking limitation:**
- Public docs (tag-based masking limitations section) state: "A masking policy cannot be assigned to a system tag."
- But internal enablement materials (Glean: "Snowflake PII Detection & Redaction" doc + PDF) show:
  `ALTER TAG SNOWFLAKE.CORE.SEMANTIC_CATEGORY SET MASKING POLICY email_mask;`
- This was NOT tested yet — start the new session by testing this empirically in bfrank_demo
- Result will determine whether the article should highlight user-defined tag path as "recommended"
  vs "required"

**Tag-based masking:**
- A tag can hold one masking policy per data type (one for STRING, one for NUMBER, etc.)
- Policy assigned via: `ALTER TAG my_tag SET MASKING POLICY p1, MASKING POLICY p2;`
- Tag applied to a TABLE → all columns of matching data types are masked
- Tag applied to a COLUMN → only that column masked
- If both a direct column policy AND a tag-based policy exist, the direct one takes precedence
- Policy can use CURRENT_ROLE() (role-based) or SYSTEM$GET_TAG_ON_CURRENT_COLUMN() (tag-value-based)
- CANNOT be assigned to system tags (per public docs — but see open question above)

**Tag propagation:**
- `PROPAGATE = ON_DATA_MOVEMENT` — propagates on CTAS, INSERT, MERGE, UPDATE, COPY INTO
- `PROPAGATE = ON_DEPENDENCY` — propagates continuously to views, dynamic tables, materialized views
- `PROPAGATE = ON_DEPENDENCY_AND_DATA_MOVEMENT` — both
- Data movement propagation is ONE-SHOT (not updated if source tag changes later)
- `CREATE TABLE ... CLONE` and `CREATE TABLE ... LIKE` ALWAYS propagate tags (no PROPAGATE setting needed)
- Tags do NOT propagate from a share to local consumer objects
- Set PROPAGATE before CTAS happens — after is too late for existing tables
- Conflict handling: default writes 'CONFLICT', or set ON_CONFLICT = 'HIGHLY CONFIDENTIAL' or ALLOWED_VALUES_SEQUENCE

**Masking + materialized views:**
- A materialized view CANNOT be created on a table protected by tag-based masking
- If MV exists and tag-based masking is added later, the MV is invalidated

---

## Planned demo structure

| Database | Schema | Contents |
|---|---|---|
| `GOVERNANCE_DB` | `TAGS` | user-defined `pii` tag |
| `GOVERNANCE_DB` | `MASKING_POLICIES` | STRING and NUMBER masking policies |
| `GOVERNANCE_DB` | `CLASSIFY` | classification profile instance |
| `ENTERPRISE_DB` | `HR` | source `employees` table (the "protected" source) |
| `TEAMSPACE_DB` | `SHARED` | explorer's CTAS copy destination |

**Source table design** (chosen for rich classification signal):
```sql
CREATE TABLE enterprise_db.hr.employees (
  employee_id   NUMBER,    -- not classified
  first_name    VARCHAR,   -- NAME → IDENTIFIER
  last_name     VARCHAR,   -- NAME → IDENTIFIER
  email         VARCHAR,   -- EMAIL → IDENTIFIER
  phone         VARCHAR,   -- PHONE_NUMBER → QUASI_IDENTIFIER
  ssn           VARCHAR,   -- NATIONAL_IDENTIFIER (US_SSN) → IDENTIFIER
  salary        NUMBER     -- SALARY → SENSITIVE
);
```

**Roles for demo:**
- `SYSADMIN` or `ACCOUNTADMIN` for setup (note in article that prod uses least-privilege)
- `DATA_GOVERNOR` — sees unmasked data
- `ANALYST` — sees masked data (to simulate the unprivileged team space user)

**Masking policy logic (role-based, simpler):**
```sql
CREATE MASKING POLICY pii_string_mask AS (val STRING) RETURNS STRING ->
  CASE WHEN CURRENT_ROLE() IN ('DATA_GOVERNOR') THEN val ELSE '***REDACTED***' END;

CREATE MASKING POLICY pii_number_mask AS (val NUMBER) RETURNS NUMBER ->
  CASE WHEN CURRENT_ROLE() IN ('DATA_GOVERNOR') THEN val ELSE -1 END;
```

---

## Clarifying questions for user (not yet answered)

1. **Classification demo mode:** Use `SYSTEM$CLASSIFY` (immediate) or profile background job
   (pre-run in advance)? Recommend `SYSTEM$CLASSIFY` for the article walkthrough.

2. **Masking logic flavor:** Role-based (simpler) or tag-string-value-based (dynamic, ties masking
   to classification tier)? Could do role-based as primary + brief mention of tag-string pattern.

---

## Article outline (draft)

1. **The Problem** — Alice copies sensitive HR data to a shared team space; colleagues without
   access can see everything. Classic data sprawl governance failure.
2. **Auto-Classification** — Create fake PII data, run SYSTEM$CLASSIFY, show JSON output
   with system tags + user-defined pii tag applied
3. **The Bridge** — System tags can't hold masking policies (verify empirically first);
   the tag map is the connection between classification and enforcement
4. **Tag-Based Masking** — Attach masking policies to pii tag; show role-based masking in source table
5. **Tag Propagation** — Set PROPAGATE = ON_DATA_MOVEMENT; Alice runs CTAS; masking enforced in copy
6. **Verify** — POLICY_REFERENCES and TAG_REFERENCES_ALL_COLUMNS on both tables
7. **Summary** — One tag, one policy set, protection follows the data anywhere it moves

---

## Files in this project

- `CONTEXT.md` — this file
- `demo/01_setup.sql` — governance DB, roles, tags, masking policies, classification profile
- `demo/02_data.sql` — enterprise DB, employees table, fake data
- `demo/03_classify.sql` — run classification, inspect results
- `demo/04_masking.sql` — attach policy to tag, verify masking works
- `demo/05_propagation.sql` — set propagate, CTAS to team space, verify masking in copy
- `demo/06_teardown.sql` — drop everything

(SQL files not yet written — start here in new session after testing the system tag question)
