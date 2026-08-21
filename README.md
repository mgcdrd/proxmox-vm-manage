proxmox-vm-manage deployment
=============================

Ad hoc deployment for post-provisioning changes on an existing ProxMox VM:
add or resize disks, add/update/remove NICs, and create/delete/roll back
snapshots. Thin wrapper around `mgcdrd.infrabase.proxmox_vm_manage` — this
repo just supplies the Vault-backed connection vars and a place for each
VM's disk/NIC/snapshot spec to live.

Not a standing service like the other deployments — there's no fixed set of
hosts this always runs against. You run it against one VM at a time.

---

## Prerequisites

- The Ansible controller must reach the PVE API host (`proxmox_vm_manage_pve_node`)
  on port 8006, and the target VM over SSH for disk changes (block-device
  detection happens on the guest).
- Target VM already exists in `../../inventory-common/hosts.yml`.
- `../../inventory-common` cloned as a sibling of `deployments/` (see that
  repo's README).
- Collections installed: `ansible-galaxy collection install -r collections/requirements.yml`
- **`proxmoxer >= 2.3` and `requests` installed on the Ansible controller**:
  - RPM: `pip3 install 'proxmoxer>=2.3' requests`
  - Debian 12+ (system Ansible): `pip3 install 'proxmoxer>=2.3' requests --break-system-packages`
  - Debian 12+ (Ansible in a venv): same command inside the venv

---

## Setup

1. `cp inventory/group_vars/all/env.yml.example inventory/group_vars/all/env.yml`
   — set `proxmox_vm_manage_pve_node` if not `pve2`.
2. `cp inventory/group_vars/all/vault.yml.example inventory/group_vars/all/vault.yml`
   — works as-is once Vault has `infra/<env>/proxmox/root` populated (already
   true if `deployments/foreman` has been set up, since both deployments
   share that same Vault path). Uncomment the API token lines once a token
   exists (see that file for how to mint one) — recommended for NIC/snapshot
   changes, not required.

---

## Usage

Define what to change for a specific VM in
`inventory/host_vars/<hostname>.yml` (hostname must match the entry in
`../../inventory-common/hosts.yml`) — copy
`inventory/host_vars/example.yml.example` as a starting point. Then:

```bash
ansible-playbook site.yml --limit <hostname>
```

Or skip the host_vars file for a true one-off and pass the spec via
extra-vars:

```bash
ansible-playbook site.yml --limit <hostname> -e '{"proxmox_vm_manage_snapshots": [{"snapname": "pre-maint", "retention": 5}]}'
```

Each section (disks/NICs/snapshots) only runs if its list has entries —
`--limit` is for speed, not safety; an unscoped run is a no-op on any host
that doesn't define one of the three lists. Scope to one kind of change with
tags:

```bash
ansible-playbook site.yml --limit <hostname> --tags storage
ansible-playbook site.yml --limit <hostname> --tags network
ansible-playbook site.yml --limit <hostname> --tags snapshot
```

---

## Role Variables

Host/group topology (`ansible_host`, `ansible_user`, `domain`, `vault_addr`,
`vault_kv_*`) comes from `../../inventory-common` (second inventory source
in `ansible.cfg`), same as every other deployment.

This repo sets:

| Variable | Where | Purpose |
|---|---|---|
| `proxmox_vm_manage_pve_node` | `group_vars/all/env.yml` | Which PVE node's API to hit (any node manages the whole cluster) |
| `vault_proxmox_api_password` | `group_vars/all/vault.yml` | Password auth — always required (disk sub-role has no token support) |
| `vault_proxmox_api_token_id` / `_secret` | `group_vars/all/vault.yml` (optional) | Token auth — used by the NIC/snapshot sub-roles when set |
| `proxmox_vm_manage_disks` / `_nics` / `_snapshots` | `host_vars/<hostname>.yml` or `-e` | Per-VM spec — the actual work to do |

Everything else — `proxmox_vm_manage_api_user`, `proxmox_vm_manage_vm_name`
(defaults to the inventory hostname), the full per-entry key list for
disks/NICs/snapshots — is `mgcdrd.infrabase.proxmox_vm_manage`'s own
interface. See that role's README for the complete reference; this repo
doesn't reinvent it.

---

## Notes

- **VM name must match PVE**: `proxmox_vm_manage_vm_name` defaults to the
  inventory hostname. If a VM's name in PVE differs from its FQDN in
  `inventory-common/hosts.yml`, set `proxmox_vm_manage_vm_name` explicitly
  in that host's `host_vars`.
- **No `proxmox_vm` (clone/lifecycle/template) coverage**: this deployment
  is for changes *after* a VM exists, same scope as the role it wraps.
  Cloning a new VM is out of scope here.
- **`foreman` runs through this deployment now**: as of 2026-08-20,
  `deployments/foreman/site.yml` no longer calls `proxmox_disk`/`proxmox_nic`
  directly — its data disk and any additional NICs are provisioned here
  first (see `inventory/host_vars/foreman.example.com.yml.example`), then
  `deployments/foreman` runs against the already-provisioned VM. This
  deployment is the one place VM-level ProxMox changes happen, whether or
  not the VM belongs to a specific service deployment's own playbook.

---

## Client/customer delivery

Portable as-is: point `ansible.cfg`'s inventory path at the customer's
`inventory-<client>` repo instead of `inventory-common`, and populate
`infra/<env>/proxmox/root` (and optionally `.../proxmox/api_token`) in their
Vault instead.
