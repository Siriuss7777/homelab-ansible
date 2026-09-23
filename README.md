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

## Workflow: from template clone to cluster node

Both VMs start as clones of the Proxmox template, on the template IP **192.168.0.204**.
Run only one clone at a time on that IP.

1. **Master:** boot it alone, then
   `ansible-playbook playbooks/bootstrap.yml --limit k3s-master`
   (sets hostname, regenerates machine-id + SSH host keys, base packages).
2. Give the master its final IP, update `ansible_host` in `inventory/hosts.yml`.
3. `ansible-playbook playbooks/site.yml --limit k3s-master` installs the K3s server.
4. **Worker:** boot it, set `k3s_data_disk` in `inventory/host_vars/k3s-worker.yml` (check `lsblk`), then
   `ansible-playbook playbooks/bootstrap.yml --limit k3s-worker`.
5. Give it its final IP, update `inventory/hosts.yml`, then
   `ansible-playbook playbooks/site.yml --limit k3s-worker` mounts the data disk and joins the cluster.

`site.yml` is idempotent: it can be re-run against the whole cluster once both nodes have their final IP.

## Layout

- `roles/bootstrap`: de-templates a clone (hostname, machine-id, SSH host keys)
- `roles/common`: packages, qemu-guest-agent, swap off, kernel modules, sysctls
- `roles/worker_storage`: formats (if blank) and mounts the data disk **before** the K3s agent
- `roles/k3s_server` / `roles/k3s_agent`: K3s install via `get.k3s.io`, config in `/etc/rancher/k3s/config.yaml`

K3s version: `k3s_channel: stable` by default, pin `k3s_version` in `inventory/group_vars/k3s_cluster.yml` for reproducible installs.
