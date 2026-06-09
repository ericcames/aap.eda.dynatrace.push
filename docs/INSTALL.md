# Install Guide — DT-EDA Push Model

## Prerequisites

| Prerequisite | Details |
|-------------|---------|
| AAP 2.5+ with EDA | Event Streams requires AAP 2.5 or later |
| Dynatrace SaaS tenant | With Red Hat Ansible Connector installed from Dynatrace Hub |
| Dynatrace OneAgent | Deployed on target hosts. Install playbooks: [Linux](https://dev.azure.com/ericcames/dc1.azure/_git/dc1.azure?path=/playbooks/install_dynatrace_oneagent_linux.yml), [Windows](https://dev.azure.com/ericcames/dc1.azure/_git/dc1.azure?path=/playbooks/install_dynatrace_oneagent_windows.yml) (dc1.azure repo) |
| `dtctl` CLI | Install from [github.com/dynatrace-oss/dtctl](https://github.com/dynatrace-oss/dtctl) |
| `ansible-galaxy` | With `~/.ansible.cfg` configured for Red Hat Automation Hub (see `ansible.cfg.example`) |
| Network connectivity | AAP must be reachable from Dynatrace (directly or via EdgeConnect) |

## Step 1: Set up secrets

```bash
cp docs/dev-environment.sh.example docs/dev-environment.sh
# Edit with your real values:
#   - AAP hostname, username, password
#   - Event Stream shared secret (generate a strong random token)
#   - Dynatrace tenant URL
#   - dtctl OAuth credentials
source docs/dev-environment.sh
```

## Step 2: Install collections

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

## Step 3: Apply AAP configuration

```bash
ansible-playbook -i aap_config/inventory/ aap_config/load.yml
```

This creates:
- Event Stream credential + Event Stream endpoint
- Controller project, inventory, Notify job template
- EDA project, rulebook activation (wired to Event Stream)

After the playbook completes, note the Event Stream endpoint URL from the AAP
UI: **Automation Decisions > Event Streams > DT-EDA-PUSH - Dynatrace Events**.
Copy the external endpoint URL — you'll need it for the Dynatrace connection.

## Step 4: Configure Dynatrace

### 4a: Install the Red Hat Ansible Connector (one-time)

In Dynatrace: **Hub > Red Hat Ansible > Install**

### 4b: Apply dtctl configuration

Update `dynatrace/eda-connection.yaml` with the Event Stream endpoint URL from
Step 3, then:

```bash
dtctl apply -f dynatrace/
```

Or configure manually:
1. **Settings > Connections > Red Hat Ansible > Event-Driven Ansible** tab
2. Create a connection with the Event Stream URL and the shared token

### 4c: Verify the Workflow

In Dynatrace: **Workflows** — confirm the service-failure Workflow is active.

## Step 5: Validate

```bash
# Verify all AAP objects exist
ansible-playbook -i aap_config/inventory/ aap_config/validate.yml
```

To test the full push path, trigger a problem in Dynatrace and watch for the
Notify JT to fire in AAP Controller.

## Closing stale problems

When hosts are decommissioned or rebuilt, Dynatrace keeps "Process unavailable"
problems open because the process group instance never comes back. Close them
with the Environment API v2:

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

`DT_API_TOKEN` is a classic access token (`dt0c01.*`) created in the Dynatrace
environment UI (**Settings > Access tokens > Generate new token**). Required
scopes: `problems.read`, `problems.write`, `securityProblems.read`,
`securityProblems.write`. This is separate from dtctl's OAuth platform token,
which does not have the `environment-api:problems:*` scopes.

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Event Stream returns 401 | Token in Dynatrace connection must match `EDA_EVENT_STREAM_TOKEN` |
| Activation not starting | Check the Decision Environment is available (default DE should work) |
| Workflow not firing | Check Workflow trigger filter matches the problem type |
| No event received in AAP | Check network connectivity (Dynatrace → AAP HTTPS 443) |
| Stale "Process unavailable" problems | Host was decommissioned — close via Problems API v2 (see above) |
