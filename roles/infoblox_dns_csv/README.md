# infoblox_dns_csv

Ansible role that reads a CSV file and creates / updates / deletes DNS records
in Infoblox. Supports both certified collections:

| Platform | Collection | Module(s) used |
|---|---|---|
| On-prem NIOS Grid | `infoblox.nios_modules` | `nios_a_record`, `nios_aaaa_record`, `nios_cname_record`, `nios_host_record`, `nios_mx_record`, `nios_naptr_record`, `nios_ptr_record`, `nios_srv_record`, `nios_txt_record` |
| Infoblox Cloud / Universal DDI | `infoblox.universal_ddi` | `dns_record` (single module, `type` field) |

The platform is selected by the AAP custom credential the operator attaches to
the job template — see `aap/credential_type_*.yml` in the repo root.

## CSV schema

```
record_type,name,zone,view,ttl,state,value,priority,extra1,extra2,extra3,extra4,extra5,comment
```

| Column | Required | Notes |
|---|---|---|
| `record_type` | yes | One of `A`, `AAAA`, `CNAME`, `HOST`, `PTR`, `MX`, `SRV`, `TXT`, `NAPTR` |
| `name` | yes | FQDN (or PTR name like `10.0.10.10.in-addr.arpa`) |
| `zone` | UDDI only | The parent zone — UDDI splits FQDN into `name_in_zone` + `zone` |
| `view` | no | DNS view; falls back to `dns_default_view` (default `"default"`) |
| `ttl` | no | Integer seconds; blank inherits zone default |
| `state` | no | `present` (default) or `absent` |
| `value` | yes (most types) | The RHS — IPv4 for A, IPv6 for AAAA, FQDN for CNAME / MX / SRV target, text for TXT, etc. |
| `priority` | MX, SRV, NAPTR | Preference / priority / order |
| `extra1..5` | type-specific | Used for SRV (weight/port), NAPTR (preference/flags/services/regexp), PTR (ipv4addr/ipv6addr explicit) |
| `comment` | no | Free-text comment stored on the record |

See `files/dns_records.csv` for a worked example covering every supported type.

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `infoblox_platform` | `nios` | `nios` or `universal_ddi`. Normally injected by the AAP credential. |
| `dns_csv_path` | `{{ playbook_dir }}/../files/dns_records.csv` | Path to the CSV file. |
| `dns_default_view` | `default` | View used when a CSV row leaves `view` blank. |
| `dns_default_ttl` | `null` | TTL used when `ttl` blank; `null` = inherit zone default. |
| `dns_dry_run` | `false` | When `true`, prints intended args without calling the API. |
| `dns_warn_only` | `false` | When `true`, the role exits 0 even if some rows failed. |
| `dns_supported_record_types` | `[A, AAAA, CNAME, HOST, PTR, MX, SRV, TXT, NAPTR]` | Record types the validator accepts. |

## Run flow

1. **Validate** — `tasks/_validate_row.yml` runs per-row asserts. Failures are captured but don't abort the run.
2. **Dispatch** — based on `infoblox_platform`, hands off to `tasks/nios.yml` or `tasks/universal_ddi.yml`.
3. **Apply** — per-row include calls the matching module. Result is recorded.
4. **Summarise** — renders `summary.j2` to `dns_csv_summary.txt` and to job output.
5. **Decide** — fails the job if any row failed (unless `dns_warn_only: true`).

## Local smoke test

```sh
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook playbooks/manage_dns.yml -e dns_dry_run=true -e infoblox_platform=nios
```

A real run needs the credentials env vars set (NIOS) or `INFOBLOX_PORTAL_KEY`
exported (UDDI). In AAP both are handled by the custom credential type.
