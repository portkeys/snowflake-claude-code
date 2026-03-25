# Snowflake Project

## Connection

- **Connection name:** `outside_snowflake`
- **Account:** `LHUMEZT-WHA29295`
- **User:** `WYANG`
- **Role:** `ROLE_ANALYST`
- **Warehouse:** `ANALYST_WH`
- **Auth:** `username_password_mfa` (MFA token caching enabled — Duo push once per 4 hours)

### Running Queries

Run queries using the Snowflake CLI with sandbox disabled (`dangerouslyDisableSandbox: true` on the Bash tool), since the sandbox cannot access the Snowflake config file or password:

```bash
snow sql -c outside_snowflake -q "SELECT ..." --format json

# For large results, pipe to a file:
snow sql -c outside_snowflake -q "SELECT ..." --format json > output.json
```

MFA token caching is enabled — the first query triggers a Duo push, then subsequent queries are MFA-free for up to 4 hours.

---

## Databases & Key Tables

### 1. AMPLITUDE_CORE — Event Analytics (Amplitude)

Two schemas with identical structure representing different Amplitude projects:
- `AMPLITUDE_CORE.SCHEMA_296225.EVENTS_296225`
- `AMPLITUDE_CORE.SCHEMA_426461.EVENTS_426461`

**Key columns for queries:**

| Column | Type | Usage |
|--------|------|-------|
| USER_ID | TEXT | RIVT UUID — join key to USER_PROFILE.UUID |
| EVENT_TYPE | TEXT | e.g. `'Loaded a Page'` |
| EVENT_TIME | TIMESTAMP_NTZ | When the event occurred |
| EVENT_PROPERTIES | VARIANT (JSON) | Semi-structured; access via `:` notation, e.g. `EVENT_PROPERTIES:pageUrl::VARCHAR` |
| USER_PROPERTIES | VARIANT (JSON) | Semi-structured user attributes at event time |
| DEVICE_TYPE | TEXT | Device type |
| PLATFORM | TEXT | Platform |
| COUNTRY / REGION / CITY | TEXT | Geo data |
| SESSION_ID | NUMBER | Session identifier |
| AMPLITUDE_ID | NUMBER | Amplitude's internal user ID |
| DEVICE_ID | TEXT | Device identifier |

**Other notable columns:** ADID, IDFA (ad tracking), IP_ADDRESS, OS_NAME, OS_VERSION, DEVICE_BRAND/MODEL/MANUFACTURER, LANGUAGE, LIBRARY, LOCATION_LAT/LNG, DATA (VARIANT), GLOBAL_USER_PROPERTIES (VARIANT), GROUP_PROPERTIES (VARIANT), GROUPS (VARIANT), PLAN (VARIANT), PAYING, IS_ATTRIBUTION_EVENT, FOLLOWED_AN_IDENTIFY.

### 2. PRD_DATAMART.CORE.SUBSCRIPTION_METRICS — Subscription Data

**Key columns for queries:**

| Column | Type | Usage |
|--------|------|-------|
| UUID | TEXT | User UUID — join key to USER_PROFILE.UUID |
| PRODUCT_GROUP | TEXT | e.g. `'O+'` for Outside Plus |
| SUBSCRIPTION_END_DATE | TIMESTAMP_NTZ | Filter active subs: `>= CURRENT_DATE()` |
| SUBSCRIPTION_START_DATE | TIMESTAMP_NTZ | When subscription started |
| SUBSCRIPTION_PLAN_NAME | TEXT | Plan name |
| PRODUCT_CODE | TEXT | Product code |
| AMOUNT_PAID | FLOAT | Payment amount |
| PAYMENT_PROCESSOR | TEXT | e.g. Stripe, Apple |
| PAYMENT_DEVICE_TYPE | TEXT | Device used for payment |
| PURCHASE_SOURCE | TEXT | Where the purchase originated |
| SIGNUP_SOURCE | TEXT | Signup source |
| BILLING_REASON | TEXT | Reason for billing event |
| COUPON_ID | TEXT | Coupon used |
| NEW | NUMBER | Flag: new subscription |
| RENEWED | NUMBER | Flag: renewal |
| CHURNED | NUMBER | Flag: churned |
| WINBACK | NUMBER | Flag: win-back |
| CANCELED_AT | TIMESTAMP_NTZ | Cancellation timestamp |
| ACTION_DATE | TIMESTAMP_NTZ | Date of subscription action |
| SUBSCRIPTION_ID | NUMBER | Subscription ID |
| SUBSCRIPTION_ORDER | NUMBER | Order number |
| SOURCE_SYSTEM_CODE | TEXT | Source system |
| SOURCE_SYSTEM_USER_ID | TEXT | User ID in source system |
| OPLUS_MIGRATE_FLAG | NUMBER | O+ migration flag |
| OPLUS_BENEFITS_MIGRATED_ON | TIMESTAMP_TZ | Migration timestamp |
| POST_MIGRATION_PRODUCT_GROUP | TEXT | Product group after migration |

