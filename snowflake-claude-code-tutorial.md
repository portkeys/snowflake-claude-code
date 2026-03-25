# How to Use Claude Code with Snowflake CLI: AI-Powered SQL from Your Terminal

Writing SQL queries against a data warehouse is one of those tasks that's ripe for AI assistance. You know what data you want, but the exact syntax — the right table names, join keys, date functions, JSON extraction — slows you down. What if you could just describe what you need in plain English and have an AI write and run the query for you?

In this tutorial, I'll walk you through setting up Snowflake CLI to work with Claude Code so you can query your data warehouse using natural language. We'll go from zero to running queries in about 10 minutes.

## What You'll Build

By the end of this tutorial, you'll be able to open your terminal and have conversations like:

```
You:    How many active O+ subscribers do we have right now?
Claude: [writes the SQL, runs it, returns the answer: 113,477]
```

Claude Code understands your database schema, writes correct Snowflake SQL, runs the queries, and returns the results — all without leaving your terminal.

## Prerequisites

- **Claude Code** installed ([claude.com/claude-code](https://claude.com/claude-code))
- **Homebrew** installed ([brew.sh](https://brew.sh))
- A Snowflake account with credentials (username, password, account identifier)
- Basic familiarity with the terminal

## Part 1: Setting Up Snowflake CLI

### Step 1: Install Snowflake CLI

First, add the Snowflake tap to Homebrew. A "tap" is a third-party repository of Homebrew packages — this tells Homebrew where to find Snowflake's CLI formula:

```bash
brew tap snowflakedb/snowflake-cli
```

Then install the CLI:

```bash
brew install snowflake-cli
```

Verify the installation:

```bash
snow --version
```

You should see something like `Snowflake CLI version: 3.x.x`.

> **Reference:** [Snowflake CLI Installation Guide](https://docs.snowflake.com/en/developer-guide/snowflake-cli/installation/installation#label-snowcli-install-homebrew)

### Step 2: Add Your Connection

Run this command, replacing the values with your own Snowflake account details:

```bash
snow connection add \
  --connection-name my_connection \
  --account "YOUR_ACCOUNT_ID" \
  --user "YOUR_USERNAME" \
  --authenticator "username_password_mfa" \
  --role "YOUR_ROLE" \
  --warehouse "YOUR_WAREHOUSE" \
  --default \
  --no-interactive
```

This saves the connection to `~/.snowflake/config.toml`.

> **Reference:** [Configure CLI](https://docs.snowflake.com/en/developer-guide/snowflake-cli/connecting/configure-cli) | [Configure Connections](https://docs.snowflake.com/en/developer-guide/snowflake-cli/connecting/configure-connections)

**Authenticator options:**
- `username_password_mfa` — Use this if your account uses password + MFA (Duo, Okta Verify, etc.)
- `externalbrowser` — Use this if your account uses SAML SSO (Okta, Azure AD, etc.)
- `snowflake` — Use this for username/password only (no MFA)

### Step 3: Set Your Password

The password should be provided via environment variable — don't add it to the config file.

**Per session (most secure):**

```bash
export SNOWFLAKE_PASSWORD='your_password_here'
```

**Persistent (add to `~/.zshrc`):**

Open `~/.zshrc` in your editor and add:

```bash
export SNOWFLAKE_PASSWORD='your_password_here'
```

Then reload: `source ~/.zshrc`

> **Tip:** Don't use `echo` to append the password — the `!` character in passwords triggers zsh history expansion and will cause errors. Edit the file directly.

### Step 4: Test the Connection

```bash
snow connection test
```

If you're using MFA, approve the push notification on your phone. You should see a success message.

### Step 5: Fix the MFA Problem (Important!)

If your account uses MFA, you'll get a Duo/Okta push for **every single query** by default. This makes the CLI nearly unusable.

The fix: ask your Snowflake admin to enable MFA token caching:

```sql
-- Admin runs this (requires ACCOUNTADMIN role):
ALTER ACCOUNT SET ALLOW_CLIENT_MFA_CACHING = TRUE;
```

With this enabled, you approve MFA once and then get **4 hours of push-free queries**. This is the single most important step for a good CLI experience.

**How to check if it's already enabled:**

```bash
snow sql -q "SHOW PARAMETERS LIKE 'ALLOW_CLIENT_MFA_CACHING' IN ACCOUNT" --format json
```

**Alternative: Key-pair authentication (zero MFA, ever):**

If you want to bypass MFA entirely for CLI usage, set up RSA key-pair auth:

```bash
# Generate key pair
mkdir -p ~/.snowflake
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out ~/.snowflake/rsa_key.p8 -nocrypt
openssl rsa -in ~/.snowflake/rsa_key.p8 -pubout -out ~/.snowflake/rsa_key.pub
chmod 600 ~/.snowflake/rsa_key.p8

# Get the public key value to send to your admin
grep -v "BEGIN\|END" ~/.snowflake/rsa_key.pub | tr -d '\n'
```

Your admin registers it:

```sql
ALTER USER YOUR_USERNAME SET RSA_PUBLIC_KEY='<the public key>';
```

Then update your config to use key-pair auth:

```toml
[connections.my_connection]
account = "YOUR_ACCOUNT_ID"
user = "YOUR_USERNAME"
warehouse = "YOUR_WAREHOUSE"
role = "YOUR_ROLE"
authenticator = "snowflake"
private_key_path = "/Users/you/.snowflake/rsa_key.p8"
```

## Part 2: Connecting Claude Code to Snowflake

### Step 1: Create a Project Directory

```bash
mkdir -p ~/Projects/snowflake
cd ~/Projects/snowflake
```

### Step 2: Create the CLAUDE.md File

This is the key step. `CLAUDE.md` is a file that Claude Code reads automatically when you start a session in the directory. It tells Claude about your database schema, connection details, and how to run queries.

Create a file called `CLAUDE.md` in your project directory:

```bash
touch CLAUDE.md
```

Start with just the connection info and instructions for how to run queries:

```markdown
# Snowflake Project

## Connection

- **Connection name:** `my_connection`
- **Warehouse:** `MY_WAREHOUSE`
- **Role:** `MY_ROLE`

### Running Queries

Use the Snowflake CLI to run queries:

    snow sql -c my_connection -q "SELECT ..." --format json
```

### Step 3: Let Claude Explore Your Schema

Here's where it gets interesting. Instead of manually documenting every table, **ask Claude to explore the schema for you**. Give it a few example queries you typically run, and let it discover the table structures.

Start Claude Code in your project directory:

```bash
cd ~/Projects/snowflake
claude
```

Then have a conversation like this:

```
You:  Here are some tables I use regularly:
      - prd_datamart.core.subscription_metrics (subscription data)
      - amplitude_core.schema_296225.events_296225 (event analytics)
      - prd_rivt.ods_public.user_profile (user profiles)
      - prd_rivt.ods_public.auth_user (user auth/email)

      Can you explore the schema of these tables and document them in CLAUDE.md?
      Here's a query I typically run to find active O+ subscribers:

      SELECT DISTINCT sm.uuid
      FROM prd_datamart.core.subscription_metrics sm
      WHERE sm.product_group = 'O+'
        AND sm.subscription_end_date::date >= CURRENT_DATE()
```

Claude will then run queries like `DESCRIBE TABLE ...` and `SELECT * FROM ... LIMIT 5` to understand the column names, types, and relationships. It documents everything in your CLAUDE.md — table schemas, join keys, data types, and common patterns.

The result is a comprehensive schema reference that Claude uses in every future conversation. You invest a few minutes upfront, and every query after that benefits from the context.

**Tip:** The more example queries and context you give Claude during this exploration phase, the better. Mention things like:
- "USER_ID in Amplitude events is actually a UUID that joins to USER_PROFILE.UUID"
- "EVENT_PROPERTIES is a JSON column — use `:` notation to extract fields"
- "We have two Amplitude schemas (296225 and 426461) — always UNION both"

These nuances get documented and used correctly in future queries.

### Step 4: Enable Claude Code to Run Queries Directly

Claude Code runs commands in a sandboxed environment by default. This sandbox can't access your Snowflake config file (`~/.snowflake/config.toml`) or the `SNOWFLAKE_PASSWORD` environment variable, so `snow sql` commands will fail inside the sandbox.

The fix: tell Claude to run `snow` commands with the sandbox disabled. Add this to your CLAUDE.md:

```markdown
### Running Queries

Run queries using the Snowflake CLI with sandbox disabled
(`dangerouslyDisableSandbox: true` on the Bash tool), since the
sandbox cannot access the Snowflake config file or password:

    snow sql -c my_connection -q "SELECT ..." --format json
```

When Claude runs a query with the sandbox disabled, it will prompt you for permission the first time. Approve it, and subsequent queries in the same session run automatically. This is what makes the workflow seamless — Claude writes the SQL, runs it, and shows you the results in one step.

> **Note:** If you prefer more control, you can skip the sandbox-disable instruction and instead run queries yourself using the `!` prefix, which executes commands in your shell session directly:
> ```
> ! snow sql -c my_connection -q "SELECT ..." --format json
> ```

## Part 3: Using It — Live Demos

Now for the fun part. Here are real examples using actual data.

### Demo 1: Simple Data Exploration

**You:** How many active O+ subscribers do we have right now?

**Claude runs:**

```sql
SELECT COUNT(DISTINCT uuid) AS active_oplus_subscribers
FROM prd_datamart.core.subscription_metrics
WHERE product_group = 'O+'
  AND subscription_end_date::date >= CURRENT_DATE()
```

**Result:**

```json
[{"ACTIVE_OPLUS_SUBSCRIBERS": 113477}]
```

Claude knows the table name, the column to filter on, and the date logic — all from the CLAUDE.md context.

### Demo 2: Cross-Table Analysis

**You:** Show me the top 10 pages viewed by O+ subscribers in the last 7 days.

**Claude runs:**

```sql
WITH oplus AS (
    SELECT DISTINCT uuid
    FROM prd_datamart.core.subscription_metrics
    WHERE product_group = 'O+'
      AND subscription_end_date::date >= CURRENT_DATE()
),
page_views AS (
    SELECT A.USER_ID, A.EVENT_PROPERTIES:pageUrl::VARCHAR AS page_url
    FROM AMPLITUDE_CORE.SCHEMA_296225.EVENTS_296225 A
    WHERE A.EVENT_TYPE = 'Loaded a Page'
      AND A.USER_ID IS NOT NULL
      AND A.EVENT_TIME >= DATEADD('day', -7, CURRENT_TIMESTAMP())
    UNION ALL
    SELECT A.USER_ID, A.EVENT_PROPERTIES:pageUrl::VARCHAR AS page_url
    FROM AMPLITUDE_CORE.SCHEMA_426461.EVENTS_426461 A
    WHERE A.EVENT_TYPE = 'Loaded a Page'
      AND A.USER_ID IS NOT NULL
      AND A.EVENT_TIME >= DATEADD('day', -7, CURRENT_TIMESTAMP())
)
SELECT pv.page_url, COUNT(*) AS view_count,
       COUNT(DISTINCT pv.USER_ID) AS unique_users
FROM page_views pv
INNER JOIN oplus o ON pv.USER_ID = o.uuid
GROUP BY pv.page_url
ORDER BY view_count DESC
LIMIT 10
```

**Result:**

```json
[
  {"PAGE_URL": "https://www.pinkbike.com/", "VIEW_COUNT": 19446, "UNIQUE_USERS": 1351},
  {"PAGE_URL": "https://velo.outsideonline.com/", "VIEW_COUNT": 12973, "UNIQUE_USERS": 1580},
  {"PAGE_URL": "https://www.pinkbike.com/buysell/", "VIEW_COUNT": 5228, "UNIQUE_USERS": 819},
  {"PAGE_URL": "https://watch.outsideonline.com/live-events/usss", "VIEW_COUNT": 3644, "UNIQUE_USERS": 367},
  {"PAGE_URL": "https://www.outsideonline.com/", "VIEW_COUNT": 3342, "UNIQUE_USERS": 1025}
]
```

This is where the schema documentation really pays off. Claude handles the JSON extraction syntax (`EVENT_PROPERTIES:pageUrl::VARCHAR`), remembers to UNION both Amplitude schemas, and applies the correct subscriber filter — all automatically.

### Demo 3: Breakdown by Dimension

**You:** Break down active subscribers by payment processor.

**Claude runs:**

```sql
SELECT payment_processor, COUNT(DISTINCT uuid) AS active_subscribers
FROM prd_datamart.core.subscription_metrics
WHERE product_group = 'O+'
  AND subscription_end_date::date >= CURRENT_DATE()
GROUP BY payment_processor
ORDER BY active_subscribers DESC
```

**Result:**

```json
[
  {"PAYMENT_PROCESSOR": "stripe", "ACTIVE_SUBSCRIBERS": 60688},
  {"PAYMENT_PROCESSOR": "piano", "ACTIVE_SUBSCRIBERS": 17624},
  {"PAYMENT_PROCESSOR": "apple", "ACTIVE_SUBSCRIBERS": 12566},
  {"PAYMENT_PROCESSOR": "appstore", "ACTIVE_SUBSCRIBERS": 11500},
  {"PAYMENT_PROCESSOR": "google", "ACTIVE_SUBSCRIBERS": 5637},
  {"PAYMENT_PROCESSOR": "braintree", "ACTIVE_SUBSCRIBERS": 1629},
  {"PAYMENT_PROCESSOR": "itunes", "ACTIVE_SUBSCRIBERS": 1083},
  {"PAYMENT_PROCESSOR": "roku", "ACTIVE_SUBSCRIBERS": 841}
]
```

### Demo 4: Time Series Trend

**You:** Show me new O+ subscriber signups per month for the last 6 months.

**Claude runs:**

```sql
SELECT DATE_TRUNC('month', subscription_start_date)::date AS month,
       COUNT(DISTINCT uuid) AS new_subscribers
FROM prd_datamart.core.subscription_metrics
WHERE product_group = 'O+'
  AND new = 1
  AND subscription_start_date >= DATEADD('month', -6, CURRENT_TIMESTAMP())
GROUP BY month
ORDER BY month
```

**Result:**

```json
[
  {"MONTH": "2025-10-01", "NEW_SUBSCRIBERS": 2954},
  {"MONTH": "2025-11-01", "NEW_SUBSCRIBERS": 2776},
  {"MONTH": "2025-12-01", "NEW_SUBSCRIBERS": 2666},
  {"MONTH": "2026-01-01", "NEW_SUBSCRIBERS": 2580},
  {"MONTH": "2026-02-01", "NEW_SUBSCRIBERS": 2052},
  {"MONTH": "2026-03-01", "NEW_SUBSCRIBERS": 1828}
]
```

Because Claude Code maintains conversation context, you can follow up naturally: "Can you visualize that as a chart?" or "What's the month-over-month change rate?"

### Demo 5: Multi-Table User Lookup

**You:** Show me the 5 most recent O+ subscribers with their email, signup source, and payment method.

**Claude runs:**

```sql
SELECT au.email, up.signup_source,
       up.created_at::date AS signup_date,
       sm.subscription_plan_name,
       sm.subscription_start_date::date AS sub_start,
       sm.payment_processor
FROM prd_datamart.core.subscription_metrics sm
JOIN prd_rivt.ods_public.user_profile up ON sm.uuid = up.uuid
JOIN prd_rivt.ods_public.auth_user au ON up.user_id = au.id
WHERE sm.product_group = 'O+'
  AND sm.subscription_end_date::date >= CURRENT_DATE()
ORDER BY sm.subscription_start_date DESC
LIMIT 5
```

**Result:**

```json
[
  {"EMAIL": "user1@gmail.com", "SIGNUP_SOURCE": "TRAILFORKS", "SIGNUP_DATE": "2026-03-10",
   "SUBSCRIPTION_PLAN_NAME": "Outside+ (O+)", "SUB_START": "2026-03-24", "PAYMENT_PROCESSOR": "apple"},
  {"EMAIL": "user2@me.com", "SIGNUP_SOURCE": "TRAILFORKS", "SIGNUP_DATE": "2022-01-21",
   "SUBSCRIPTION_PLAN_NAME": "Outside+ (O+)", "SUB_START": "2026-03-24", "PAYMENT_PROCESSOR": "apple"}
]
```

Claude joins across three tables (subscription metrics, user profile, auth user) using the correct join keys — because the CLAUDE.md documents the relationships.

## Tips for Getting the Most Out of This Setup

1. **Invest in your CLAUDE.md.** The more schema detail and example patterns you include, the better the queries will be. Think of it as onboarding a new data analyst — what would they need to know?

2. **Include common gotchas.** If certain columns need special handling (JSON extraction, timezone conversion, null filtering), document it. Claude will follow the patterns you show.

3. **Save query results to files.** For large result sets, pipe output to JSON files that Claude can then analyze:
   ```
   snow sql -c my_connection -q "SELECT ..." --format json > results.json
   ```
   Then ask Claude to summarize, visualize, or further analyze the data.

4. **Use conversation context.** Don't start every request from scratch. Build on previous queries in the same conversation — "now filter that to just US users" or "add a month-over-month comparison."

5. **Let Claude explain the data.** After running a query, ask "what does this tell us?" or "are there any anomalies here?" Claude can interpret the results in context.

## Troubleshooting

### "Connection not configured" error

The Snowflake CLI looks for config at `~/.snowflake/config.toml`. If your config is elsewhere (e.g., `~/Library/Application Support/snowflake/config.toml`), either:
- Move it to `~/.snowflake/config.toml`
- Or specify the path explicitly: `snow sql --config-file "/path/to/config.toml" ...`

### MFA push on every query

Your admin needs to enable MFA token caching:
```sql
ALTER ACCOUNT SET ALLOW_CLIENT_MFA_CACHING = TRUE;
```
Without this, the server doesn't return a cacheable token regardless of client settings.

### Password not found / empty password error

Make sure `SNOWFLAKE_PASSWORD` is exported in the shell session where you run the query. The env var must be set in your shell profile (`~/.zshrc`).

### "Insufficient privileges" errors

Check your role has access to the tables you're querying:
```bash
snow sql -q "SELECT CURRENT_ROLE()" --format json
```

## What's Next

Once you're comfortable with this workflow, consider:

- **Adding more tables** to CLAUDE.md as you discover what you query most often
- **Creating saved query templates** in your project directory for recurring analyses
- **Piping results into Python scripts** for visualization (Claude can write those too)
- **Setting up key-pair auth** for a completely MFA-free CLI experience

The combination of Claude Code's natural language understanding and Snowflake CLI's direct database access turns your terminal into a conversational data analysis environment. Instead of context-switching between a SQL editor, documentation, and chat, everything happens in one place.
