# aap.eda.dynatrace.push — Roadmap

## Vision

Let **Ansible Automation Platform** react to **Dynatrace** problems automatically,
using Event-Driven Ansible in the **push** model:

> *"Dynatrace sees the problem, pushes the event. AAP fixes it — on your terms."*

Dynatrace Workflows detect problems and POST events to AAP's built-in Event
Streams endpoint. A rulebook activation evaluates conditions and launches
remediation (a Controller job template), optionally closing the loop back to the
Dynatrace problem. For customer environments where AAP is not internet-reachable,
Dynatrace EdgeConnect tunnels the push through an outbound-only WebSocket.

This is the **push** counterpart to
[aap.eda.dynatrace](https://github.com/ericcames/aap.eda.dynatrace) (pull model,
`dt_esa_api` polling). Both repos can coexist on the same AAP — objects are
namespaced `DT-EDA-PUSH -` (push) and `DT-EDA -` (pull).

Seeded from the 2026-06-04 customer meeting with the Dynatrace account team.

---

## Guiding Principles

- **Push, not pull** — Dynatrace initiates; AAP receives. The Dynatrace Workflow
  detects the problem and sends the event. EdgeConnect is only needed when the
  AAP endpoint is not directly reachable.
- **CaC on both sides** — AAP config via `infra.aap_configuration`; Dynatrace
  config via `dtctl`. No manual-only setup steps.
- **Verify against the upstream collection** — plugin args and event shape come
  from `github.com/Dynatrace/Dynatrace-EventDrivenAnsible`, not assumptions.
- **Notify before you remediate** — every environment starts notify-only, then
  human-gated, then full auto. Blast radius is controlled in two places: the
  Dynatrace Workflow trigger filter *and* the rulebook condition.
- **Secrets never land in the repo** — the real tenant id and all tokens live in
  the gitignored `docs/dev-environment.sh`; committed files use `<env-id>` and
  `REPLACE_ME_*`. CI fails the build on a leak.
- **Ship `ansible.cfg.example`, never a live `ansible.cfg`** — Ansible loads one
  cfg (no merge); a live project-local cfg shadows the home cfg and breaks Hub
  installs.
- **Stands on the dc1.azure RHDP AAP** — dev target is the dc1.azure AAP 2.6;
  install additively and namespace objects so it co-exists.

---

## Architecture

```
Dynatrace SaaS
  ├─ OneAgent (on dc1.azure RHEL hosts) → detects service failures
  ├─ Custom metric event rule (failed service)
  └─ Workflow (dtctl-managed)
       ├─ Trigger: problem opens matching failed-service criteria
       └─ Action: "Send event to Event-Driven Ansible"
            │
            ▼ HTTPS POST (Bearer token auth)
            │
     ┌──────┤ Customer env: EdgeConnect tunnels this
     │      │ Demo env: direct HTTPS (RHDP is internet-reachable)
     ▼      ▼
AAP EDA Event Stream endpoint
  /eda-event-streams/api/eda/v1/external_event_stream/<uuid>/post
     └─ Token Event Stream credential validates the Bearer token
          └─ Proxies to internal webhook source (0.0.0.0:5000)
               └─ Rulebook Activation
                    └─ Condition: match service failure event
                         └─ Action: run_job_template
                              └─ DT-EDA-PUSH - Restart Service
                                   └─ playbooks/restart_service.yml
                                        └─ systemctl restart <service>
```

Full diagrams: [`docs/architecture.md`](docs/architecture.md).

---

## Phases

### Phase 0 — Scaffold  ✅
- ✅ Create repo mirroring dc1.azure conventions
- ✅ Top-level governance (ROADMAP, CLAUDE.md, CHANGELOG, CI)
- ✅ CaC skeleton (`aap_config/`, `dynatrace/`, `rulebooks/`, `playbooks/`)
- ✅ docs/dev-environment.sh.example + architecture docs

**Exit criteria:** repo structure passes CI; ready for CaC content.

### Phase 1 — AAP plumbing (notify-only)  🔄
- 🔄 Event Stream credential (`DT-EDA-PUSH - Event Stream Token`)
- 🔄 Event Stream (`DT-EDA-PUSH - Dynatrace Events`)
- 🔄 Controller project, inventory, credentials
- 🔄 Notify-only job template (`DT-EDA-PUSH - Notify`)
- 🔄 Rulebook (`ansible.eda.webhook` source, push events)
- 🔄 Rulebook activation wired to Event Stream
- 🔄 `aap_config/load.yml` dispatch + validation

