# AAP Config-as-Code

This directory provisions every **non-secret** AAP object the Infoblox
DNS-from-CSV solution needs: **organization, execution environment,
project, inventory (plus localhost), both custom credential types, and
the job template** — using the **`ansible.platform`** collection that
ships with AAP 2.5+.

Credential **instances** (the secret-bearing objects) are deliberately
out of scope. AAP itself encrypts and stores those — see [Where do the
secrets go?](#where-do-the-secrets-go) below.

## Quick start

```sh
# 1. Install collections (pulls in ansible.platform).
ansible-galaxy collection install -r ../../collections/requirements.yml

# 2. Tell the modules where the controller is.
export CONTROLLER_HOST=https://aap.example.com
export CONTROLLER_OAUTH_TOKEN=********           # or USERNAME/PASSWORD

# 3. Edit object names, EE image, and Git URL to match your environment.
$EDITOR aap/cac/group_vars/all/objects.yml

# 4. Apply.
ansible-playbook aap/cac/configure_aap.yml

# 5. (One-time, manual) Create Infoblox credentials in the AAP UI.
#    Resources → Credentials → Add → choose
#       "Infoblox NIOS Grid"   or   "Infoblox Universal DDI"
#    The credential types this playbook just installed will appear in
#    that dropdown. Fill in the host / user / password / API key. AAP
#    encrypts the values; nobody reads them from disk again.

# 6. Launch the job template — operator picks the credential at launch.
```

## What this creates

| AAP object | Source of truth |
|---|---|
| Organization | `group_vars/all/objects.yml` → `aap_organization` |
| Execution Environment | `group_vars/all/objects.yml` → `aap_execution_environment` |
| Project (this repo) | `group_vars/all/objects.yml` → `aap_project` |
| Inventory + localhost | `group_vars/all/objects.yml` → `aap_inventory` |
| Credential Type — NIOS | `aap/credential_type_nios.yml` (loaded at runtime) |
| Credential Type — UDDI | `aap/credential_type_universal_ddi.yml` (loaded at runtime) |
| Job Template | `group_vars/all/objects.yml` → `aap_job_template` |

The two existing `aap/credential_type_*.yml` files remain authoritative
for the credential-type schemas — `configure_aap.yml` loads them via
`include_vars`, so the AAP-import path (Administration → Credential
Types → Import) and the CaC path stay in lock-step.

## Where do the secrets go?

**Into AAP, not into Git.** The two custom credential types installed
by this playbook each define an `injectors:` block that maps user-typed
fields to the env vars the Infoblox collections read at runtime
(`NIOS_HOST`, `NIOS_USERNAME`, `NIOS_PASSWORD`, `INFOBLOX_PORTAL_KEY`,
…). The flow:

```
Admin in AAP UI ──▶ creates a credential of type "Infoblox NIOS Grid"
                    fills in host / user / password
                    AAP encrypts inputs in its DB (Fernet)
                          │
Operator launches JT ─────┘
       │
       └─▶ AAP injects the env vars into the EE container
                          │
                          └─▶ infoblox.nios_modules picks them up
                              automatically (no `provider:` arg needed)
```

This means:

- **No vault password to manage.** No `ansible-vault encrypt` step.
- **No secret material in this Git repo.** Ever.
- **Per-environment credentials are trivial.** Create
  `Infoblox NIOS — dev`, `Infoblox NIOS — prod`, etc. in AAP. The JT
  has `ask_credential_on_launch: true`, so the operator picks at launch.
- **RBAC is AAP's.** Who can use which Infoblox endpoint is governed
  by AAP credential ownership/permissions, not file-permissions on a
  vaulted YAML.

## Idempotency

Every task uses `state: present`, and the `ansible.platform` modules
upsert by name — re-running the playbook converges to the declared
state. To remove an object, delete it from the YAML *and* set its state
to `absent`, or remove it via the UI (CaC will not re-create it unless
you re-list it).

## Layout

```
aap/cac/
├── README.md                       ← you are here
├── configure_aap.yml               ← entrypoint playbook
└── group_vars/all/
    ├── aap.yml                     ← controller endpoint + auth (env-driven)
    └── objects.yml                 ← Org, EE, Project, Inventory, JT
```
