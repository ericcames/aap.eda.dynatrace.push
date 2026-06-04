# aap_config/ — AAP Configuration as Code

Declarative AAP objects applied by `load.yml` via the
`infra.aap_configuration.dispatch` role. Every object is keyed by name —
re-running is idempotent.

## Auth

Username/password basic auth (no token). The certified `ansible.eda` 2.5.0
modules reject `controller_token`, so `aap_token` is left empty in
`group_vars/all.yml`. All roles authenticate with
`AAP_CONTROLLER_USERNAME`/`AAP_CONTROLLER_PASSWORD`.

## Structure

```
aap_config/
├── load.yml                   Entry point — dispatches all files/
├── validate.yml               Standalone post-deploy verification
├── validate_tasks.yml         Shared validation tasks
├── inventory/aap.yml          Localhost (API-driven, no remote hosts)
├── group_vars/all.yml         Connection vars, secret references, object names
└── files/
    ├── controller_projects.yml        Controller project (this repo)
    ├── controller_inventories.yml     Empty inventory (localhost for notify)
    ├── controller_job_templates.yml   Notify JT (Phase 1) + Restart JT (Phase 4)
    ├── eda_credentials.yml            Event Stream Token + Controller creds
    ├── eda_event_streams.yml          Dynatrace Event Stream (push receiver)
    ├── eda_projects.yml               EDA project (this repo)
    └── eda_rulebook_activations.yml   Activation wired to Event Stream
```

## Run

```bash
source docs/dev-environment.sh && \
ansible-playbook -i aap_config/inventory/ aap_config/load.yml
```
