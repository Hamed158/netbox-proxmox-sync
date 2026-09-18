# netbox-proxmox-sync

Ansible playbook that keeps [NetBox](https://netbox.dev/) automatically in sync with a Proxmox VE cluster's live inventory. Runs via [AWX](https://github.com/ansible/awx) on a schedule (or on demand) and pushes VM/LXC existence, status, resource specs, and IP addresses into NetBox — no manual data entry.

## What it does

- Pulls a live inventory of Proxmox nodes, QEMU VMs, and LXC containers via the `community.proxmox.proxmox` Ansible inventory plugin
- Creates/updates each VM and container as a NetBox Virtual Machine object (status, vCPUs, memory, cluster)
- For QEMU VMs: queries the QEMU Guest Agent (where installed and running) via the Proxmox API to pull the VM's real network interfaces and IP addresses
- For LXC containers: connects over SSH to the Proxmox host and runs `pct exec ... ip addr` to pull the container's real IP directly (containers don't have a guest agent)
- Creates the matching NetBox interface, assigns the IP address to it, and sets it as the VM's Primary IPv4
- Safe to re-run — every step is idempotent (`state: present`), so repeated runs just update existing objects instead of duplicating them

## Requirements

**Ansible collections** (see `collections/requirements.yml`):
- `community.proxmox` — Proxmox inventory plugin and modules
- `community.general`
- `netbox.netbox` — NetBox modules

**Python packages** (see `requirements.txt`):
- `pynetbox`

> Note: if running under AWX with a container-based Execution Environment, `requirements.txt` is **not** automatically installed into the EE at runtime. This playbook installs `pynetbox` into a writable temp path (`/tmp/py-libs`) at the start of each play as a workaround, rather than requiring a custom EE image.

## Credentials needed

Three separate credentials, all environment-variable based:

| Purpose | Variables | Used for |
|---|---|---|
| Proxmox API | `PROXMOX_URL`, `PROXMOX_USER`, `PROXMOX_TOKEN_ID`, `PROXMOX_TOKEN_SECRET` | Inventory plugin + guest agent API queries |
| NetBox API | `NETBOX_URL`, `NETBOX_TOKEN` | Writing objects into NetBox |
| Proxmox SSH | Machine credential (username + SSH key) | `pct exec` calls for LXC IP addresses |

The Proxmox API token needs, at minimum, the `PVEAuditor` role (includes `VM.GuestAgent.Audit`) granted explicitly to the **token** itself if Privilege Separation is enabled — group/user-level permissions do not automatically extend to a privilege-separated token.

NetBox token must be a **v1-style** token — the `netbox.netbox` collection does not yet support NetBox's newer `Bearer <key>.<token>` v2 token format.

## Inventory source

`inventory/proxmox.proxmox.yml` — configuration for the `community.proxmox.proxmox` inventory plugin. Filename must end in `.proxmox.yml` (plugin requirement).

Includes a `compose` block mapping each Proxmox node name to its real IP, since node names aren't otherwise DNS-resolvable — required for the `delegate_to` used in the LXC IP-fetching task.

## Running it

Designed to run as an AWX Job Template:
- **Inventory**: sourced from `inventory/proxmox.proxmox.yml` via an Inventory Source
- **Project**: this repository
- **Playbook**: `sync-vms-to-netbox.yml`
- **Credentials**: all three listed above, attached to the Job Template

Can also be run directly with `ansible-playbook` if the required environment variables are exported and an inventory file is supplied with `-i`.

## Known limitations

- QEMU VMs without the guest agent installed/running (e.g. pfSense, some appliance VMs) won't get IP addresses synced — their status/specs still sync normally
- Only the first non-loopback IPv4 address found is used as the VM/container's primary interface; additional interfaces or IPs are not currently synced
- Assumes a `/24` prefix when writing IP addresses; adjust if your environment uses different subnetting
