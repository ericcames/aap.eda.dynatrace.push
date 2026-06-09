---
name: dt-eda-push-install
description: >-
  Install (or re-install) the DT-EDA-PUSH Dynatrace-push integration into a
  working AAP via aap_config/load.yml — creates the Event Stream, credentials,
  project, job templates, and starts the rulebook activation. Then applies the
  Dynatrace dtctl configs.
trigger: >-
  When the user asks to install / set up / deploy the DT-EDA push integration
  on AAP, run load.yml, or apply the dtctl configs.
---

# DT-EDA-PUSH Install Skill

## Prerequisites

Before running, confirm:
1. `docs/dev-environment.sh` exists and is sourced (ask the user to create it from
   `docs/dev-environment.sh.example` if missing)
2. Collections are installed: `ansible-galaxy collection install -r collections/requirements.yml`
3. AAP is reachable at `$AAP_HOSTNAME`
4. dtctl is installed and authenticated (for the Dynatrace side)
5. `DT_API_TOKEN` is set in `docs/dev-environment.sh` (classic `dt0c01.*` access
   token for problem management — see Step 3b)

## Steps

### Step 1: Source the environment

```bash
source docs/dev-environment.sh
```

### Step 2: Apply AAP configuration

```bash
ansible-playbook -i aap_config/inventory/ aap_config/load.yml
```

Watch for:
- All tasks should be `ok` or `changed` — no failures
- Validation at the end confirms every object exists
- Note the Event Stream endpoint URL from the output or AAP UI

### Step 3: Apply Dynatrace configuration (optional)

If dtctl is set up:

```bash
dtctl apply -f dynatrace/
```

#### Workflow trigger: `analysisReady: false`, `onProblemClose: true`

The workflow fires immediately when Davis opens a problem — it does not wait for
root cause analysis to complete. This cuts detection from ~6 min to ~2-3 min.
Root cause data is queried separately via the Problems API v2 after remediation
(see AB#154 in dc1.azure).

The workflow also fires on problem close (`onProblemClose: true`). When
Dynatrace confirms the problem has cleared, a CLOSED event reaches AAP EDA and
the rulebook launches the `DC1.Azure - Confirm Resolution (DT)` JT, which
posts a confirmation work note to the ServiceNow incident — closing the
detect→remediate→confirm loop.

#### Process group availability alerting

Without this rule, Dynatrace sees httpd stop/start as informational events but
never generates a Davis problem — so the Workflow trigger never fires.

```bash
dtctl create settings \
  --schema builtin:availability.process-group-alerting \
  --scope  PROCESS_GROUP-685F2A71A785ADB9 \
  --file   dynatrace/process-group-availability.yaml
```

Verify the rule is in place:

```bash
dtctl get settings --schema builtin:availability.process-group-alerting
```

> **Note:** The scope (`PROCESS_GROUP-685F2A71A785ADB9`) is the httpd process
> group on the current dc1.azure environment. If the process group entity ID
> differs in a new environment, find it with:
> `dtctl query 'fetch dt.entity.process_group'`

### Step 3b: Close stale problems (optional)

When hosts are decommissioned or the environment is rebuilt, Dynatrace keeps
"Process unavailable" problems open because the process group instance never
comes back. Close them with the Environment API v2 using `DT_API_TOKEN` (a
classic `dt0c01.*` access token — dtctl's OAuth platform token does not have the
required `environment-api:problems:*` scopes).

List open problems:

```bash
source docs/dev-environment.sh
curl -s "${DT_API_HOST}/api/v2/problems?problemSelector=status(open)" \
  -H "Authorization: Api-Token ${DT_API_TOKEN}" | python3 -m json.tool
```

Close a stale problem by ID:

```bash
curl -s -X POST "${DT_API_HOST}/api/v2/problems/<problemId>/close" \
  -H "Authorization: Api-Token ${DT_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"message":"Host decommissioned"}'
```

> **Token setup:** Create the classic access token in the Dynatrace environment
> UI: **Settings > Access tokens > Generate new token**, name `dtctl-problems`,
> scopes: `problems.read`, `problems.write`, `securityProblems.read`,
> `securityProblems.write`. Save the `dt0c01.*` value to `DT_API_TOKEN` in
> `docs/dev-environment.sh`.

### Step 4: Verify

```bash
ansible-playbook -i aap_config/inventory/ aap_config/validate.yml
```

Check the AAP UI:
- **Automation Decisions > Event Streams** — `DT-EDA-PUSH - Dynatrace Events` exists
- **Automation Decisions > Rulebook Activations** — `DT-EDA-PUSH - Service Remediation` is running
- **Automation Controller > Templates** — `DT-EDA-PUSH - Notify` exists

Check Dynatrace:
- **Process group availability** — `dtctl get settings --schema builtin:availability.process-group-alerting` returns the httpd rule with `enabled: true`
- **Workflow** — `dtctl get workflows` shows `DT-EDA-PUSH - Service Failure → AAP` as active

## Event shape reference

The live Davis problem event shape is documented in
`dynatrace/davis-problem-event-shape.yaml` (captured 2026-06-07). Key points:

- Event arrives wrapped: `dt_push_event.eventData.<fields>`
- Field names use **dots** (`event.name`, `event.status`, `host.name`) — require
  **bracket notation** in Jinja2: `_dt_event['event.name']`
- Problem ID is `display_id` (not `problemId`)
- Hostname is in `host.name[0]` (list, FQDN with `lnx`/`win` for OS detection)
- `affected_entity_names[0]` is the process group name, not the hostname
