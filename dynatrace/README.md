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

## Verified schemas (2026-06-05)

| Schema | Purpose |
|--------|---------|
| `app:dynatrace.redhat.ansible:eda-webhook.connection` | EDA Event Stream connections |
| `app:dynatrace.redhat.ansible:automation-controller.connection` | Controller connections (not used in push) |

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
```
