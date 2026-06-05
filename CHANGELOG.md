# Changelog

All notable changes to `aap.eda.dynatrace.push` are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added

- Initial repo scaffold mirroring dc1.azure conventions
- AAP Config-as-Code: Event Stream, credentials, project, inventory, job templates, rulebook activation
- Dynatrace Config-as-Code: Workflow JSON (`dtctl create workflow`) + EDA connection reference
- Push-model rulebook (`ansible.eda.webhook` source via Event Streams)
- Notify-only playbook (Phase 1 first action)
- Architecture docs, install guide, dev-environment template
- CI: yamllint + secret-leak guard
- dtctl quick-reference and setup docs in `dynatrace/README.md`

### Fixed

- Controller project: added `scm_type: git`, `scm_clean`, `wait: true` — without
  `scm_type` the project created as manual with no SCM URL (#2)
- EDA project: removed unsupported `scm_branch` param (ansible.eda 2.5.0) (#2)

### Verified (2026-06-05)

- Full push path: direct POST → Event Stream → rulebook match → Notify JT fires (job #205)
- Event payload lands intact in the job log — all fields visible, no data loss
- dtctl v0.28.1 authenticates via browser OAuth, manages Workflows and settings
- Settings schema: `app:dynatrace.redhat.ansible:eda-webhook.connection`
- Default Decision Environment works with Event Streams (no custom DE needed)
