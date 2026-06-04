# Install Guide — DT-EDA Push Model

## Prerequisites

| Prerequisite | Details |
|-------------|---------|
| AAP 2.5+ with EDA | Event Streams requires AAP 2.5 or later |
| Dynatrace SaaS tenant | With Red Hat Ansible Connector installed from Dynatrace Hub |
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

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Event Stream returns 401 | Token in Dynatrace connection must match `EDA_EVENT_STREAM_TOKEN` |
| Activation not starting | Check the Decision Environment is available (default DE should work) |
| Workflow not firing | Check Workflow trigger filter matches the problem type |
| No event received in AAP | Check network connectivity (Dynatrace → AAP HTTPS 443) |
