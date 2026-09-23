# ansible-iac

Ansible configuration for the homelab K3s cluster on Proxmox (1 master + 1 worker).

| Host | Role |
|---|---|
| `k3s-master` | K3s server, tainted `CriticalAddonsOnly=true:NoExecute`, ServiceLB disabled (MetalLB) |
| `k3s-worker` | K3s agent, runs all workloads; 1 TB data disk mounted on `/var/lib/rancher/k3s/storage` |

## Setup

```bash
pip install ansible-core
ansible-galaxy collection install -r requirements.yml
```

## Hosts come from `~/.ssh/config`

The inventory holds no IPs: `k3s-master` and `k3s-worker` are SSH aliases, and
`HostName`/`User`/`Port` are read from `~/.ssh/config`. To move a node, change its `HostName` there.

`homeserver-template` (the `template_host` variable) is the alias fresh template clones answer on.
It is not an inventory host: the bootstrap play reaches the node through it.

## Playbooks

| Playbook | When | What it does |
|---|---|---|
| `playbooks/bootstrap.yml` | once per fresh clone | Connects through `homeserver-template`, gives the clone the node's hostname, a new machine-id and SSH host keys, sets the node's static IP from `~/.ssh/config`, reboots onto it. Refuses to run on more than one host. |
| `playbooks/cluster.yml` | any time (idempotent) | Brings nodes to their final state: base OS (`common`), worker data disk (`worker_storage`), K3s server on the master, K3s agent on the worker. |

Both are run one node at a time with `--limit`.

## Workflow: from template clone to cluster node

Every clone of the Proxmox template (Debian 13) boots as `homeserver-template` (192.168.0.204).
Run only one fresh clone at a time. `sudo` needs a password, so Ansible prompts for it.

1. Clone the template, boot the clone, then
   `ansible-playbook playbooks/bootstrap.yml --limit k3s-master`
   It sets the hostname, regenerates the machine-id and SSH host keys, sets the static IP
   of `k3s-master` from `~/.ssh/config` and reboots onto it.
2. `ansible-playbook playbooks/cluster.yml --limit k3s-master` installs the K3s server.
3. Same for the worker: add its data disk in Proxmox, set `k3s_data_disk` in
   `inventory/host_vars/k3s-worker.yml` (check `lsblk`), then
   `ansible-playbook playbooks/bootstrap.yml --limit k3s-worker` and
   `ansible-playbook playbooks/cluster.yml --limit k3s-worker`.

Rebuilding a node gives it new host keys: run `ssh-keygen -R <ip>` first, since
known_hosts refuses a changed key.

## Layout

- `roles/bootstrap`: turns a template clone into a node (hostname, machine-id, SSH host keys, static IP, reboot)
- `roles/common`: packages, qemu-guest-agent, swap off, kernel modules, sysctls
- `roles/worker_storage`: formats (if blank) and mounts the data disk **before** the K3s agent
- `roles/k3s_server` / `roles/k3s_agent`: K3s install via `get.k3s.io`, config in `/etc/rancher/k3s/config.yaml`

K3s version: `k3s_channel: stable` by default, pin `k3s_version` in `inventory/group_vars/k3s_cluster.yml` for reproducible installs.
