# aap.eda.dynatrace.push

**Dynatrace → AAP Event-Driven Ansible (push model via Event Streams)**

Dynatrace Workflows detect problems and push events to AAP's Event-Driven
Ansible via Event Streams. A rulebook activation evaluates conditions and
launches remediation (a Controller job template), optionally closing the loop
back to the Dynatrace problem.

This is the **push** counterpart to
[aap.eda.dynatrace](https://github.com/ericcames/aap.eda.dynatrace) (pull).
Both repos can coexist on the same AAP — objects are namespaced
`DT-EDA-PUSH -` (push) and `DT-EDA -` (pull).

## Architecture

```
Dynatrace SaaS
  └─ Workflow (trigger: problem detected)
       └─ Action: "Send event to Event-Driven Ansible"
            │
            ▼ HTTPS POST (Bearer token auth)
            │
     [EdgeConnect — customer private networks only]
            │
            ▼
AAP EDA Event Stream endpoint
  └─ Rulebook Activation (ansible.eda.webhook source)
       └─ Condition match → run_job_template
            └─ Remediation playbook (e.g. restart a failed service)
```

All configuration is code:
- **AAP side:** `aap_config/` via `infra.aap_configuration.dispatch`
- **Dynatrace side:** `dynatrace/` via `dtctl apply`

## Quick start

```bash
# 1. Install collections
ansible-galaxy collection install -r collections/requirements.yml

# 2. Set up secrets
cp docs/dev-environment.sh.example docs/dev-environment.sh
# Edit docs/dev-environment.sh with your real values
source docs/dev-environment.sh

# 3. Apply AAP configuration
ansible-playbook -i aap_config/inventory/ aap_config/load.yml

# 4. Apply Dynatrace configuration
dtctl apply -f dynatrace/
```

See [`docs/INSTALL.md`](docs/INSTALL.md) for the full setup guide and
[`ROADMAP.md`](ROADMAP.md) for phasing, architecture, and the decisions log.

## Repository layout

```
aap_config/         AAP Config-as-Code (Event Streams, credentials, JTs)
dynatrace/          Dynatrace Config-as-Code (Workflows, connections via dtctl)
rulebooks/          EDA rulebook (webhook source for push events)
playbooks/          Remediation + notify playbooks
docs/               Setup, demo, architecture docs
collections/        Ansible collection requirements (control node)
```

## Related repositories

| Repo | Role |
|------|------|
| [aap.eda.dynatrace](https://github.com/ericcames/aap.eda.dynatrace) | Pull model (dt_esa_api polls Dynatrace) |
| [dc1.azure](https://github.com/ericcames/dc1.azure) | Convention template + dc1 RHEL hosts |
| [Dynatrace/Dynatrace-EventDrivenAnsible](https://github.com/Dynatrace/Dynatrace-EventDrivenAnsible) | Upstream collection (dt_esa_api + dt_webhook) |
| [dynatrace-oss/dtctl](https://github.com/dynatrace-oss/dtctl) | Dynatrace CLI for CaC |
