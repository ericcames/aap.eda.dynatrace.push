# Manual Install Guide (UI) — DT-EDA Push Model

Step-by-step guide for setting up the **Dynatrace → AAP Event-Driven Ansible**
push integration entirely through the AAP and Dynatrace web UIs. A new SE
should be able to follow this guide and have a working end-to-end push path
without prior knowledge of the repo.

For the CLI/CaC-driven install, see [`INSTALL.md`](INSTALL.md).

> **Order matters.** AAP must be configured first (Part 1) because the
> Dynatrace side needs the Event Stream URL that AAP generates.

---

## Prerequisites

| Prerequisite | Details |
|-------------|---------|
| **AAP 2.5+** with EDA | Event Streams require AAP 2.5 or later. Dev target: dc1.azure RHDP AAP 2.6 |
| **Dynatrace SaaS tenant** | With OneAgent deployed on the hosts you want to monitor. Install playbooks: [Linux](https://dev.azure.com/ericcames/dc1.azure/_git/dc1.azure?path=/playbooks/install_dynatrace_oneagent_linux.yml), [Windows](https://dev.azure.com/ericcames/dc1.azure/_git/dc1.azure?path=/playbooks/install_dynatrace_oneagent_windows.yml) (dc1.azure repo) |
| **Network connectivity** | AAP must be reachable from Dynatrace SaaS over HTTPS/443 (directly or via EdgeConnect) |
| **Event Stream token** | A strong random shared secret that both sides will use. Generate one before you start: `openssl rand -base64 32` |

---

## Part 1 — AAP Configuration

You will create objects in both **Automation Controller** and **Automation
Decisions** (EDA). The organization for all objects is `IT Service Automation`
(or whichever org your AAP uses).

### Step 1: Create the Controller project

The project syncs playbooks and rulebooks from this Git repo.

1. Go to **Automation Controller > Projects > Add**
2. Fill in:

| Field | Value |
|-------|-------|
| Name | `DT-EDA-PUSH` |
| Organization | `IT Service Automation` |
| Source Control Type | `Git` |
| Source Control URL | `https://github.com/ericcames/aap.eda.dynatrace.push.git` |
| Source Control Branch | `main` |
| Options | Check **Clean** |

3. Click **Save**
4. Wait for the project sync to complete (status turns green)

> **The project must finish syncing before you create the job template** — the
> playbook dropdown is empty until sync completes.

### Step 2: Create the Controller inventory

1. Go to **Automation Controller > Inventories > Add > Add inventory**
2. Fill in:

| Field | Value |
|-------|-------|
| Name | `DT-EDA-PUSH` |
| Organization | `IT Service Automation` |

3. Click **Save**

No hosts needed — the Notify playbook runs on localhost.

### Step 3: Create the Notify job template

1. Go to **Automation Controller > Templates > Add > Add job template**
2. Fill in:

| Field | Value |
|-------|-------|
| Name | `DT-EDA-PUSH - Notify` |
| Job Type | `Run` |
| Inventory | `DT-EDA-PUSH` |
| Project | `DT-EDA-PUSH` |
| Playbook | `playbooks/notify_push_event.yml` |
| Options | Check **Prompt on launch** next to Variables |

3. Click **Save**

### Step 4: Create the EDA credential — Event Stream Token

This credential validates the Bearer token that Dynatrace sends with each
event.

1. Go to **Automation Decisions > Credentials > Create credential**
2. Fill in:

| Field | Value |
|-------|-------|
| Name | `DT-EDA-PUSH - Event Stream Token` |
| Organization | `IT Service Automation` |
| Credential Type | `Token Event Stream` |
| Auth Type | `Token` |
| HTTP Header Key | `Authorization` |
| Token | Your generated shared secret (the same value you will paste into Dynatrace in Part 2) |

3. Click **Create credential**

### Step 5: Create the EDA credential — Controller

This credential lets the rulebook activation launch Controller job templates.

1. Go to **Automation Decisions > Credentials > Create credential**
2. Fill in:

| Field | Value |
|-------|-------|
| Name | `DT-EDA-PUSH - Controller` |
| Organization | `IT Service Automation` |
| Credential Type | `Red Hat Ansible Automation Platform` |
| Host | Your AAP URL + `/api/controller/` (e.g. `https://aap-aap.apps.cluster-xxxxx.rhdp.net/api/controller/`) |
| Username | `admin` |
| Password | Your AAP admin password |
| Verify SSL | Unchecked (for RHDP self-signed certs) |

3. Click **Create credential**

### Step 6: Create the Event Stream

1. Go to **Automation Decisions > Event Streams > Create event stream**
2. Fill in:

| Field | Value |
|-------|-------|
| Name | `DT-EDA-PUSH - Dynatrace Events` |
| Credential | `DT-EDA-PUSH - Event Stream Token` |
| Organization | `IT Service Automation` |
| Forward Events | Enabled (checked) |

3. Click **Create event stream**
4. **Copy the External URL** from the event stream detail page — you need this
   for the Dynatrace connection in Part 2. It looks like:
   `https://<aap>/eda-event-streams/api/eda/v1/external_event_stream/<uuid>/post/`

> **Save this URL somewhere handy** — you will paste it into Dynatrace in
> Step 11.

### Step 7: Create the EDA project

1. Go to **Automation Decisions > Projects > Create project**
2. Fill in:

| Field | Value |
|-------|-------|
| Name | `DT-EDA-PUSH` |
| Organization | `IT Service Automation` |
| SCM URL | `https://github.com/ericcames/aap.eda.dynatrace.push.git` |

3. Click **Create project**
4. Wait for the project sync to complete

> **Do not set SCM Branch** — the `ansible.eda` 2.5.0 module does not support
> it. EDA syncs the repo's default branch (`main`), which is correct.

### Step 8: Create the Rulebook Activation

This is the core wiring — it connects the Event Stream to the rulebook and
gives the rulebook permission to launch Controller jobs.

1. Go to **Automation Decisions > Rulebook Activations > Create rulebook
   activation**
2. Fill in:

| Field | Value |
|-------|-------|
| Name | `DT-EDA-PUSH - Service Remediation` |
| Organization | `IT Service Automation` |
| Project | `DT-EDA-PUSH` |
| Rulebook | `dynatrace_push_events.yml` |
| Decision Environment | `Default Decision Environment` |
| Enabled | Checked |

3. Under **Event Streams**, add:
   - Event Stream: `DT-EDA-PUSH - Dynatrace Events`
   - Source name: `__SOURCE_1`

4. Under **Credentials**, add:
   - `DT-EDA-PUSH - Controller`

5. Under **Variables** (extra vars), paste:

```yaml
dt_match_pattern: ".*"
my_organization: "IT Service Automation"
notify_template: "DT-EDA-PUSH - Notify"
remediate_workflow: "DC1.Azure - Remediate Website"
```

6. Click **Create rulebook activation**
7. Verify the activation status turns to **Running**

> **The Default Decision Environment works** — no custom DE build needed. The
> `ansible.eda.webhook` source plugin ships with the default DE.

---

## Part 2 — Dynatrace Configuration

### Step 9: Install the Red Hat Ansible Connector (one-time)

This is a one-time setup per Dynatrace environment. It adds the "Send event to
Event-Driven Ansible" Workflow action.

1. In Dynatrace, go to **Hub** (left nav)
2. Search for **Red Hat Ansible**
3. Click **Install**

### Step 10: Create the EDA connection

The connection tells Dynatrace where to send events. It needs the Event Stream
URL from Step 6 and the shared token from Step 4.

1. Go to **Settings > Connections > Red Hat Ansible**
2. Click the **Event-Driven Ansible** tab
3. Click **+ Connection** and fill in:

| Field | Value |
|-------|-------|
| Name | `DT-EDA-PUSH - AAP Event Stream` |
| URL | The Event Stream external URL from Step 6 |
| Token | The same shared secret you used in Step 4 |
| Type | `api-token` |
| Event Stream Enabled | Checked |

4. Click **Save**

### Step 11: Create the Workflow

The Workflow fires when Davis opens a problem and pushes the event to AAP.

1. Go to **Workflows** (left nav) > **+ Workflow**
2. Set the Workflow name: `DT-EDA-PUSH - Service Failure → AAP`
3. Configure the **Trigger**:
   - Click the trigger node
   - Type: **Davis problem**
   - Analysis ready: **true** (wait for root-cause analysis)
   - Categories: check **Availability**, **Error**, **Resource** — uncheck
     **Info**
   - On problem close: **unchecked** (fire on open only)
4. Add a **Task**:
   - Click **+** to add a task after the trigger
   - Search for **Send event to Event-Driven Ansible** (from the Red Hat
     Ansible Connector)
   - Name the task: `send_to_aap`
   - Connection: select **DT-EDA-PUSH - AAP Event Stream** (the connection from
     Step 10)
   - Event data: `{{ event() }}` (sends the full Davis problem event)
5. Click **Save**
6. **Activate** the Workflow (toggle at the top)

### Step 12: Create a classic access token (for problem management)

dtctl's OAuth platform token cannot list or close Dynatrace problems. A separate
classic access token is needed for the Problems API v2.

1. Go to **Settings > Access tokens** (or search "Access tokens" in the DT
   search bar)
2. Click **Generate new token**
3. Fill in:

| Field | Value |
|-------|-------|
| Token name | `dtctl-problems` |
| Expiration date | (leave empty for no expiry, or set a rotation date) |

4. Search for `problems` in the scopes table and check:
   - `Read problems` (`problems.read`)
   - `Write problems` (`problems.write`)
   - `Read security problems` (`securityProblems.read`)
   - `Write security problems` (`securityProblems.write`)
5. Click **Generate token**
6. **Copy the `dt0c01.*` token** (you only see it once)
7. Save it as `DT_API_TOKEN` in `docs/dev-environment.sh`

> **Why a separate token?** Platform tokens (`dt0s16.*`) do not expose
> `environment-api:problems:*` scopes. The classic access token is the only way
> to list/close problems via the API — for example, cleaning up stale "Process
> unavailable" problems from decommissioned hosts.

### Step 13: Enable process group availability alerting

**This step is critical.** Without it, Dynatrace sees a process stop/start as
an informational event but never generates a Davis problem — so the Workflow
trigger never fires and no event reaches AAP.

1. Go to **Infrastructure & Operations > Technologies and processes**
2. Find the process group you want to monitor (e.g. **Apache Web Server httpd**)
   and click into it
3. Click the **...** menu (or **Settings**) on the process group page
4. Navigate to **Availability monitoring** (or find it via **Settings >
   Anomaly detection > Process group availability**)
5. Set:

| Field | Value |
|-------|-------|
| Enable process group availability monitoring | Checked |
| Alerting mode | `Alert if any process group instance is unavailable` |

6. **Save**

Alternatively, navigate directly:

1. Go to **Settings > Anomaly detection > Process group availability**
2. Click **+ Add rule**
3. Scope the rule to your process group
4. Set alerting mode to **Alert if any process group instance is unavailable**
5. **Save**

> **Repeat for each process group** you want to monitor. Each needs its own
> alerting rule. Without this rule, Davis generates only informational events —
> the Workflow trigger fires on *problems*, not informational events.

---

## Part 3 — Validate and Test

### Step 14: Verify AAP objects in the UI

Walk through each section of the AAP UI and confirm the objects exist and are
healthy:

| Where to check | What to verify |
|----------------|---------------|
| **Controller > Projects** | `DT-EDA-PUSH` — status is **Successful** (synced) |
| **Controller > Inventories** | `DT-EDA-PUSH` — exists |
| **Controller > Templates** | `DT-EDA-PUSH - Notify` — exists, playbook is `playbooks/notify_push_event.yml` |
| **Decisions > Credentials** | `DT-EDA-PUSH - Event Stream Token` — exists |
| **Decisions > Credentials** | `DT-EDA-PUSH - Controller` — exists |
| **Decisions > Projects** | `DT-EDA-PUSH` — status is **Completed** |
| **Decisions > Event Streams** | `DT-EDA-PUSH - Dynatrace Events` — exists, has an External URL |
| **Decisions > Rulebook Activations** | `DT-EDA-PUSH - Service Remediation` — status is **Running** |

### Step 15: Verify Dynatrace objects in the UI

| Where to check | What to verify |
|----------------|---------------|
| **Hub** | Red Hat Ansible Connector is installed |
| **Settings > Connections > Red Hat Ansible > Event-Driven Ansible** | `DT-EDA-PUSH - AAP Event Stream` connection exists with correct URL and token |
| **Workflows** | `DT-EDA-PUSH - Service Failure → AAP` is active |
| **Settings > Anomaly detection > Process group availability** | Alerting rule exists for your target process group(s) |

### Step 16: End-to-end test

Stop the monitored service on a target host to trigger a real Davis problem:

```bash
# SSH to the target host (e.g. dc1.azure Linux host)
sudo systemctl stop httpd
```

Then watch the chain fire:

1. **Dynatrace > Problems** — a "Process unavailable" problem should open
   within 1–2 minutes
2. **Dynatrace > Workflows** — click into the Workflow, then **Executions** —
   the `send_to_aap` task should show a successful run
3. **AAP > Automation Decisions > Rulebook Activations > DT-EDA-PUSH - Service
   Remediation > History** — you should see the received event
4. **AAP > Automation Controller > Jobs** — a job should have fired with the
   full Davis problem event payload visible in the job output

Restart the service when done:

```bash
sudo systemctl start httpd
```

> **Timing:** after stopping the service, allow 1–2 minutes for OneAgent to
> detect the process is gone, Davis to open a problem, the Workflow to fire,
> and the event to arrive in AAP.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Event Stream returns **401** | Token mismatch | The token in the Dynatrace EDA connection must exactly match the token in the AAP `DT-EDA-PUSH - Event Stream Token` credential |
| Rulebook activation won't start | Decision Environment not available or EDA project sync failed | Check **Decisions > Decision Environments** — Default DE should be present. Check EDA project sync status |
| Activation starts but no events arrive | Event Stream not wired to activation | Edit the activation and confirm `DT-EDA-PUSH - Dynatrace Events` is listed under Event Streams with source name `__SOURCE_1` |
| Workflow not firing on problem | Process group alerting not enabled (Step 13) | Without the alerting rule, Davis only generates informational events. The Workflow trigger fires on *problems*, not informational events |
| Workflow fires but event not received in AAP | Network — Dynatrace can't reach AAP | Check Workflow execution logs for HTTP errors. Verify AAP is reachable from the internet on HTTPS/443 |
| Workflow fires but wrong connection | Multiple EDA connections exist | Verify the Workflow's `send_to_aap` task references the correct connection (`DT-EDA-PUSH - AAP Event Stream`) |
| Controller JT not found | Controller project hasn't synced | Go to **Controller > Projects > DT-EDA-PUSH** and trigger a sync. Wait for it to complete before checking templates |
| Job template shows "Playbook not found" | Project SCM type is wrong | Edit the Controller project — Source Control Type must be `Git` with the correct URL. Re-sync after fixing |
| Orphaned active problem in Dynatrace | Host/process was deleted; Dynatrace won't see recovery | Close via Problems API v2: `curl -X POST "${DT_API_HOST}/api/v2/problems/<id>/close" -H "Authorization: Api-Token ${DT_API_TOKEN}" -H "Content-Type: application/json" -d '{"message":"Host decommissioned"}'` (requires classic access token from Step 12) |

---

## Event Shape Reference

When Dynatrace pushes a Davis problem event, it arrives in AAP wrapped as
`dt_push_event.eventData.<fields>`. The key fields (captured from a live event,
2026-06-07):

| Field | Example | Notes |
|-------|---------|-------|
| `display_id` | `P-26068` | Problem ID |
| `event.name` | `Process unavailable` | Problem title |
| `event.status` | `ACTIVE` | Problem status |
| `event.category` | `AVAILABILITY` | Category (AVAILABILITY, ERROR, RESOURCE) |
| `event.severity` | `3` | Severity level |
| `affected_entity_names[0]` | `Apache Web Server httpd` | Process group name |
| `host.name[0]` | `dc1az-lnx-medium-2cpu-8gb-...` | FQDN (contains `lnx`/`win` for OS detection) |
| `dt.entity.host[0]` | `HOST-E856A4268260CA23` | Host entity ID |
| `event.description` | Markdown narrative | Human-readable problem description |

Full event shape: [`dynatrace/davis-problem-event-shape.yaml`](../dynatrace/davis-problem-event-shape.yaml)

> **Dot-notated keys** (`event.name`, `event.status`, `host.name`) require
> **bracket notation** in Jinja2: `_dt_event['event.name']` — not
> `_dt_event.event.name`.

---

## Quick Reference — Object Names

All AAP objects use the `DT-EDA-PUSH -` prefix to co-exist with the pull
model's `DT-EDA -` objects on the same AAP:

| Object | Name |
|--------|------|
| Controller project | `DT-EDA-PUSH` |
| Controller inventory | `DT-EDA-PUSH` |
| Controller job template | `DT-EDA-PUSH - Notify` |
| EDA credential (token) | `DT-EDA-PUSH - Event Stream Token` |
| EDA credential (controller) | `DT-EDA-PUSH - Controller` |
| EDA project | `DT-EDA-PUSH` |
| EDA event stream | `DT-EDA-PUSH - Dynatrace Events` |
| EDA rulebook activation | `DT-EDA-PUSH - Service Remediation` |
| Dynatrace Workflow | `DT-EDA-PUSH - Service Failure → AAP` |
| Dynatrace EDA connection | `DT-EDA-PUSH - AAP Event Stream` |
