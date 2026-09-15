# Automation orchestrator demo setup

`ao_setup.yml` provisions automation orchestrator (AO) through a Red Hat Demo Platform
AAP instance and loads base configuration into it. It runs against `localhost` and uses only
`ansible.builtin` modules, so any execution environment works.

## What it does

1. **Demo AAP**: launches `APD | Single demo setup` with survey answer `demo: infrastructure`
   and waits for it to finish.
2. **Demo AAP**: launches `Infrastructure | Automation Orchestrator | Install` and waits
   (the install can take a long time; polling allows up to 2 hours).
3. Reads the `Display access information` task output from that install job and parses the AO
   URL, username, and password.
4. **AO**: logs in (retrying while AO finishes starting), then creates or updates:
   - a `Demo AAP` credential holding the demo AAP username and password
   - a global `Demo AAP` integration holding the demo AAP URL, using that credential for
     health checks. The run then triggers the integration's health check and prints the result.
5. **AO**: imports three workflows into the `Default` project and points every AAP job template
   step at the `Demo AAP` integration and credential:
   - [disk-demo-101.json](https://raw.githubusercontent.com/ansible-tmm/aap-orchestrator-demos/main/disk-utilization/ao/disk-demo-101.json) as *Disk Utilization Demo 101*
   - [rhel-cve-remediation.json](https://raw.githubusercontent.com/ansible-tmm/aap-orchestrator-demos/main/cve-remediation/ao/rhel-cve-remediation.json) as *RHEL CVE Remediation - Intelligent Patching*
   - [workflows/build-ee.json](workflows/build-ee.json) as *Build-EE*, from this repo

   Job template IDs are specific to the AAP a workflow was exported from, so when a step names
   its template, the ID is dropped and the step resolves the template by name.

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
`raw.githubusercontent.com`.

From a CLI instead:

```bash
ansible-playbook ao_setup.yml -e demo_aap_url=https://aap.apps.cluster-xxxx.example.com -e demo_aap_password='...'
```

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `demo_aap_url` | *required* | Demo AAP URL. Any path is stripped; `https://` is added if missing. |
| `demo_aap_password` | *required* | Demo AAP admin password. |
| `demo_aap_username` | `admin` | Demo AAP user. |
| `demo_aap_validate_certs` | `false` | Verify TLS for the demo AAP. |
| `run_apd_setup` | `true` | Set `false` on reruns to skip the APD setup job. |
| `run_ao_install` | `true` | Set `false` to skip the install and read access details from the last successful install job. |
| `aap_job_poll_delay` / `aap_job_poll_retries` | `30` / `240` | Job wait interval and attempts. |
| `ao_validate_certs` | `false` | Verify TLS for AO. |
| `ao_project_name` | `Default` | AO project that owns the credential and workflows. |
| `ao_aap_credential_name` | `Demo AAP` | Name of the AAP credential created in AO. |
| `ao_aap_integration_name` | `Demo AAP` | Name of the AAP integration created in AO. TLS verification follows `demo_aap_validate_certs`. |
| `ao_aap_credential_inputs` | *(discovered)* | Override the credential inputs dict if the field mapping is wrong. |
| `ao_workflows` | the three workflows above | List of workflows to import. Each entry has a `url` or a `file` (relative to the playbook directory), plus optional `name` (defaults to the name in the JSON) and `description`. |
| `ao_publish_workflows` | `false` | Try to publish after import. A publish failure only warns. |

## Tags

| Tag | Runs |
|---|---|
| `full` | Everything: the APD setup, the AO install, and the AO configuration. Same as running with no tags. |
| `aoconfig` | Only the AO configuration: reading the access details, then the credential, integration, and workflows. |

`aoconfig` skips both demo AAP jobs and reads the AO URL and password from the **last successful**
`Infrastructure | Automation Orchestrator | Install` job, so AO must already be installed. It
still runs the short setup tasks that check the required variables and find the demo AAP API.

```bash
ansible-playbook ao_setup.yml --tags aoconfig -e demo_aap_url=... -e demo_aap_password=...
```

On a job template, set **Job tags** to `aoconfig` (or enable *Prompt on launch* for job tags).

## Reruns

The playbook can be rerun safely: the credential and integration are updated in place, and a
workflow that already exists (matched by name) gets a new draft version instead of a duplicate.
`run_apd_setup=false` and `run_ao_install=false` still skip either demo AAP job individually
under `full`.

## After it runs

- **Workflows import as drafts.** They won't pass publish validation until their dependencies
  exist:
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
- The integration and workflow request bodies are checked against that instance's OpenAPI
  schemas, but integration creation and workflow import haven't run live yet.
- The default project is assumed to be named `Default`. If it isn't found, the run fails and
  lists the available project names.
- Publishing reads the `version` field from `GET /workflows/{id}/versions`.
- AO's Swagger UI at `https://<ao-host>/api_docs/v1/docs` is the authoritative schema.

## Extending

Users, groups, and permissions map to `POST /api/v1/users`, `POST /api/v1/groups`,
`POST /api/v1/groups/{id}/members`, and `POST /api/v1/role_assignments`, using the same login
token as the tasks above.