**Exit criteria:** `load.yml` applies cleanly; activation starts; Event Stream
endpoint is reachable and returns 401 without a valid token.

### Phase 2 — Dynatrace plumbing (dtctl)  🔄
- 🔄 Install Red Hat Ansible Connector from Dynatrace Hub (manual, one-time)
- 🔄 EDA connection config (`dynatrace/eda-connection.yaml`)
- 🔄 Workflow: problem trigger → "Send event to EDA" (`dynatrace/workflow-service-failure.yaml`)
- 🔄 Verify dtctl can manage the connection via settings schema
  `app:dynatrace.redhat.ansible.eda.webhook.connection`

**Exit criteria:** `dtctl apply -f dynatrace/` creates the Workflow + connection;
Workflow shows as active in Dynatrace.

### Phase 3 — End-to-end test (notify-only)  ⬜
- ⬜ Wire Dynatrace → AAP and fire the push path
- ⬜ Capture raw event payload from the Notify JT job log
- ⬜ Document the verified event shape
- ⬜ Tune the rulebook condition against the real payload

**Exit criteria:** push path fires end-to-end; event shape documented; condition
tuned against live data.

### Phase 4 — Remediation (restart service)  ⬜
- ⬜ `playbooks/restart_service.yml` — receives service name + host, runs
  `systemctl restart`, with safety controls (allowlist, throttle)
- ⬜ `DT-EDA-PUSH - Restart Service` job template
- ⬜ Rulebook second rule or action swap from notify to restart
- ⬜ Optional: Dynatrace problem comment write-back

**Exit criteria:** failed service on a dc1.azure host → Dynatrace detects → push
→ EDA → restart → service recovers.

### Phase 5 — Documentation & demo polish  ⬜
- ⬜ `docs/INSTALL.md` — full setup guide (AAP + Dynatrace sides)
- ⬜ `docs/DEMO.md` — meeting script, talk track, Claude prompts
- ⬜ `docs/dtctl.md` — dtctl setup, auth, apply commands
- ⬜ Screenshots in `docs/images/`
- ⬜ Push vs pull comparison table

**Exit criteria:** a new SE can set up and demo the push integration from docs
alone.

### Phase 6 — EdgeConnect (customer env)  ⬜
- ⬜ Document EdgeConnect deployment (Docker/K8s) for private networks
- ⬜ Host pattern config for routing to AAP
- ⬜ Optional dtctl management of EdgeConnect instances
- ⬜ Test: Workflow → EdgeConnect → Event Stream

**Exit criteria:** push path works through EdgeConnect in a non-internet-reachable
AAP environment.

### Phase 7 — Hardening  ⬜
- ⬜ Safety ramp: notify-only → human-gated (workflow approval node) → full auto
- ⬜ Event Stream token rotation plan
- ⬜ Observability: monitor activation health; alert if Event Stream stops receiving
- ⬜ Service restart allowlist + throttle guard
- ⬜ Kill switch documentation

**Exit criteria:** safety controls, monitoring, and kill switch in place.

---

## Naming Conventions

To co-exist safely with the pull model in a shared AAP instance, namespace
objects with a `DT-EDA-PUSH -` prefix.

| Object type             | Pattern                           | Example                                |
|-------------------------|-----------------------------------|----------------------------------------|
| AAP credential          | `DT-EDA-PUSH - <purpose>`        | `DT-EDA-PUSH - Event Stream Token`    |
| EDA project             | `DT-EDA-PUSH`                    | `DT-EDA-PUSH`                         |
| EDA event stream        | `DT-EDA-PUSH - <source>`         | `DT-EDA-PUSH - Dynatrace Events`      |
| EDA rulebook activation | `DT-EDA-PUSH - <use case>`       | `DT-EDA-PUSH - Service Remediation`   |
| Controller JT           | `DT-EDA-PUSH - <verb> <object>`  | `DT-EDA-PUSH - Restart Service`       |

---

## Decisions Log

