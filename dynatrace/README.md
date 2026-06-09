# dynatrace/ — Dynatrace Configuration as Code

Declarative Dynatrace configuration applied via
[dtctl](https://github.com/dynatrace-oss/dtctl). This is the Dynatrace-side
counterpart to `aap_config/` (AAP side).

## Prerequisites

1. **Install dtctl:** see [dtctl releases](https://github.com/dynatrace-oss/dtctl/releases)
2. **Create a context and authenticate:**
   ```bash
   dtctl config set-context aap-push --environment https://<env-id>.apps.dynatrace.com
   dtctl auth login
   ```
   This opens a browser for OAuth login. Tokens are stored in your OS keyring.
3. **Install the Red Hat Ansible Connector** from Dynatrace Hub (one-time):
   Hub > search "Red Hat Ansible" > Install

## Files

| File | Purpose |
|------|---------|
| `eda-connection.yaml` | Reference: EDA connection schema + value structure |
| `workflow-service-failure.json` | Workflow JSON for `dtctl create workflow` |
| `workflow-service-failure.yaml` | Reference docs for the workflow (trigger config, event shape) |

## Setup flow

The connection and workflow depend on values from the AAP side (Event Stream
UUID, connection objectId), so the setup order is:

1. **AAP first:** `aap_config/load.yml` creates the Event Stream
2. **Get the Event Stream URL** from AAP (see `eda-connection.yaml` comments)
3. **Create the EDA connection** in Dynatrace UI:
   Settings > Connections > Red Hat Ansible > Event-Driven Ansible > + Connection
4. **Get the connection objectId:**
   ```bash
   dtctl get settings --schema app:dynatrace.redhat.ansible:eda-webhook.connection
   ```
5. **Update `workflow-service-failure.json`** with the `connectionId`
6. **Create the workflow:**
   ```bash
   dtctl create workflow --file dynatrace/workflow-service-failure.json
   ```

> **Note:** The workflow trigger uses `analysisReady: false` — it fires
> immediately when Davis opens a problem, without waiting for root cause
> analysis. This cuts detection time from ~6 min to ~2-3 min. Root cause data
> is queried separately via the Problems API v2 after remediation completes.

## Classic Access Token (Problems API)

dtctl's OAuth platform token (`dt0s16.*`) does **not** expose
`environment-api:problems:*` scopes. To list or close problems (e.g. stale
"Process unavailable" from decommissioned hosts) you need a separate classic
access token:

1. In Dynatrace: **Settings > Access tokens > Generate new token**
2. Name: `dtctl-problems`
3. Scopes: `problems.read`, `problems.write`, `securityProblems.read`,
   `securityProblems.write`
4. Save the `dt0c01.*` value to `DT_API_TOKEN` in `docs/dev-environment.sh`

```bash
source docs/dev-environment.sh

# List open problems
curl -s "${DT_API_HOST}/api/v2/problems?problemSelector=status(open)" \
  -H "Authorization: Api-Token ${DT_API_TOKEN}" | python3 -m json.tool

# Close a stale problem
curl -s -X POST "${DT_API_HOST}/api/v2/problems/<problemId>/close" \
  -H "Authorization: Api-Token ${DT_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"message":"Host decommissioned"}'
```

## Verified schemas

| Schema | Purpose | Verified |
|--------|---------|----------|
| `app:dynatrace.redhat.ansible:eda-webhook.connection` | EDA Event Stream connections | 2026-06-05 |
| `app:dynatrace.redhat.ansible:automation-controller.connection` | Controller connections (not used in push) | 2026-06-05 |
| `builtin:availability.process-group-alerting` | Process group availability rules | 2026-06-07 |

## Testing

Test the full push path by POSTing directly to the Event Stream endpoint:

```bash
source docs/dev-environment.sh
curl -sk -X POST \
  "https://<aap-host>/eda-event-streams/api/eda/v1/external_event_stream/<uuid>/post/" \
  -H "Authorization: Bearer ${EDA_EVENT_STREAM_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"event.name": "Test push", "event.status": "ACTIVE", "source": "manual-test"}'
```

Then check AAP Controller for a `DT-EDA-PUSH - Notify` job.

## dtctl quick reference

```bash
dtctl doctor                              # verify auth + connectivity
dtctl get workflows                       # list workflows
dtctl describe workflow <id>              # workflow details
dtctl exec workflow <id>                  # manual trigger (needs real event context)
dtctl get wfe --workflow <id>             # list executions
dtctl logs wfe <execution-id>             # execution logs
dtctl get settings --schema <schema>      # list settings objects
dtctl query 'fetch dt.entity.process_group'           # find process groups + IDs
dtctl query 'fetch dt.entity.host | fields entity.name, id, state'  # list hosts
```

### Problem management (uses DT_API_TOKEN, not dtctl)

```bash
source docs/dev-environment.sh
# List open problems
curl -s "${DT_API_HOST}/api/v2/problems?problemSelector=status(open)" \
  -H "Authorization: Api-Token ${DT_API_TOKEN}" | python3 -m json.tool
# Close a problem
curl -s -X POST "${DT_API_HOST}/api/v2/problems/<problemId>/close" \
  -H "Authorization: Api-Token ${DT_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"message":"Host decommissioned"}'
```
