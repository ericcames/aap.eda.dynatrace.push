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

Without this rule, Dynatrace sees the web process stop/start as informational
events but never generates a Davis problem — so the Workflow trigger never
fires.

**You need one rule per process group you want to self-heal — one per OS.** The
rule is scoped to a specific `PROCESS_GROUP` entity, so a Linux-only setup
covers httpd but **silently misses Windows/IIS** (a real trap: the Windows
self-heal looks "installed" but never fires because no problem is ever raised).

First find the process-group entity IDs for the web tier (the names differ by
OS — `Apache Web Server httpd` on Linux, `IIS app pool DefaultAppPool` on
Windows):

```bash
dtctl query 'fetch dt.entity.process_group | filter contains(entity.name, "httpd") or contains(entity.name, "IIS app pool")'
```

Then create the rule for **each** one (the value file is OS-agnostic):

```bash
# Linux — Apache httpd
dtctl create settings \
  --schema builtin:availability.process-group-alerting \
  --scope  PROCESS_GROUP-<httpd-id> \
  --file   dynatrace/process-group-availability.yaml

# Windows — IIS app pool DefaultAppPool (don't skip this one)
dtctl create settings \
  --schema builtin:availability.process-group-alerting \
  --scope  PROCESS_GROUP-<iis-app-pool-id> \
  --file   dynatrace/process-group-availability.yaml
```

Verify both rules are in place (expect one object per web PG):

```bash
dtctl get settings --schema builtin:availability.process-group-alerting
```

> **Note:** Process-group entity IDs are environment-specific — always resolve
> them by name (above) rather than hardcoding. On a multi-OS environment, confirm
> you have a rule for **both** the httpd and IIS-app-pool process groups; a
> missing IIS rule is the most common reason "Windows self-heal doesn't work."

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
- **Process group availability** — `dtctl get settings --schema builtin:availability.process-group-alerting` returns a rule with `enabled: true` for **each** web process group (httpd on Linux, IIS app pool on Windows)
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

> **Windows host-name gotcha:** Windows caps the OS computer name at the 15-char
> NetBIOS limit, so a long VM name (e.g. `myapp-win-small-2cpu-4gb-<suffix>`)
> truncates to `myapp-win-small` and **every Windows build collides on one
> Dynatrace host identity** — a problem/incident then attributes to a *stale
> prior instance* even though remediation fixes the live host. Install the
> Windows OneAgent with `--set-host-name=<unique FQDN>` (mirroring how Linux
> already reports its full hostname) so `host.name` is unique per host. Linux is
> unaffected.
