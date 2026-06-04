# dynatrace/ — Dynatrace Configuration as Code

Declarative Dynatrace configuration applied via
[dtctl](https://github.com/dynatrace-oss/dtctl). This is the Dynatrace-side
counterpart to `aap_config/` (AAP side).

## Prerequisites

1. **Install dtctl:** see [dtctl releases](https://github.com/dynatrace-oss/dtctl/releases)
2. **Authenticate:** `dtctl login` (OAuth recommended) or set env vars:
   ```bash
   export DT_OAUTH_CLIENT_ID="..."
   export DT_OAUTH_CLIENT_SECRET="..."
   export DT_OAUTH_SSO_ENDPOINT="https://sso.dynatrace.com/sso/oauth2/token"
   export DT_ENV_URL="https://<env-id>.apps.dynatrace.com"
   ```
3. **Install the Red Hat Ansible Connector** from Dynatrace Hub (manual, one-time):
   Hub > Red Hat Ansible > Install

## Files

| File | Purpose |
|------|---------|
| `eda-connection.yaml` | EDA connection pointing Dynatrace at the AAP Event Stream endpoint |
| `workflow-service-failure.yaml` | Workflow: problem trigger → "Send event to EDA" action |

## Apply

```bash
source docs/dev-environment.sh
dtctl apply -f dynatrace/eda-connection.yaml
dtctl apply -f dynatrace/workflow-service-failure.yaml
```

Or apply everything at once:

```bash
dtctl apply -f dynatrace/
```

## Notes

- The EDA connection URL must match the Event Stream endpoint from AAP
  (see `docs/INSTALL.md` Step 3).
- The connection token must match `EDA_EVENT_STREAM_TOKEN`.
- The Workflow trigger filter should be tuned to match the specific problem
  types you want to push to EDA (Phase 3).
- dtctl manages the connection via the Settings API schema
  `app:dynatrace.redhat.ansible.eda.webhook.connection`. Verify this works
  with your dtctl version; fall back to the Dynatrace UI if needed.
