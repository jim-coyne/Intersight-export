# Intersight Configuration Export / Import Workflow

Automates exporting all Cisco Intersight policies, pools, and profiles to JSON files and re-creating them in any target organization using Ansible playbooks.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Ansible | `ansible-playbook` on PATH |
| `cisco.intersight` collection | `ansible-galaxy collection install cisco.intersight` |
| `api-key.txt` | Your Intersight API Key ID (one line, no trailing newline) |
| `SecretKey.txt` | Your Intersight EC private key in PEM format |

Both credential files must be in the same directory as the playbooks.

---

## Step 1 — Pull all configurations from Intersight

This exports every policy, pool, and profile to `./exports/`.

```bash
ansible-playbook pull_intersight_config.yml
```

Expected output:
```
localhost : ok=65  changed=18  unreachable=0  failed=0
```

Each policy type is exported two ways:
- **Flat array file**: `exports/<type>.json` — a JSON array of all objects of that type (restore all)
- **Individual files**: `exports/<type>/<Name>.json` — one file per object, convenient for viewing and editing a single policy

---

## Step 2 — Choose and inspect a policy

List any policy type's exported files:

```bash
ls exports/bios_policies/        # BIOS policies
ls exports/ntp_policies/         # NTP policies
ls exports/boot_policies/        # Boot policies
ls exports/server_profiles/      # Server profiles
# etc.
```

Open a policy to inspect its fields:

```bash
cat exports/bios_policies/SMF.json | python3 -m json.tool | head -40
```

---

## Step 3 — Prepare and edit the push file

The push playbook reads `push/<type>.json` as a JSON **array** of objects.

### 3a. Copy one policy into the push folder

```bash
python3 -c "
import json
with open('exports/bios_policies/SMF.json') as f:
    policy = json.load(f)
with open('push/bios_policies.json', 'w') as f:
    json.dump([policy], f, indent=2)
print('Ready:', policy['Name'])
"
```

