# AI Units Quota Workflow

A Dynatrace Automation workflow that monitors per-user AI Units consumption and enforces daily quotas by restricting access to AI-powered features when a threshold is exceeded.

This blueprint is adapted from the [Log Query Quota Workflow](https://github.com/Dynatrace/community-examples/tree/main/cost-intelligence-blueprints/set-quotas-with-workflow) in the Dynatrace Community Examples repository.

---

## How It Works

The workflow runs every minute and executes two independent chains:

```
Every minute
    ├── check_for_ai_quota (DQL)            → users at 100%+ of quota, not yet locked out today
    │       ├── lock_out_users (JS)         → adds to "AI Quota Exceeded" group + logs bizevents
    │       │       └── send_admin_lockout_summary (Email) → admin summary of locked-out users
    │       └── send_email_about_ai_quota (Email) → notifies each locked-out user
    │
    └── check_for_quota_warning (DQL)       → users at 80–99% of quota, not yet warned today
            ├── send_quota_warning_email (Email) → warns each approaching-quota user
            └── log_quota_warning_events (JS)    → logs bizevents to prevent duplicate warnings

At 00:00 UTC
    └── quota_reset_at_midnight (JS)        → removes all users from "AI Quota Exceeded" group
```

**Enforcement flow:**

1. `check_for_ai_quota` queries `dt.system.events` billing data to identify users whose AI Units consumption in the past day exceeds the configured threshold (default: 1,000 units).
2. Users already notified today (tracked via a business event) are excluded, preventing duplicate enforcement.
3. Over-quota users are added to the **AI Quota Exceeded** IAM group, which has a deny policy attached to restrict AI feature access.
4. Each affected user receives an email notification.
5. At midnight UTC, all users are removed from the group, resetting their access for the new day.

---

## Prerequisites

- Dynatrace platform with Account Management permissions
- `storage:events:write` permission for the workflow actor (to ingest business events)
- Email connector enabled in your Dynatrace environment
- Billing events data available in Grail (`dt.system.events`)

---

## Before You Deploy: Verify Your Billing Event Schema

AI Units billing events may differ between tenants and product tiers. **Before importing the workflow**, run the following DQL queries in a Dynatrace Notebook to confirm the correct field names for your environment:

**Step 1 — Find the right event type:**
```dql
fetch dt.system.events
| filter event.kind == "BILLING_USAGE_EVENT"
| dedup event.type
| fields event.type
```

**Step 2 — Inspect the AI Units event fields:**
```dql
fetch dt.system.events
| filter event.kind == "BILLING_USAGE_EVENT"
| filter event.type == "AI Units"
| limit 5
```

Look for:
- The exact `event.type` string for AI Units (`"AI Units"`)
- The metric field name (the workflow uses `usage.quantity.billable` — verify this is present in your events)
- Whether `user.email` is present (required for per-user quota enforcement)

Update the DQL query in the `check_for_ai_quota` task accordingly before importing.

---

## Setup (~15 minutes)

### 1. Create an IAM Deny Policy

In **Account Management → Policies**, create a policy that denies access to the AI features you want to restrict. Attach this policy at the **Account** scope.

The specific permissions to deny depend on which AI features your users consume (e.g. Davis CoPilot, AI analysis). Consult your Dynatrace IAM documentation for the relevant permission identifiers.

### 2. Create the "AI Quota Exceeded" Group

In **Account Management → Groups**, create a group named exactly:

```
AI Quota Exceeded
```

Attach the deny policy from Step 1 to this group at the Account scope.

### 3. Create a Service User

Create a dedicated service user (e.g. `ai-quota-check@your-domain.com`) and assign it only the permissions needed to run the quota-check DQL query. Note this user's email — you'll need it as the workflow Actor.

### 4. Create an OAuth Client

In **Account Management → OAuth Clients**, create a client with:

- **Subject**: an Account Manager user (not the service user)
- **Scopes**: `account-idm-read account-idm-write`

> **Important:** The subject user email must exactly match the email shown in Account Management — this may differ from the address you log in with (e.g. it may include a tenant suffix). Check **Account Management → Users** to confirm the exact email address before creating the client. If the subject email doesn't match a valid active user, the OAuth client will show as "Not available" and token requests will fail.

Note the **Client Secret** — it will be used in the next step.

### 5. Store Credentials in the Credential Vault

In your Dynatrace environment, go to **Settings → Credential Vault** and create a new credential:

| Field | Value |
|---|---|
| **Name** | `Quota check OAuth` |
| **Type** | Username & password |
| **Username** | Your Account ID (found in Account Management → Account Info) |
| **Password** | The OAuth Client Secret from Step 4 |
| **Scope** | AppEngine |

The name must be exactly `Quota check OAuth`.

### 6. Configure the Outbound Allowlist

The workflow makes outbound HTTPS calls to `sso.dynatrace.com` (OAuth token) and `api.dynatrace.com` (Account Management API). Verify both hosts are permitted:

1. In your Dynatrace environment, go to **Settings → Preferences → Allowlist for outbound connections**.
2. Confirm `sso.dynatrace.com` and `api.dynatrace.com` are present. If not, add them.

### 7. Import the Workflow

1. In your Dynatrace environment, go to **Automations**.
2. Click **Import workflow** and upload `workflow.yaml`.
3. Open the workflow settings and set the **Actor** to the service user created in Step 3.
4. Go to **Settings → Automations → Authorization settings** and confirm the Actor user has an entry there. If not, add the user and grant the required permissions — without this, the workflow runtime cannot act on the user's behalf and credential vault access will fail.
5. Save and run the workflow once manually to validate all steps.

> **Note on DQL customization:** If you modify the `check_for_ai_quota` DQL query, use only single-line `//` comments within pipeline steps. Block comments (`/* ... */`) are not supported by the Grail DQL parser and will cause a parse error.

---

## Customization

| What to change | Where |
|---|---|
| AI Units threshold (default: 1,000) | `check_for_ai_quota` task → `filter ai_units > 1000` **and** `check_for_quota_warning` task → `filter ai_units <= 1000` and `filter ai_units > 800` (keep both in sync) |
| Warning threshold (default: 80%) | `check_for_quota_warning` task → `filter ai_units > 800` |
| Admin notification email | `send_admin_lockout_summary` task → `to` field (replace `admin@example.com`) |
| Quota reset time (default: 00:00 UTC) | `quota_reset_at_midnight` task → condition `"00:00"` |
| User lockout email subject / body | `send_email_about_ai_quota` task → `subject` / `content` fields |
| Warning email subject / body | `send_quota_warning_email` task → `subject` / `content` fields |
| Restrict to specific users/teams | Add a `filter` on `user.email` in both DQL tasks |
| Change to weekly quota | Adjust the `from:` timeframe in both DQL tasks to `-7d@d` and update the reset task |

---

## Cost Considerations

This workflow is configured to run **every minute** (1,440 executions/day), which consumes Dynatrace Automation execution units on each run. Since AI Units billing data arrives with up to a **24-hour delay**, running every minute provides no enforcement advantage over a much lower frequency — the underlying data does not change within a single day.

**Recommendation:** Increase the schedule interval to reduce execution costs with no meaningful impact on enforcement behaviour:

| Interval | Executions/day | Notes |
|---|---|---|
| Every 1 minute | 1,440 | Default — no benefit over lower frequencies given billing data delay |
| Every 15 minutes | 96 | Good balance for near-real-time notification delivery |
| Every 30 minutes | 48 | Recommended for most deployments |
| Every 60 minutes | 24 | Sufficient given the 24-hour billing data lag |

To change the interval, edit the workflow trigger in **Automations → [workflow] → Settings → Schedule → Interval**.

---

## Known Limitations

- **Billing data delay**: Billing events for AI Units arrive with up to a 24-hour delay. The workflow enforces quotas based on the previous day's consumption.
- **Per-user attribution**: This workflow requires AI Units billing events to include a `user.email` field. If your environment reports AI Units at the environment or application level only, the quota cannot be enforced per user.
- **Duplicate business events**: If the workflow runs multiple times in a minute window when a user is first detected, a small number of duplicate `ai.units.quota.exceeded` business events may be written. This does not affect enforcement.
- **Manual override**: Administrators can unblock a user at any time by removing them from the **AI Quota Exceeded** group directly in Account Management.

---

## Related Resources

- [Log Query Quota Workflow](https://github.com/Dynatrace/community-examples/tree/main/cost-intelligence-blueprints/set-quotas-with-workflow) — the original blueprint this is adapted from
- [Dynatrace Account Management API](https://developer.dynatrace.com/reference/api-reference/account-management/)
- [Dynatrace Automations documentation](https://developer.dynatrace.com/develop/automations/)
- [DQL billing events reference](https://developer.dynatrace.com/reference/system-events/)
