# Changelog

All notable changes to `aap.eda.dynatrace.push` are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added

- Initial repo scaffold mirroring dc1.azure conventions
- AAP Config-as-Code: Event Stream, credentials, project, inventory, job templates, rulebook activation
- Dynatrace Config-as-Code: Workflow + EDA connection via dtctl
- Push-model rulebook (`ansible.eda.webhook` source via Event Streams)
- Notify-only playbook (Phase 1 first action)
- Architecture docs, install guide, dev-environment template
- CI: yamllint + secret-leak guard
