# Automation orchestrator demo setup

`ao_setup.yml` provisions automation orchestrator (AO) through a Red Hat Demo Platform
AAP instance and loads base configuration into it. It runs against `localhost` and uses only
`ansible.builtin` modules, so any execution environment works.

> **Want AO without OpenShift?** [ao-on-microshift/INSTRUCTIONS.md](ao-on-microshift/INSTRUCTIONS.md)
> is a step-by-step runbook, written for an AI agent to follow, that builds AO on MicroShift in a
> single RHEL VM and connects it to your existing AAP, with optional demo workflows.

## What it does

1. **Demo AAP**: launches `APD | Single demo setup` with survey answer `demo: infrastructure`
   and waits for it to finish.
2. **Demo AAP**: launches `Infrastructure | Automation Orchestrator | Install` and waits
   (the install can take a long time; polling allows up to 2 hours).
3. **Demo AAP**: creates or updates the content the [Fleet Health Demo](#fleet-health-demo) runs:
   an `AO Demo Content` project that syncs this repository, an `AO Demo Inventory` with
   `localhost`, and five `AO Demo | ...` job templates.
4. Reads the `Display access information` task output from the install job and parses the AO
   URL, username, and password.
5. **AO**: logs in (retrying while AO finishes starting), then creates or updates:
   - a `Demo AAP` credential holding the demo AAP username and password
   - a global `Demo AAP` integration holding the demo AAP URL, using that credential for
     health checks. The run then triggers the integration's health check and prints the result.
6. **AO**: imports four workflows into the `Default` project and points every AAP job template
   step at the `Demo AAP` integration and credential:
   - [disk-demo-101.json](https://raw.githubusercontent.com/ansible-tmm/aap-orchestrator-demos/main/disk-utilization/ao/disk-demo-101.json) as *Disk Utilization Demo 101*
   - [rhel-cve-remediation.json](https://raw.githubusercontent.com/ansible-tmm/aap-orchestrator-demos/main/cve-remediation/ao/rhel-cve-remediation.json) as *RHEL CVE Remediation - Intelligent Patching*
   - [workflows/build-ee.json](workflows/build-ee.json) as *Build-EE*, from this repo
   - [workflows/fleet-health-demo.json](workflows/fleet-health-demo.json) as *Fleet Health Demo*,
     from this repo, **published** so it's ready to run

   Job template IDs are specific to the AAP a workflow was exported from, so when a step names
   its template, the ID is dropped and the step resolves the template by name.

The last task prints the AO URL, username, and password.

## Run it from aap.gregsowell.com

1. Push this directory to a Git repo and add it as a **Project**.
2. Create a **Job template**:
   - Inventory: any inventory with `localhost` (for example, *Demo Inventory*)
   - Playbook: `ao_setup.yml`
   - Credentials: none
   - Timeout: `0`, or at least 3 hours
3. Add a **Survey**:

   | Question | Variable | Type | Required |
   |---|---|---|---|
   | Demo AAP URL | `demo_aap_url` | Text | Yes |
   | Demo AAP admin password | `demo_aap_password` | Password | Yes |

The execution node needs outbound HTTPS to the demo AAP, the AO route, and
`raw.githubusercontent.com`. The demo AAP needs outbound HTTPS to `github.com` to sync the
`AO Demo Content` project.

From a CLI instead:

```bash
ansible-playbook ao_setup.yml -e demo_aap_url=https://aap.apps.cluster-xxxx.example.com -e demo_aap_password='...'
```

## Fleet Health Demo

A workflow that actually runs end to end, so AO has execution data to show. Every step is a
small playbook in [playbooks/fleet](playbooks/fleet) that works on simulated data, runs on
`localhost` in the execution environment, and passes its results to the next step as job
artifacts.

| Step | Job template | What it does |
|---|---|---|
| Collect Fleet Metrics | `AO Demo \| Collect Fleet Metrics` | Generates CPU, memory, and disk usage for each simulated host. |
| Analyze Fleet Metrics | `AO Demo \| Analyze Fleet Metrics` | Scores each host, computes averages, and sets the fleet status to `ok`, `degraded`, or `critical`. |
| Remediate Flagged Hosts | `AO Demo \| Remediate Hosts` | Applies a fix per host and records before and after usage. Hosts at 98% or higher need manual intervention. |
| Publish Fleet Report | `AO Demo \| Publish Fleet Report` | Builds a markdown report and sets the final result to `all_clear`, `recovered`, or `escalate`. |
| Route by Result | *(switch)* | Picks the notification from the final result. |
| Notify: All Clear / Auto-Remediated / Page On-Call | `AO Demo \| Send Notification` | Posts a simulated message to `#ops-reports`, `#ops-alerts`, or `#oncall`. |

Run it from **Workflows > Fleet Health Demo**. The trigger asks for:

| Input | Default | Effect |
|---|---|---|
| `scenario` | `degraded` | `healthy` ends all clear, `degraded` recovers automatically, `critical` escalates to on-call, and `random` can land on any of the three. |
| `host_count` | `12` | Number of simulated hosts, 3 to 50. |

Each run takes about a minute. The job templates prompt for extra variables on launch, so they
can also be run on their own from the demo AAP; each playbook documents its inputs at the top.

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `demo_aap_url` | *required* | Demo AAP URL. Any path is stripped; `https://` is added if missing. |
| `demo_aap_password` | *required* | Demo AAP admin password. Not needed when `demo_aap_token` is set. |
| `demo_aap_token` | *(unset)* | AAP OAuth token to use instead of a password. API calls send it as a Bearer header, and the AO credential stores it as `oauth_token`. |
| `demo_aap_username` | `admin` | Demo AAP user. Ignored when a token is used. |
| `demo_aap_validate_certs` | `false` | Verify TLS for the demo AAP. |
| `run_apd_setup` | `true` | Set `false` on reruns to skip the APD setup job. |
| `run_ao_install` | `true` | Set `false` to skip the install and read access details from the last successful install job. |
| `aap_job_poll_delay` / `aap_job_poll_retries` | `30` / `240` | Job wait interval and attempts. |
| `demo_content_enabled` | `true` | Create the Fleet Health Demo project, inventory, and job templates on the demo AAP. |
| `demo_content_organization` | `Default` | Demo AAP organization that owns the demo content. |
| `demo_content_project` | `AO Demo Content` | Name of the project that syncs this repository. |
| `demo_content_scm_url` / `demo_content_scm_branch` | this repo / `main` | Where the demo playbooks come from. |
| `demo_content_inventory` | `AO Demo Inventory` | Name of the inventory holding `localhost`. |
| `demo_content_job_templates` | the five templates above | Job template names and playbooks. The names must match the workflow's steps. |
| `ao_url` | *(discovered)* | Set to configure an AO this playbook didn't install, such as one on MicroShift. Skips reading the install job. |
| `ao_username` / `ao_password` | `admin` / *(discovered)* | Credentials for that existing AO. |
| `ao_validate_certs` | `false` | Verify TLS for AO. |
| `ao_project_name` | `Default` | AO project that owns the credential and workflows. |
| `ao_aap_credential_name` | `Demo AAP` | Name of the AAP credential created in AO. |
| `ao_aap_integration_name` | `Demo AAP` | Name of the AAP integration created in AO. TLS verification follows `demo_aap_validate_certs`. |
| `ao_aap_credential_inputs` | *(discovered)* | Override the credential inputs dict if the field mapping is wrong. |
| `ao_workflows` | the four workflows above | List of workflows to import. Each entry has a `url` or a `file` (relative to the playbook directory), plus optional `name` (defaults to the name in the JSON), `description`, and `publish`. |
| `ao_publish_workflows` | `false` | Publish workflows that don't set their own `publish`. A publish failure only warns. |

## Tags

| Tag | Runs |
|---|---|
| `full` | Everything: the APD setup, the AO install, the demo content, and the AO configuration. Same as running with no tags. |
| `aoconfig` | Everything after the AO install: the demo content on the demo AAP, reading the access details, then the credential, integration, and workflows. |

`aoconfig` skips both demo AAP jobs and reads the AO URL and password from the **last successful**
`Infrastructure | Automation Orchestrator | Install` job, so AO must already be installed. It
still runs the short setup tasks that check the required variables and find the demo AAP API.

```bash
ansible-playbook ao_setup.yml --tags aoconfig -e demo_aap_url=... -e demo_aap_password=...
```

On a job template, set **Job tags** to `aoconfig` (or enable *Prompt on launch* for job tags).

## Point it at an existing automation orchestrator

Pass `ao_url` and `ao_password` to configure an AO this playbook didn't install, such as one
running on MicroShift. The demo AAP variables still say which AAP the credential, integration,
and demo content are built for:

```bash
ansible-playbook ao_setup.yml --tags aoconfig \
  -e ao_url=https://ao.example.com -e ao_password='...' \
  -e demo_aap_url=https://aap.example.com -e demo_aap_token='...'
```

`demo_aap_token` is an AAP OAuth token (create one under **Access Management > Users > your
user > Tokens**, scope `write`). It replaces `demo_aap_password` everywhere: the controller API
calls send it as a Bearer header, and the AO credential stores it in `oauth_token` rather than a
username and password. Keep credentials out of shell history with a vars file:
`-e @creds.yml`.

AO must be allowed to reach that AAP. On a private network, set
`APP_INTEGRATION_URL_ALLOWED_HOSTS` (a JSON list containing the AAP and AO hostnames) and
`APP_OIDC_ALLOW_PRIVATE_NETWORKS=true` on the AO deployments, which is what the Red Hat demo
platform does for its own installs.

## Reruns

The playbook can be rerun safely: the demo project, inventory, job templates, credential, and
integration are updated in place, and a workflow that already exists (matched by name) gets a
new draft version instead of a duplicate. `run_apd_setup=false` and `run_ao_install=false` still
skip either demo AAP job individually under `full`.

## After it runs

- **Fleet Health Demo is published** and runs as soon as the build call finishes.
- **The other workflows import as drafts.** They won't pass publish validation until their
  dependencies exist:
  - The disk demo calls job templates that the infrastructure demo doesn't create:
    *Disk Utilization Check*, *Linux - Remediate - Disk Cleanup / Disk Expand / Continue*,
    *Disk Utilization - Fallback*, and *Notify Chatroom*.
  - The CVE demo calls *CVE - Fetch and Commit*, *CVE - Sync and Deploy Remediation*, and
    *CVE - Notify Mattermost Investigation*. Its Task agent steps also need an LLM provider
    and the Lightspeed and AAP MCP integrations.
  - Build-EE is a visual demo: its four AAP steps all call *Demo Job Template*, and its AI
    repair step needs an LLM provider.

## Built against

AO 2026.8 REST API documentation and a live instance's OpenAPI spec
(`/api_docs/v1/openapi.json`). The demo AAP stages, AO login, and the project and credential
type lookups have run against a live instance. Check these on the next run:

- On AO 2026.8 the AAP credential type holds auth only (`username`, `password`, `oauth_token`),
  and the AAP URL and TLS settings live on the integration. The docs still list "AAP Host" and
  "Verify SSL" credential fields, so the playbook reads the credential type's fields at run
  time and sends **only** the fields it declares. It fills in a host field only on builds that
  have one. If the username and password can't be mapped, the run stops and prints the schema;
  set `ao_aap_credential_inputs` and rerun.
- Every workflow definition is validated against that instance's `WorkflowDefinition` schema
  before it ships, but integration creation, workflow import, and publishing haven't run live yet.
- The demo content uses the automation controller API (projects, inventories, job templates)
  and hasn't run against the demo AAP yet. The Fleet Health Demo playbooks and their wiring
  were checked by running each scenario through Ansible's templating engine, not as AAP jobs.
- AO passes artifacts to the next step's extra vars. The playbooks accept structured values
  either as objects or as JSON or YAML strings, since the exact form isn't documented.
- The default project is assumed to be named `Default`. If it isn't found, the run fails and
  lists the available project names.
- Publishing reads the `version` field from `GET /workflows/{id}/versions`.
- AO's Swagger UI at `https://<ao-host>/api_docs/v1/docs` is the authoritative schema.

## Extending

Users, groups, and permissions map to `POST /api/v1/users`, `POST /api/v1/groups`,
`POST /api/v1/groups/{id}/members`, and `POST /api/v1/role_assignments`, using the same login
token as the tasks above.
