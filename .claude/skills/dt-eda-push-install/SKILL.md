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

### Step 4: Verify

```bash
ansible-playbook -i aap_config/inventory/ aap_config/validate.yml
```

Check the AAP UI:
- **Automation Decisions > Event Streams** — `DT-EDA-PUSH - Dynatrace Events` exists
- **Automation Decisions > Rulebook Activations** — `DT-EDA-PUSH - Service Remediation` is running
- **Automation Controller > Templates** — `DT-EDA-PUSH - Notify` exists