Replace `bios_policies` and `SMF.json` with any type and policy name from the table in [Supported Types](#supported-policy--object-types).

### 3b. Copy multiple policies into the push folder

To push several policies at once, read each individual file and combine them:

```bash
python3 -c "
import json, glob
policies = []
for path in glob.glob('exports/bios_policies/*.json'):
    with open(path) as f:
        policies.append(json.load(f))
with open('push/bios_policies.json', 'w') as f:
    json.dump(policies, f, indent=2)
print(f'Ready: {len(policies)} policies')
"
```

### 3c. Edit the policy before pushing

Open `push/bios_policies.json` (or whichever type file) in any text editor and change the fields you want.

**Fields you are safe to change:**

| Field | What it does | Example |
|---|---|---|
| `"Name"` | The name the policy will be created with in Intersight. **Must be unique** within the org. | `"SMF-Production"` |
| `"Description"` | Free-text description shown in the Intersight UI. | `"Modified for prod use"` |
| Any policy-specific setting | e.g. `"NtpServers"`, `"CpuPowerManagement"`, `"BootDevices"`, etc. | See field list below |

**Fields you must NOT change or remove** (the push playbook strips these automatically, but editing them manually can cause errors):

| Field | Reason |
|---|---|
| `"Moid"` | Instance ID — do not set this; the API assigns it on creation |
| `"AccountMoid"`, `"DomainGroupMoid"` | Account-scoped identifiers — auto-stripped |
| `"Organization"` | Overwritten by the playbook with the `target_org` value |
| `"Owners"`, `"PermissionResources"` | Auto-assigned on creation — auto-stripped |
| `"CreateTime"`, `"ModTime"` | Timestamps — auto-stripped |
| `"Profiles"` | Profile attachments — auto-stripped to avoid conflicts |
| `"Ancestors"`, `"SharedScope"` | Read-only metadata — auto-stripped |

> **Tip:** The safest approach is to only edit `"Name"`, `"Description"`, and the actual settings fields for the policy type. Leave everything else as exported.

### 3d. Quick reference — key settings fields by policy type

**BIOS Policy** (`bios_policies`):
Common fields to tune: `"CpuPowerManagement"`, `"CpuPerformance"`, `"IntelHyperThreadingTech"`, `"IntelTurboBoostTech"`, `"NumaOptimized"`, `"WorkLoadConfig"`.
Values are either a specific string (e.g. `"enabled"`, `"disabled"`) or `"platform-default"` to inherit the server default.

**NTP Policy** (`ntp_policies`):
Key field: `"NtpServers"` — a JSON array of NTP server IP addresses or hostnames.
```json
"NtpServers": ["10.0.0.1", "10.0.0.2"],
"Timezone": "America/New_York"
```

**Boot Policy** (`boot_policies`):
Key field: `"BootDevices"` — ordered array of boot device objects (PXE, local disk, etc.).

**NTP, SSH, KVM, SNMP, Syslog, Power, Thermal Policies**:
Edit the named settings fields. All other fields can remain as exported.

**LAN / SAN Connectivity Policies** (`lan_policies`, `san_policies`):
These are parent objects. The actual vNIC/vHBA interface objects live in `vnic_eth_ifs` / `vhba_fc_ifs` and reference the parent policy by Moid. Push the parent first, then the child interfaces.

**Pools** (`pools_ip`, `pools_mac`, `pools_uuid`, `pools_wwn`):
Key fields: `"IpV4Blocks"` / `"MacBlocks"` / `"UuidSuffixBlocks"` / `"FcBlocks"` — define the address ranges to allocate from.

**Server Profiles / Templates**:
These reference other policies by Moid. After pushing the dependent policies, update the Moid references in the profile JSON to match the newly created objects before pushing profiles.

---

## Step 4 — Push to Intersight

```bash
ansible-playbook push_intersight_config.yml \
  -e "target_org=default" \
  -e "push_types=bios_policies" \
  -e "import_dir=$(pwd)/push" \
  -e "name_suffix=-NEW"
```

The `name_suffix` is appended/prepended to the `"Name"` field from the JSON file. This lets you create a copy without editing the file at all. If you already edited `"Name"` directly in the JSON, omit `name_suffix`.

Expected output:
```
localhost : ok=6  changed=1  unreachable=0  failed=0
```

- `changed=1` — object was **created** (new)
- `changed=0` — object **already existed** and no fields changed (idempotent)
- `failed=1` — see [Troubleshooting](#troubleshooting)

### Full workflow example for an NTP policy

```bash
# 1. Pull
ansible-playbook pull_intersight_config.yml

# 2. Copy one NTP policy to push folder (corp-ntp)
python3 -c "
import json
with open('exports/ntp_policies/corp-ntp.json') as f:
    policy = json.load(f)
# Edit the NTP servers before writing
policy['NtpServers'] = ['10.1.0.1', '10.1.0.2']
policy['Description'] = 'DR site NTP policy'
with open('push/ntp_policies.json', 'w') as f:
    json.dump([policy], f, indent=2)
print('Ready:', policy['Name'])
"

# 3. Push as a new policy named corp-ntp-DR
ansible-playbook push_intersight_config.yml \
  -e "target_org=default" \
  -e "push_types=ntp_policies" \
  -e "import_dir=$(pwd)/push" \
  -e "name_suffix=-DR"
```

---

## Push Playbook Parameters

| Parameter | Default | Description |
|---|---|---|
| `target_org` | `"default"` | Intersight organization name to create objects in |
| `push_types` | `""` (all) | Comma-separated list of object types to push. Empty = push every type. |
| `import_dir` | `./exports` | Directory containing the `<type>.json` files to read |
| `name_prefix` | `""` | String prepended to every object's `Name` on creation |
| `name_suffix` | `""` | String appended to every object's `Name` on creation |

### Examples

Push only BIOS policies, creating copies named `<original>-COPY`:
```bash
ansible-playbook push_intersight_config.yml \
  -e "push_types=bios_policies" \
  -e "name_suffix=-COPY"
```

Push BIOS and NTP policies to a specific org:
```bash
ansible-playbook push_intersight_config.yml \
  -e "target_org=my-org" \
  -e "push_types=bios_policies,ntp_policies" \
  -e "import_dir=$(pwd)/push"
```

Push all types from exports (full migration):
```bash
ansible-playbook push_intersight_config.yml \
  -e "target_org=my-org" \
  -e "import_dir=$(pwd)/exports" \
  -e "name_suffix=-COPY"
```

---

## Supported Policy / Object Types

| `push_types` value | Object Type | Export file | Individual file location |
|---|---|---|---|
| `pools_ip` | IP Pools | `exports/pools_ip.json` | `exports/pools_ip/<Name>.json` |
| `pools_mac` | MAC Pools | `exports/pools_mac.json` | `exports/pools_mac/<Name>.json` |
| `pools_uuid` | UUID Pools | `exports/pools_uuid.json` | `exports/pools_uuid/<Name>.json` |
| `pools_wwn` | WWN (FC) Pools | `exports/pools_wwn.json` | `exports/pools_wwn/<Name>.json` |
| `eth_adapter_policies` | Ethernet Adapter Policies | `exports/eth_adapter_policies.json` | `exports/eth_adapter_policies/<Name>.json` |
| `eth_network_policies` | Ethernet Network Policies | `exports/eth_network_policies.json` | `exports/eth_network_policies/<Name>.json` |
| `eth_qos_policies` | Ethernet QoS Policies | `exports/eth_qos_policies.json` | `exports/eth_qos_policies/<Name>.json` |
| `fc_adapter_policies` | FC Adapter Policies | `exports/fc_adapter_policies.json` | `exports/fc_adapter_policies/<Name>.json` |
| `fc_network_policies` | FC Network Policies | `exports/fc_network_policies.json` | `exports/fc_network_policies/<Name>.json` |
| `fc_qos_policies` | FC QoS Policies | `exports/fc_qos_policies.json` | `exports/fc_qos_policies/<Name>.json` |
| `lan_policies` | LAN Connectivity Policies | `exports/lan_policies.json` | `exports/lan_policies/<Name>.json` |
| `vnic_eth_ifs` | vNIC Ethernet Interfaces | `exports/vnic_eth_ifs.json` | `exports/vnic_eth_ifs/<Name>_<Moid>.json` |
| `san_policies` | SAN Connectivity Policies | `exports/san_policies.json` | `exports/san_policies/<Name>.json` |
| `vhba_fc_ifs` | vHBA FC Interfaces | `exports/vhba_fc_ifs.json` | `exports/vhba_fc_ifs/<Name>_<Moid>.json` |
| `adapter_config_policies` | Adapter Config Policies | `exports/adapter_config_policies.json` | `exports/adapter_config_policies/<Name>.json` |
| `bios_policies` | BIOS Policies | `exports/bios_policies.json` | `exports/bios_policies/<Name>.json` |
| `boot_policies` | Boot Order Policies | `exports/boot_policies.json` | `exports/boot_policies/<Name>.json` |
| `firmware_policies` | Firmware Policies | `exports/firmware_policies.json` | `exports/firmware_policies/<Name>.json` |
| `imc_access_policies` | IMC Access Policies | `exports/imc_access_policies.json` | `exports/imc_access_policies/<Name>.json` |
| `kvm_policies` | KVM Policies | `exports/kvm_policies.json` | `exports/kvm_policies/<Name>.json` |
| `local_user_policies` | Local User Policies | `exports/local_user_policies.json` | `exports/local_user_policies/<Name>.json` |
| `ntp_policies` | NTP Policies | `exports/ntp_policies.json` | `exports/ntp_policies/<Name>.json` |
| `power_policies` | Power Policies | `exports/power_policies.json` | `exports/power_policies/<Name>.json` |
| `snmp_policies` | SNMP Policies | `exports/snmp_policies.json` | `exports/snmp_policies/<Name>.json` |
| `ssh_policies` | SSH Policies | `exports/ssh_policies.json` | `exports/ssh_policies/<Name>.json` |
| `storage_policies` | Storage Policies | `exports/storage_policies.json` | `exports/storage_policies/<Name>.json` |
| `syslog_policies` | Syslog Policies | `exports/syslog_policies.json` | `exports/syslog_policies/<Name>.json` |
| `thermal_policies` | Thermal Policies | `exports/thermal_policies.json` | `exports/thermal_policies/<Name>.json` |
| `server_profile_templates` | Server Profile Templates | `exports/server_profile_templates.json` | `exports/server_profile_templates/<Name>.json` |
| `server_profiles` | Server Profiles | `exports/server_profiles.json` | `exports/server_profiles/<Name>.json` |
| `chassis_profiles` | Chassis Profiles | `exports/chassis_profiles.json` | `exports/chassis_profiles/<Name>.json` |
| `domain_profiles` | Domain Profiles | `exports/domain_profiles.json` | `exports/domain_profiles/<Name>.json` |

> **Note:** vNIC/vHBA individual files use `<Name>_<Moid>.json` because interface names are only unique within their parent policy, not globally.

---

## Troubleshooting

### `'target_org_moid' is undefined`
The org lookup task couldn't find the organization. Verify `target_org` matches the exact organization name in Intersight (case-sensitive).

### `Policy type is already attached to profiles`
The source policy is linked to server profiles. The push playbook strips the `Profiles` field automatically. If this error still appears, a previous push created the policy with profile references attached — delete that policy from Intersight and re-run.

### `Relationship to organization is missing`
The Organization reference is invalid. Ensure `target_org` exactly matches the organization name in Intersight and your API key has access to it.

### `Cannot modify read-only relationship Organization`
A PATCH is trying to change the Organization field. This means `query_params` matched an existing policy but it belongs to a different org. Verify the name (with prefix/suffix) is unique within `target_org`.

### `changed=0` when expecting a new policy
The policy already exists in Intersight with that exact name. Delete it first or use a different `name_suffix`.
