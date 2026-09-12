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
4. **AO**: logs in (retrying while AO finishes starting), then creates or updates an
   Ansible Automation Platform credential named `Demo AAP` that points at the demo AAP.
5. **AO**: imports two workflows into the `Default` project and points every AAP job template
   step at the `Demo AAP` credential:
   - [disk-demo-101.json](https://raw.githubusercontent.com/ansible-tmm/aap-orchestrator-demos/main/disk-utilization/ao/disk-demo-101.json) as *Disk Utilization Demo 101*
   - [rhel-cve-remediation.json](https://raw.githubusercontent.com/ansible-tmm/aap-orchestrator-demos/main/cve-remediation/ao/rhel-cve-remediation.json) as *RHEL CVE Remediation - Intelligent Patching*

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
| `ao_aap_credential_inputs` | *(discovered)* | Override the credential inputs dict if the field mapping is wrong. |
| `ao_workflows` | the two workflows above | List of `{name, url, description?}` to import. |
| `ao_publish_workflows` | `false` | Try to publish after import. A publish failure only warns. |
| `show_ao_password` | `false` | Print the AO password in the summary. |

## Reruns

The playbook can be rerun safely: the credential is updated in place, and a workflow that
already exists (matched by name) gets a new draft version instead of a duplicate.

```bash
ansible-playbook ao_setup.yml -e demo_aap_url=... -e demo_aap_password=... -e run_apd_setup=false -e run_ao_install=false
```

## After it runs

- **Add the AAP integration.** AAP job template steps select an integration in the builder,
  and the AO docs don't publish the API fields for creating one, so this step is manual. In AO,
  go to **Configuration > Integrations > Configure integration > Ansible Automation Platform**.
  Set the AAP URL to the demo AAP URL and the health check credential to `Demo AAP`.
- **Workflows import as drafts.** They won't pass publish validation until their dependencies
  exist:
  - The disk demo calls job templates that the infrastructure demo doesn't create:
    *Disk Utilization Check*, *Linux - Remediate - Disk Cleanup / Disk Expand / Continue*,
    *Disk Utilization - Fallback*, and *Notify Chatroom*.
  - The CVE demo calls *CVE - Fetch and Commit*, *CVE - Sync and Deploy Remediation*, and
    *CVE - Notify Mattermost Investigation*. Its Task agent steps also need an LLM provider
    and the Lightspeed and AAP MCP integrations.

## Built against

AO 2026.8 REST API documentation. It has not been tested against a live AO instance yet, so
check these on the first run:

- The AAP credential type's input field names are not in the docs or the OpenAPI spec
  (`inputs` is a free-form object validated server-side), so the playbook reads them from
  `GET /credential_types/{id}` at run time, prints them, and sends **only** field names the
  type declares. If it cannot identify the AAP URL field, the run stops and prints the schema;
  set `ao_aap_credential_inputs` to the correct names and rerun.
- The default project is assumed to be named `Default`. If it isn't found, the run fails and
  lists the available project names.
- Publishing reads the `version` field from `GET /workflows/{id}/versions`.
- AO's Swagger UI at `https://<ao-host>/api_docs/v1/docs` is the authoritative schema.

## Extending

Users, groups, and permissions map to `POST /api/v1/users`, `POST /api/v1/groups`,
`POST /api/v1/groups/{id}/members`, and `POST /api/v1/role_assignments`, using the same login
token as the tasks above.
