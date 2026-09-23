# AI Units Quota Workflow

A Dynatrace Automation workflow that monitors per-user AI Units consumption and enforces daily quotas by restricting access to AI-powered features when a threshold is exceeded.

This blueprint is adapted from the [Log Query Quota Workflow](https://github.com/Dynatrace/community-examples/tree/main/cost-intelligence-blueprints/set-quotas-with-workflow) in the Dynatrace Community Examples repository.

---

## How It Works

The workflow runs every minute and executes two independent chains:

```
Every minute
    └── check_for_ai_quota (DQL)
            ├── lock_out_users (JS)     → adds user to "AI Quota Exceeded" group + logs business event
            └── send_email_about_ai_quota (Email) → notifies affected user

At 00:00 UTC
    └── quota_reset_at_midnight (JS)    → removes all users from "AI Quota Exceeded" group
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
- The metric field name (the workflow assumes `consumed_units` — adjust if different)
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

### 6. Import the Workflow

1. In your Dynatrace environment, go to **Automations**.
2. Click **Import workflow** and upload `workflow.yaml`.
3. Set the workflow **Actor** to the service user created in Step 3.
4. Save and run the workflow once manually to validate all steps.

---

## Customization

| What to change | Where |
|---|---|
| AI Units threshold (default: 1,000) | `check_for_ai_quota` task → DQL `filter ai_units > 1000` |
| Quota reset time (default: 00:00 UTC) | `quota_reset_at_midnight` task → condition `"00:00"` |
| Email subject / body | `send_email_about_ai_quota` task → `subject` / `content` fields |
| Restrict to specific users/teams | Add a `filter` on `user.email` in the DQL query |
| Change to weekly quota | Adjust the `from:` timeframe in the DQL to `-7d@d` and update the reset task |

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
