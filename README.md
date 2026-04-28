# Infoblox DNS-from-CSV (AAP-ready)

Manage Infoblox DNS records from a CSV file maintained in this repository,
using the certified Ansible collections and packaged for **Ansible Automation
Platform** (not bare `ansible-core`).

Supports both Infoblox platforms via two custom AAP credential types:

| Platform | Collection | AAP Credential Type |
|---|---|---|
| On-prem NIOS Grid | `infoblox.nios_modules` | `aap/credential_type_nios.yml` |
| Cloud / Universal DDI | `infoblox.universal_ddi` | `aap/credential_type_universal_ddi.yml` |

## Repository layout

```
.
├── README.md                          ← you are here
├── ansible.cfg                        ← galaxy server list + roles_path
├── collections/requirements.yml       ← certified collections to install
├── inventory/hosts.yml                ← localhost (the role runs against the API, not nodes)
├── playbooks/manage_dns.yml           ← AAP job-template entrypoint
├── files/dns_records.csv              ← THE DATA — edit via PR
├── execution-environment/             ← EE definition for ansible-builder
├── aap/                               ← custom credential type YAMLs to import
│   ├── credential_type_nios.yml
│   └── credential_type_universal_ddi.yml
└── roles/
    └── infoblox_dns_csv/              ← the role doing the work
        ├── README.md                  ← role-level docs (CSV schema, vars, flow)
        ├── defaults/main.yml
        ├── vars/main.yml
        ├── meta/main.yml
        ├── tasks/
        │   ├── main.yml               ← read CSV, validate, dispatch, summarise
        │   ├── _validate_row.yml      ← ⚠ contribution point #1
        │   ├── nios.yml               ← NIOS dispatch loop
        │   ├── _nios_record.yml       ← per-record NIOS module calls (⚠ contribution point #2 — SRV, NAPTR)
        │   ├── universal_ddi.yml      ← UDDI dispatch loop
        │   └── _uddi_record.yml       ← per-record UDDI dns_record call
        └── templates/summary.j2
```

## How the dual-mode dispatch works

1. The operator attaches **one** of the two custom credential types to the
   job template in AAP.
2. Each credential type sets `infoblox_platform` to `nios` or
   `universal_ddi` via `injectors.extra_vars` — see `aap/credential_type_*.yml`.
3. The role's `tasks/main.yml` does
   `include_tasks: "{{ infoblox_platform }}.yml"`, so the right backend
   runs with no UI prompting.
4. The CSV format is identical for both platforms — the role does the
   per-platform parameter shaping internally.

## Bootstrapping AAP

1. **Add this repo as a Project** (Source Control → Git → this repo URL).
2. **Build/import the EE** from `execution-environment/execution-environment.yml`
   (`ansible-builder build -t infoblox-ee:latest …`), push to your registry,
   register it in AAP under *Execution Environments*.
3. **Import both credential types** under *Administration → Credential Types →
   Import* using the two YAML files in `aap/`.
4. **Create credentials** of those types — one per Infoblox endpoint.
5. **Create a job template** that points at `playbooks/manage_dns.yml`,
   uses the EE, and prompts for / has attached one Infoblox credential.

## Running locally (for a smoke test only)

```sh
# install certified collections from public Galaxy (see ansible.cfg
# for how to point this at console.redhat.com instead)
ansible-galaxy collection install -r collections/requirements.yml

# dry run — prints what it WOULD do without calling Infoblox
ansible-playbook playbooks/manage_dns.yml \
  -e infoblox_platform=nios \
  -e dns_dry_run=true
```

## Editing the CSV

Edit `files/dns_records.csv` and open a PR. The PR review IS the change-control
gate. Schema is documented in [`roles/infoblox_dns_csv/README.md`](roles/infoblox_dns_csv/README.md).

## Status

- [x] Repo skeleton, ansible.cfg, requirements, EE, inventory
- [x] Both AAP credential types
- [x] Validation + dispatch + summary
- [x] NIOS modules wired for A, AAAA, CNAME, HOST, PTR, TXT, MX, SRV, NAPTR
- [x] UDDI: all record types routed through `dns_record`
- [x] Per-record-type validation rules for A, AAAA, CNAME, PTR, MX, SRV, HOST, NAPTR

## Column convention for multi-field record types

The validator and the NIOS dispatch agree on the same `extra1..extra5` mapping, so SRV and NAPTR rows are unambiguous:

| Type  | `name`      | `value`       | `priority` | `extra1`     | `extra2` | `extra3`   | `extra4`   |
|-------|-------------|---------------|------------|--------------|----------|------------|------------|
| SRV   | service FQDN | target host   | priority   | weight       | port     | —          | —          |
| NAPTR | NAPTR FQDN  | replacement   | order      | preference   | flags    | services   | regexp     |
| MX    | mail domain | exchanger     | preference | —            | —        | —          | —          |
| PTR   | reverse FQDN | ptrdname     | —          | ipv4addr*    | ipv6addr*| —          | —          |

\* PTR `extra1`/`extra2` are optional — only set them if you want to pin the explicit forward IP on the record.