| Date       | Decision                                                        | Rationale |
|------------|-----------------------------------------------------------------|-----------|
| 2026-06-04 | **Push** model (Dynatrace Workflows → AAP Event Streams)        | Customer wants to evaluate push vs pull; Dynatrace account team recommended EdgeConnect + Workflows |
| 2026-06-04 | **Separate repo** from the pull integration                      | Architecture is different enough (Event Streams vs polling, dtctl vs DE build) to warrant clean separation |
| 2026-06-04 | **Event Streams** (AAP built-in), not `dt_webhook` plugin        | Simpler — uses default DE, no custom build; AAP 2.5+ supports it natively |
| 2026-06-04 | **No EdgeConnect in Eric's demo env**                            | RHDP AAP is internet-reachable; EdgeConnect is only for customer's private network |
| 2026-06-04 | **`DT-EDA-PUSH -` prefix** for all AAP objects                  | Distinguishes from pull model's `DT-EDA -` prefix when both coexist on the same AAP |
| 2026-06-04 | **dtctl from the start** for Dynatrace CaC                     | Customer account team shared dtctl; CaC on both sides from day one |
| 2026-06-04 | **dc1.azure RHEL hosts** as remediation targets                  | First use case: monitor failed services on dc1.azure hosts, restart if appropriate |
| 2026-06-04 | **Custom metric/event** for service failure detection            | Customer will define specific service monitoring, not relying on Davis AI auto-detection alone |
| 2026-06-04 | Dynatrace account team shared **dtctl** (`github.com/dynatrace-oss/dtctl`) | kubectl-style CLI for Dynatrace; manages Workflows, connections, EdgeConnect declaratively |

---

## Push vs Pull Comparison

| Dimension            | Push (this repo)                            | Pull (aap.eda.dynatrace)                  |
|----------------------|---------------------------------------------|--------------------------------------------|
| Direction            | Dynatrace → AAP (inbound to AAP)            | AAP → Dynatrace (outbound from AAP)       |
| Trigger              | Dynatrace Workflow fires on problem          | EDA polls every N seconds                  |
| Latency              | Near-real-time (event-driven)               | Up to `delay` seconds (default 60s)        |
| Connectivity         | AAP must be reachable (or EdgeConnect)       | Only outbound HTTPS from AAP              |
| EdgeConnect          | Needed for private AAP networks              | Not needed                                 |
| Custom DE            | Not needed (default DE works)                | Required (dt_esa_api source plugin)        |
| Dynatrace config     | Workflow + Connector + connection            | API token only                             |
| AAP config           | Event Stream + webhook rulebook              | Credential + polling rulebook              |
| CaC tool (DT side)   | dtctl                                       | N/A                                        |

---

## Risks / Open Questions

- **Event payload shape** — the exact fields in the Dynatrace Workflow "Send
  event to EDA" action payload need to be captured from a live event (Phase 3).
  The rulebook condition will be tuned after.
- **dtctl settings schema** — verify that `dtctl apply` can manage the Red Hat
  Ansible Connector connection (schema
  `app:dynatrace.redhat.ansible.eda.webhook.connection`). If not, document the
  manual UI steps as a fallback.
- **Service restart safety** — which services are safe to auto-restart? Need an
  allowlist and throttle to prevent restart storms.
- **dc1.azure OneAgent** — the RHEL hosts need OneAgent installed to detect
  service failures. Tracked in `ericcames/dc1.azure#1`.
- **Event Stream token rotation** — the shared secret between Dynatrace and AAP
  needs a rotation plan.

---

## Reference Repositories

| Repo | Role |
|------|------|
| [aap.eda.dynatrace](https://github.com/ericcames/aap.eda.dynatrace) | Pull model counterpart |
| [dc1.azure](https://github.com/ericcames/dc1.azure) | Convention template + Event Streams CaC pattern |
| [Dynatrace/Dynatrace-EventDrivenAnsible](https://github.com/Dynatrace/Dynatrace-EventDrivenAnsible) | Source of truth for dt_webhook/dt_esa_api args + event shape |
| [dynatrace-oss/dtctl](https://github.com/dynatrace-oss/dtctl) | Dynatrace CLI for CaC |
| [aap.as.code](https://github.com/ericcames/aap.as.code) | AAP bootstrap / CaC patterns |

---

## Status Legend

- ✅ Complete
- 🔄 In progress
- ⬜ Not started