### 3. PRD_RIVT.ODS_PUBLIC.USER_PROFILE — User Profiles

**Key columns for queries:**

| Column | Type | Usage |
|--------|------|-------|
| UUID | TEXT | **Primary join key** — links to Amplitude USER_ID and SUBSCRIPTION_METRICS.UUID |
| USER_ID | NUMBER | Foreign key to AUTH_USER.ID |
| ID | NUMBER (PK) | Internal profile ID (NOT NULL) |
| ACTIVATED | BOOLEAN | Account activated |
| SIGNUP_SOURCE | TEXT | How user signed up |
| CREATED_AT | TIMESTAMP_TZ | Profile creation time |
| CITY / STATE_CODE / COUNTRY_CODE / ZIP_CODE | TEXT | Location |
| GENDER | TEXT | Gender |
| BIRTHDAY | DATE | Date of birth |
| PHONE | TEXT | Phone number |
| NEWSLETTER_SUBSCRIBED | BOOLEAN | Newsletter opt-in |
| STRIPE_CUSTOMER_ID | TEXT | Stripe ID |
| CORDIAL_CUSTOMER_ID | TEXT | Cordial (email platform) ID |
| HUBSPOT_CONTACT_ID | TEXT | HubSpot CRM ID |
| TEST_ACCOUNT | BOOLEAN | Flag for test accounts |
| DUPLICATE | BOOLEAN | Duplicate account flag |

### 4. PRD_RIVT.ODS_PUBLIC.AUTH_USER — User Authentication

**Key columns for queries:**

| Column | Type | Usage |
|--------|------|-------|
| ID | NUMBER (PK) | **Primary key** — links to USER_PROFILE.USER_ID |
| EMAIL | TEXT | User's email address |
| USERNAME | TEXT | Username |
| FIRST_NAME / LAST_NAME | TEXT | Name |
| IS_ACTIVE | BOOLEAN | Active account flag |
| DATE_JOINED | TIMESTAMP_TZ | When user joined |
| LAST_LOGIN | TIMESTAMP_TZ | Last login time |
| IS_STAFF / IS_SUPERUSER | BOOLEAN | Admin flags |

---

## Table Relationships

```
AMPLITUDE EVENTS (USER_ID = UUID)
        |
        v
USER_PROFILE (UUID) <-----> SUBSCRIPTION_METRICS (UUID)
        |
        | USER_ID = ID
        v
    AUTH_USER (ID) --> EMAIL
```

- **Amplitude → User Profile:** `EVENTS.USER_ID = USER_PROFILE.UUID`
- **User Profile → Auth User:** `USER_PROFILE.USER_ID = AUTH_USER.ID`
- **Subscription Metrics → User Profile:** `SUBSCRIPTION_METRICS.UUID = USER_PROFILE.UUID`

---

## Common SQL Patterns

### Pattern: Identify O+ subscribers who viewed specific content

