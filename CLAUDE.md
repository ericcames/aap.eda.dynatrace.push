# Claude Code working guidelines — aap.eda.dynatrace.push

Repo-specific guidance for AI contributors. Read [`ROADMAP.md`](ROADMAP.md) first.

## What this repo is

A working setup for **Dynatrace → AAP Event-Driven Ansible** in the **push**
model: Dynatrace Workflows detect problems and POST events to AAP's Event
Streams endpoint. A rulebook activation evaluates conditions and launches
remediation. This is the push counterpart to
[aap.eda.dynatrace](https://github.com/ericcames/aap.eda.dynatrace) (pull).

## Invariants (do not break without a Decisions Log entry)

- **Push via Event Streams.** Dynatrace initiates; AAP receives via the
  built-in Event Stream endpoint. No custom `dt_webhook` source plugin — use
  `ansible.eda.webhook` (ships with the default DE).
- **EdgeConnect is for the customer env only.** Eric's RHDP demo env is
  internet-reachable, so Dynatrace SaaS posts directly. EdgeConnect is
  documented but not deployed in the demo.
- **No secrets, ever, in the repo.** Real Dynatrace tenant id and all tokens
  live only in the gitignored `docs/dev-environment.sh`. Committed files use
  `<env-id>` / `REPLACE_ME_*`. CI fails on a tenant-id leak.
- **Secrets file is `docs/dev-environment.sh`** (sourceable, gitignored) — never
  a `.md`. Commit only `docs/dev-environment.sh.example`.
- **Never ship a live `ansible.cfg`.** Only `ansible.cfg.example`. A live
  project-local cfg shadows `~/.ansible.cfg` and breaks Hub collection installs.
- **Verify against the upstream collection.** Plugin args and event shape come
  from `github.com/Dynatrace/Dynatrace-EventDrivenAnsible`, not memory.
- **Notify before remediate.** Default new automation to notify-only.
- **CaC on both sides.** AAP via `infra.aap_configuration`; Dynatrace via
  `dtctl`. No manual-only setup steps.

## Conventions

- Mirror [`dc1.azure`](https://github.com/ericcames/dc1.azure) structure/tone.
- Namespace AAP objects with a `DT-EDA-PUSH -` prefix (see ROADMAP).
- YAML must pass `yamllint` against `.yamllint` (CI gate).
- One concern per PR; update `CHANGELOG.md` under `[Unreleased]`; update the
  ROADMAP phase status / Decisions Log when the plan changes.
- Dev target is the **dc1.azure RHDP AAP 2.6** — install additively.
