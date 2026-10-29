# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Does

Manages a Cisco AI Pod data center fabric (on Nexus Dashboard 4.3 / NDFC) using the **Network as Code** approach. Network engineers declare desired fabric state in YAML files; the `cisco.nac_dc_vxlan` Ansible collection translates those files into NDFC API calls.

Three fabrics: **FEN** (VXLAN EVPN front-end, collection-managed), **BEN** (AI/GPU routed, script-managed), **Admin** (OOB Classic LAN, script-managed).

## Environment Setup

**Python 3.12 is required.** Python 3.10 and 3.11 also work; 3.13+ is not supported by `ansible-core==2.16.x`.

```bash
uv venv --python 3.12
source .venv/bin/activate
uv pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yaml
```

## Key Commands

```bash
# Validate YAML via Ansible role (role-level checks)
ansible-playbook validate.yaml

# Validate YAML against schema (if schemas/ is available from the collection)
nac-validate host_vars/ --schema schemas/ndfc_schema.yaml

# Dry-run deploy of FEN
ansible-playbook vxlan.yaml --check

# Deploy FEN to NDFC
ansible-playbook vxlan.yaml
```

## Required Environment Variables

```bash
export ND_HOST=<your-nexus-dashboard-ip>   # no https://
export ND_USERNAME=admin
export ND_PASSWORD=<password>
export ND_DOMAIN=local                      # or LDAP domain
export NDFC_SW_USERNAME=admin               # switch credentials
export NDFC_SW_PASSWORD=<password>
# Optional: export ND_FQDN=ndfc.example.com  # if cert uses FQDN but ND_HOST is IP
```

`inventory.yaml` and `group_vars/ndfc/connection.yml` resolve these via Ansible `lookup('env', ...)` with safe defaults.

## Architecture

### Source of Truth: `host_vars/<fabric>/`

**FEN** (collection-managed) — files use `*.nac.yaml` naming:

| File | Purpose |
|------|---------|
| `global.nac.yaml` | Fabric type, BGP ASN, anycast GW MAC, DNS/NTP |
| `underlay.nac.yaml` | OSPF underlay, multicast replication, IP ranges |
| `topology_switches.nac.yaml` | Switch inventory (serial numbers, roles, mgmt IPs) |
| `topology_interfaces_all_types.nac.yaml` | Host-facing interface templates |
| `networks.nac.yaml` | L2/L3 overlay segments (VLANs, VNIs, subnets) |
| `vrfs.nac.yaml` | Tenant VRF definitions |
| `policy.nac.yaml` | TCAM and switch policies |
| `fabric_ext.nac.yaml` | nvPairs gap-fill applied via REST API (CDP, NGOAM) |

**BEN and Admin** (script-managed) — single `fabric_provision.yaml` per fabric.

All FEN files share a common top-level key: `vxlan:`.

### Ansible Roles (in `vxlan.yaml`)

The deploy playbook runs two roles in sequence against FEN only:
1. **`cisco.nac_dc_vxlan.dtc.create`** — prepares configuration objects in NDFC (idempotent)
2. **`cisco.nac_dc_vxlan.dtc.deploy`** — pushes pending config from NDFC to switches

The `dtc.remove` role is intentionally commented out — uncomment only when deliberately removing fabric config.

### Key Behavior Settings (`group_vars/ndfc/ndfc.yaml`)

- `force_run_all: true` — every run is a full push (not differential). Set to `false` for incremental/diff runs.
- All `*_delete_mode` flags control what gets cleaned up when using the remove role.

## CI/CD Pipeline

**`Jenkinsfile`** — full AI pod pipeline on `main`:

```
Setup → Validate → Provision Non-NAC Fabrics → Deploy → Configure Extended → Wait Managed → Test
```

Non-`main` branches run Setup + Validate only (no NDFC changes).

### Jenkins Credentials Required (type: Secret text)

`ND_HOST`, `ND_USERNAME`, `ND_PASSWORD`, `NDFC_SW_USERNAME`, `NDFC_SW_PASSWORD`, `webex-chat-ops-credentials`

**Must be "Secret text" type**, not "Username with password" — the latter sets `<ID>_USR`/`<ID>_PSW` env vars, not the bare name Ansible expects.

## Ansible Configuration (`ansible.cfg`)

- Inventory: `inventory.yaml`
- Collections path: `./collections/ansible_collections/`
- Log: `./ansible.log`
- Connection/command timeout: 7200s (2 hours — NDFC deploys are slow)
- `host_key_checking = False` (NDFC uses self-signed certs; `ansible_httpapi_validate_certs: false` is set in `group_vars`)