```sql
-- 1. Find events matching criteria from BOTH Amplitude schemas
WITH matched_events AS (
    SELECT DISTINCT A.USER_ID AS rivt_id
    FROM AMPLITUDE_CORE.SCHEMA_296225.EVENTS_296225 A
    WHERE A.EVENT_TYPE = 'Loaded a Page'
      AND A.USER_ID IS NOT NULL
      AND A.EVENT_TIME >= DATEADD('day', -90, CURRENT_TIMESTAMP())
      AND A.EVENT_PROPERTIES:pageUrl::VARCHAR ILIKE '%/adventure-travel/%'
    UNION
    SELECT DISTINCT A.USER_ID AS rivt_id
    FROM AMPLITUDE_CORE.SCHEMA_426461.EVENTS_426461 A
    WHERE A.EVENT_TYPE = 'Loaded a Page'
      AND A.USER_ID IS NOT NULL
      AND A.EVENT_TIME >= DATEADD('day', -90, CURRENT_TIMESTAMP())
      AND A.EVENT_PROPERTIES:pageUrl::VARCHAR ILIKE '%/adventure-travel/%'
),
-- 2. Get active O+ subscribers
oplus_subscribers AS (
    SELECT DISTINCT sm.uuid
    FROM prd_datamart.core.subscription_metrics sm
    WHERE sm.product_group = 'O+'
      AND sm.subscription_end_date::date >= CURRENT_DATE()
      AND sm.uuid IS NOT NULL
)
-- 3. Join to get user details and O+ status
SELECT
    U.UUID AS rivt_id, AU.EMAIL,
    CASE WHEN op.uuid IS NOT NULL THEN TRUE ELSE FALSE END AS is_oplus
FROM matched_events ME
    INNER JOIN PRD_RIVT.ODS_PUBLIC.USER_PROFILE U ON U.UUID = ME.rivt_id
    INNER JOIN PRD_RIVT.ODS_PUBLIC.AUTH_USER AU ON U.USER_ID = AU.ID
    LEFT JOIN oplus_subscribers op ON U.UUID = op.uuid
WHERE is_oplus = TRUE;
```

### Pattern: Count unique users by time window with O+ breakdown

```sql
WITH time_windows AS (
    SELECT 30 AS window_days UNION ALL SELECT 60 UNION ALL SELECT 90
),
matched_events AS (
    -- UNION both Amplitude schemas, include EVENT_TIME for windowing
    SELECT DISTINCT A.USER_ID AS rivt_id, A.EVENT_TIME
    FROM AMPLITUDE_CORE.SCHEMA_296225.EVENTS_296225 A
    WHERE A.EVENT_TYPE = 'Loaded a Page'
      AND A.USER_ID IS NOT NULL
      AND A.EVENT_TIME >= DATEADD('day', -90, CURRENT_TIMESTAMP())
      AND A.EVENT_PROPERTIES:pageUrl::VARCHAR ILIKE '%/adventure-travel/%'
    UNION
    SELECT DISTINCT A.USER_ID AS rivt_id, A.EVENT_TIME
    FROM AMPLITUDE_CORE.SCHEMA_426461.EVENTS_426461 A
    WHERE A.EVENT_TYPE = 'Loaded a Page'
      AND A.USER_ID IS NOT NULL
      AND A.EVENT_TIME >= DATEADD('day', -90, CURRENT_TIMESTAMP())
      AND A.EVENT_PROPERTIES:pageUrl::VARCHAR ILIKE '%/adventure-travel/%'
),
oplus_subscribers AS (
    SELECT DISTINCT sm.uuid
    FROM prd_datamart.core.subscription_metrics sm
    WHERE sm.product_group = 'O+'
      AND sm.subscription_end_date::date >= CURRENT_DATE()
      AND sm.uuid IS NOT NULL
)
SELECT
    tw.window_days || ' days' AS time_window,
    COUNT(DISTINCT me.rivt_id) AS unique_registered_users,
    COUNT(DISTINCT CASE WHEN op.uuid IS NOT NULL THEN me.rivt_id END) AS oplus_users,
    COUNT(DISTINCT CASE WHEN op.uuid IS NULL THEN me.rivt_id END) AS non_oplus_users
FROM time_windows tw
    LEFT JOIN matched_events me
        ON me.EVENT_TIME >= DATEADD('day', -tw.window_days, CURRENT_TIMESTAMP())
    LEFT JOIN oplus_subscribers op ON me.rivt_id = op.uuid
GROUP BY tw.window_days
ORDER BY tw.window_days;
```

### Key Query Notes

- **Always UNION both Amplitude schemas** (296225 and 426461) to capture all events
- **EVENT_PROPERTIES is VARIANT** — use `:` notation to extract fields: `EVENT_PROPERTIES:pageUrl::VARCHAR`
- **ILIKE** for case-insensitive URL matching
- **O+ active subscriber filter:** `product_group = 'O+' AND subscription_end_date::date >= CURRENT_DATE()`
- **RIVT UUID** is the universal user identifier across all tables
