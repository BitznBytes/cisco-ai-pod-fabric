# cisco-ai-pod-fabric

Network-as-Code configuration template for a Cisco AI Pod data center fabric, managed via the [`cisco.nac_dc_vxlan`](https://netascode.cisco.com/) Ansible collection against Nexus Dashboard 4.3 (NDFC).

## What's Included / Not Included

**Included:**
- `host_vars/` — fabric configuration files (YAML data model) with placeholder values to customize
- `group_vars/` — Ansible connection and role behavior settings
- `Jenkinsfile` — CI/CD pipeline skeleton
- Ansible playbooks: `vxlan.yaml`, `validate.yaml`, `security_groups.yaml`
- `requirements.txt` / `requirements.yaml` — Python and collection dependencies

**Not included** (obtain separately or implement for your environment):
- `schemas/` and `defaults/` — required by `nac-validate` and `nac-test`; provided with the `cisco.nac_dc_vxlan` collection
- `jinja_filters/` — custom Jinja2 filters for `nac-test`; part of the collection
- Operational scripts (`ndfc_provision_fabrics.py`, `ndfc_apply_fabric_ext.py`, `ndfc_wait_managed.py`) — implement these using the NDFC REST API for your environment
- `security/` — SGT/GPO role content required by `security_groups.yaml`

---

## Fabrics

| Fabric | Template | Role | Switches |
|--------|----------|------|----------|
| **FEN** | VXLAN EVPN (iBGP, ASN 65101) | Front-end compute + storage overlay | 2 spines, 8 leaves |
| **BEN** | Easy_Fabric_eBGP (eBGP, ASN 65201, EVPN off) | Back-end GPU routed fabric — RoCEv2/PFC/ECN via AI_Fabric_QOS_400G | 2 spines, 4 leaves |
| **Admin** | Easy_Fabric_Classic | Out-of-band management | 2 switches |

**FEN** is fully managed by the Ansible collection. **BEN** and **Admin** use NDFC templates not supported by the collection and are provisioned via custom scripts against the NDFC REST API.

---

## Prerequisites

**Python 3.12 is required.** Python 3.10 and 3.11 also work; 3.13+ is not supported by `ansible-core==2.16.x`.

```bash
uv venv --python 3.12
source .venv/bin/activate
uv pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yaml
```

**Required environment variables:**

```bash
export ND_HOST=<your-nexus-dashboard-ip>   # IP or hostname only — no https://
export ND_USERNAME=admin
export ND_PASSWORD=<password>
export ND_DOMAIN=local                      # or your LDAP domain
export NDFC_SW_USERNAME=admin               # switch credentials for NDFC
export NDFC_SW_PASSWORD=<password>
```

---

## Configuration — Customize for Your Environment

Before deploying, update the placeholder values in `host_vars/`:

1. **Switch serial numbers** — replace `SERIAL-*` placeholders in `topology_switches.nac.yaml` and `fabric_provision.yaml` with your actual Nexus switch serial numbers.
2. **Management IP addresses** — replace `192.0.2.x` addresses with your real OOB management subnet.
3. **Overlay IPs** — replace `192.0.2.x` / `198.51.100.x` gateway addresses in `networks.nac.yaml` with your site-specific overlay subnets.
4. **DNS / NTP servers** — update `global.nac.yaml` with your environment's servers.

---

## Full Deployment — From Scratch

Run these steps in order. The pipeline automates steps 3–6 on `main`; step 2 (switch discovery) is a one-time manual operation.

### Step 1 — Validate the data model

Always run validation before touching NDFC. This checks FEN's `*.nac.yaml` files against the schema and runs the collection's role-level validation.

```bash
nac-validate host_vars/
ansible-playbook validate.yaml
```

> If you have the `schemas/ndfc_schema.yaml` from the collection, add `--schema schemas/ndfc_schema.yaml` to the `nac-validate` command for stricter checking.

### Step 2 — Provision BEN and Admin fabrics

Creates the `Easy_Fabric_eBGP` (BEN) and `Easy_Fabric_Classic` (Admin) fabrics in NDFC via REST API. Implement a provisioning script using the fabric definitions in `host_vars/BEN/fabric_provision.yaml` and `host_vars/Admin/fabric_provision.yaml`.

After the fabrics exist in NDFC, discover switches into them — either via the NDFC GUI or a custom discovery script.

### Step 3 — Deploy FEN

Runs the collection's `dtc.create` then `dtc.deploy` roles to push FEN configuration to NDFC and from NDFC to the switches.

```bash
ansible-playbook vxlan.yaml
```

### Step 4 — Apply extended fabric settings

Pushes nvPairs not exposed by the collection (CDP, NGOAM, etc.) to FEN via the NDFC REST API. These settings are defined in `host_vars/FEN/fabric_ext.nac.yaml`. Implement a script to POST them to the NDFC fabric API, or apply them manually via the NDFC GUI.

### Step 5 — Wait for switches to reach managed state

Poll NDFC until all FEN switches are In-Sync / managedMode before running tests. Implement using the NDFC REST API endpoint:
`GET /appcenter/cisco/ndfc/api/v1/lan-fabric/rest/control/fabrics/{fabric}/inventory/manageable`

### Step 6 — Run post-deploy tests

Validates the deployed configuration against the data model using `nac-test`. Requires `defaults/` and `jinja_filters/` from the collection. Results are written to `tests/results/vxlan/FEN/log.html`.

```bash
nac-test -d host_vars/FEN -d defaults -t tests/templates -f jinja_filters \
         -o tests/results/vxlan/FEN
```

---

## CI/CD Pipeline

The `Jenkinsfile` automates the full sequence on `main`. Non-`main` branches run **Setup + Validate only** (no NDFC changes).

```
All branches:  Setup → Validate
main only:     Setup → Validate → Provision Non-NAC Fabrics → Deploy
                     → Configure Extended → Wait Managed → Test
```

**Jenkins credentials required** (all must be `Secret text` type):

| Credential ID | Value |
|---|---|
| `ND_HOST` | Nexus Dashboard IP or hostname |
| `ND_USERNAME` | NDFC username |
| `ND_PASSWORD` | NDFC password |
| `NDFC_SW_USERNAME` | Switch username |
| `NDFC_SW_PASSWORD` | Switch password |
| `webex-chat-ops-credentials` | Webex bot token for notifications |

> **Important:** Use `Secret text` credentials, not `Username with password`. The latter sets `<ID>_USR`/`<ID>_PSW` env vars, which Ansible and scripts do not read.

Before running, customize the Jenkinsfile:
- Set `label 'YOUR-BUILD-AGENT-LABEL'` to your Jenkins agent label
- Set `WEBEX_ROOM_ID` to your Webex space ID
- Implement the script-dependent stages (Provision Non-NAC Fabrics, Configure Extended, Wait Managed)

---

## Optional — Security Groups (SGT/GPO)

`security_groups.yaml` is a standalone playbook for deploying Cisco Security Group Tags (SGT) / Group Policy Option (GPO) to FEN. It requires Ansible roles from the `security/` directory (not included in this template).

---

## Repository Layout

```
host_vars/
  FEN/                              # VXLAN EVPN — managed by collection
    global.nac.yaml                 #   fabric type, ASN, DNS/NTP
    underlay.nac.yaml               #   OSPF underlay, multicast, IP ranges
    topology_switches.nac.yaml      #   switch inventory with serials + mgmt IPs
    topology_interfaces_all_types.nac.yaml  # host-facing interface templates
    networks.nac.yaml               #   overlay VLANs/VNIs
    vrfs.nac.yaml                   #   tenant VRFs
    policy.nac.yaml                 #   TCAM and switch policies
    fabric_ext.nac.yaml             #   nvPairs gap-fill applied via REST API (CDP, NGOAM)
  BEN/
    fabric_provision.yaml           # AI/GPU eBGP fabric definition
  Admin/
    fabric_provision.yaml           # OOB Classic LAN fabric definition

group_vars/ndfc/
  connection.yml                    # Ansible connection parameters for NDFC
  ndfc.yaml                         # Collection role behavior settings

security_groups.yaml                # Optional playbook: deploy SGT/GPO to FEN
vxlan.yaml                          # Main deployment playbook (FEN)
validate.yaml                       # Validation playbook (FEN)
Jenkinsfile                         # CI/CD pipeline
```

---

## References

- [NAC Data Model docs](https://netascode.cisco.com/docs/data_models/)
- [NDFC REST API (ND 4.3)](https://developer.cisco.com/docs/nexus-dashboard-orchestrator/4-3-1/api-reference/)
- [cisco.nac_dc_vxlan collection](https://galaxy.ansible.com/ui/repo/published/cisco/nac_dc_vxlan/)
- [Cisco AI Pod CVD — VAST Data](https://www.cisco.com/c/en/us/td/docs/unified_computing/ucs/UCS_CVDs/cisco_aipod_vast_data.html)
