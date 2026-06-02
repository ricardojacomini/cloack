# Deploy ARCH Slurm from a remote workstation (Ansible)

End-to-end guide for running the CLOACK Ansible playbooks **from a workstation
other than the ColdFront host** against a bare-metal Slurm cluster (controller
+ login + compute). Goal: reproduce the same binary stack the Docker dev
deploy produces — same Slurm version, same SPANK plugins, same UIDs 449
(munge) / 450 (slurm) — and validate the targets BEFORE touching them with
[slurm-verify/playbook.yml](../ansible/slurm-verify/playbook.yml).

Source of truth for the `ansible/` tree: the Python dict `STATIC_FILES` in
[coldfront/custom/plugins/arch_sync/ansible_scaffold_data.py](../coldfront/custom/plugins/arch_sync/ansible_scaffold_data.py).
The tree is **gitignored** — regenerated on demand via
`./start.sh slurm ansible all`. Edits to playbooks must be re-embedded with
[scripts/regen_ansible_scaffold.py](../scripts/regen_ansible_scaffold.py) so
fresh clones keep producing the same content (memory
`feedback_ansible_scaffold_refresh`).

## Audience

Operators with shell access to the **ColdFront host** (where the source of
truth lives) AND SSH key access to the **bare-metal cluster** (Rocky Linux 9
controller + nodes). Two separate machines; the workstation is in the middle.

```
┌──────────────────────────┐         ┌──────────────────────────┐
│  Workstation (you)       │  SSH    │  Bare-metal cluster      │
│  - Python 3.10+          │ ──────▶ │   - <cluster>-ctld       │
│  - ansible-core ≥ 2.15   │         │   - <cluster>-dbd        │
│  - ansible/ tree (rsync) │         │   - <cluster>-restd      │
└──────────────────────────┘         │   - login01, cn-01..N    │
            ▲                        └──────────────────────────┘
            │ rsync ansible/
┌──────────────────────────┐
│  ColdFront host          │
│  ./start.sh slurm ansible│
│       all                │
└──────────────────────────┘
```

> **Source-of-truth rule (`feedback_primary_repo_only`):** never edit the
> `ansible/` tree on the workstation as a permanent change. Edit
> `ansible_scaffold_data.py` on the ColdFront host, re-run
> `scripts/regen_ansible_scaffold.py`, commit, push, then rsync. The
> workstation is a *deployment vehicle*, not a development environment.

---

## 1. Workstation prerequisites

```bash
# Python 3.10 or newer (Linux: dnf/apt; macOS: brew or pyenv)
python3 --version

# ansible-core 2.15+ (any pip-installable variant works; use a venv to avoid
# polluting system Python)
python3 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate
pip install --upgrade pip
pip install 'ansible-core>=2.15'

ansible --version              # confirm 2.15.x or newer
```

Galaxy collections (matches [ansible/requirements.yml](../ansible/requirements.yml)):

```bash
# Run AFTER you have the tree in step 2 — needs ansible/requirements.yml
ansible-galaxy collection install -r ansible/requirements.yml
```

Required (declared in [requirements.yml](../ansible/requirements.yml)):
- `community.general >= 8.0.0`
- `ansible.posix >= 1.5.0`

**SSH key:** every target host must accept your key for the deploy user
without a password prompt. The deploy user must have `sudo` (the playbooks
use `become: true`).

```bash
ssh-copy-id deploy@<cluster>-ctld
ssh-copy-id deploy@login01
ssh-copy-id deploy@cn-01
# ... one per host
ssh deploy@<cluster>-ctld 'sudo -n true' && echo "sudo NOPASSWD ok"
```

---

## 2. Get the `ansible/` tree onto the workstation

The tree is generated on the ColdFront host and copied over. Two paths —
pick whichever fits your workflow.

### Path A — rsync from the ColdFront host (preferred)

On the **ColdFront host**:

```bash
cd /Users/ricardo/arch_jhu_dsai_schmidt           # or wherever the repo lives
./start.sh slurm ansible all                       # writes ./ansible/*
```

Then on the **workstation**:

```bash
mkdir -p ~/arch
rsync -av --delete coldfront-host:/Users/ricardo/arch_jhu_dsai_schmidt/ansible/ ~/arch/ansible/
cd ~/arch/ansible
ansible-galaxy collection install -r requirements.yml
```

### Path B — generate on the workstation directly

Only viable if you have a checkout of the full repo on the workstation
(rare for ops staff). The generator is just a Python dict serialisation:

```bash
cd ~/arch_jhu_schmidt
python3 -c "
from coldfront.custom.plugins.arch_sync import ansible_scaffold_data as m
import os, sys
for path, content in m.STATIC_FILES.items():
    full = os.path.join('ansible', path)
    os.makedirs(os.path.dirname(full), exist_ok=True)
    with open(full, 'w') as f: f.write(content)
print(f'wrote {len(m.STATIC_FILES)} files')
"
```

Either path leaves you with a tree that matches the [layout in
ansible/README.md](../ansible/README.md):

```
ansible/
├── ansible.cfg
├── requirements.yml
├── group_vars/all.yml             ← shared defaults (versions, UIDs)
├── clusters/<name>/               ← per-cluster inventory + slurm.conf
│   ├── inventory.ini
│   ├── slurm.conf
│   ├── group_vars/all.yml
│   └── README.md
├── roles/{common,slurm_users,slurm_build,munge,slurm_config,slurm_server,slurm_client}/
├── slurm-verify/playbook.yml      ← read-only pre-flight (you start here)
├── slurm-server/playbook.yml      ← controller + dbd + restd
└── slurm-client/playbook.yml      ← compute + login
```

---

## 3. Fill in `clusters/<name>/inventory.ini`

The generator emits a placeholder inventory. Edit it with the real
hostnames/IPs for your cluster. Required groups (drop the `<cluster>_`
prefix if your inventory is single-cluster):

```ini
# clusters/prod/inventory.ini
[prod_slurmctld]
prod-ctld.internal ansible_host=10.0.1.10 ansible_user=deploy

[prod_slurmdbd]
prod-dbd.internal  ansible_host=10.0.1.11 ansible_user=deploy

[prod_slurmrestd]
prod-restd.internal ansible_host=10.0.1.12 ansible_user=deploy

[prod_baremetal]
login01.internal   ansible_host=10.0.2.10 ansible_user=deploy slurm_node_type=login
cn-01.internal     ansible_host=10.0.3.11 ansible_user=deploy slurm_node_type=compute
cn-02.internal     ansible_host=10.0.3.12 ansible_user=deploy slurm_node_type=compute
gpu-01.internal    ansible_host=10.0.3.20 ansible_user=deploy slurm_node_type=compute gres="gpu:4"
```

Verify connectivity before doing anything else:

```bash
cd ~/arch/ansible
ansible -i clusters/prod/inventory.ini all -m ping
```

Every host must return `pong`. Fix DNS, SSH keys, or `become_pass` issues
before proceeding.

`clusters/<name>/group_vars/all.yml` carries per-cluster overrides; the
shared defaults (Slurm version, paths, munge/slurm UIDs) live in
`ansible/group_vars/all.yml`. Don't touch UIDs 449 / 450 — they MUST match
the LDAP records ColdFront writes (memory `project_uid_gid_ranges`).

---

## 4. Pre-flight: `slurm-verify` (read-only)

```bash
ansible-playbook -i clusters/prod/inventory.ini slurm-verify/playbook.yml
```

`slurm-verify` makes **no changes**. It checks:

| Gate | What it verifies |
|---|---|
| OS family | Rocky Linux 9.x (or RHEL-compatible) |
| Time sync | chronyd present and synced |
| Munge UID | 449 system user exists |
| Slurm UID | 450 system user exists |
| Munge key | present and `0400 munge:munge` |
| Slurm version | matches `slurm_version` in group_vars (default `25.11.2`) — WARN if absent (will install) |
| SPANK | C headers present, `plugstack.conf` resolvable, `.so` linker symbols available |
| Firewall | ports 6817 (ctld) / 6818 (slurmd) / 6819 (dbd) / 6820 (restd) open per-group |
| SSSD entry cache | `entry_cache_timeout`, `entry_cache_user_timeout`, `entry_cache_group_timeout` ≤ 60 on login nodes (WARN otherwise — per memory `project_sssd_flush_hook`) |

A non-green play recap means **stop here**. Fix the OS/network problem on
the target host before running `slurm-server` or `slurm-client` — those
playbooks WILL make changes and won't roll back cleanly from a broken base.

---

## 5. Optional: `prod-sim` dry-run (`--check --diff`)

A wrapper on the ColdFront host runs both `slurm-server` and `slurm-client`
playbooks in Ansible's `--check --diff` mode against the per-cluster
inventory (no changes applied, but every diff is printed):

```bash
# On the ColdFront host:
./start.sh slurm prod-sim prod
```

You can replicate this on the workstation by running the playbooks with
`--check --diff` flags directly:

```bash
ansible-playbook -i clusters/prod/inventory.ini slurm-server/playbook.yml --check --diff
ansible-playbook -i clusters/prod/inventory.ini slurm-client/playbook.yml --check --diff
```

Use this BEFORE the real provision — particularly on hosts that already
have a Slurm install you don't want to clobber. The `--check` mode shows
every file rewrite and service restart the playbook would do.

---

## 6. Real provision — order matters

Run **server first**, then **client**. `slurmd` (client) won't register
without a live `slurmctld` (server) on the network, and `slurmctld` won't
start without `slurmdbd` (also server) accepting connections.

### 6a. Controller / DBD / RESTD

```bash
ansible-playbook -i clusters/prod/inventory.ini slurm-server/playbook.yml
```

What it does (high-level — see [roles/slurm_server/](../ansible/roles/slurm_server/)):

1. `common` → EPEL + CRB, build deps (gcc, autotools, libssl-dev, mariadb-devel, jansson-devel, etc.), chronyd, sssd
2. `slurm_users` → create `munge` (UID 449) + `slurm` (UID 450) system users with home dirs
3. `slurm_build` → fetch Slurm 25.11.2 source, `./configure` with the same flags as `slurm/base/Dockerfile`, `make && make install`
4. `munge` → install munge, distribute the key file, start munged
5. `slurm_config` → deploy `slurm.conf` + `plugstack.conf`, compile SPANK `.so` plugins from `slurm/spank-plugins/`
6. `slurm_server` → systemd units for `slurmctld`, `slurmdbd`, `slurmrestd`; enable + start

**Wait** for `slurmdbd` to report `active (running)` before moving on:

```bash
ssh deploy@prod-dbd 'systemctl is-active slurmdbd'   # → active
ssh deploy@prod-ctld 'systemctl is-active slurmctld' # → active
```

### 6b. Compute + login nodes

```bash
ansible-playbook -i clusters/prod/inventory.ini slurm-client/playbook.yml
```

Same `common`/`slurm_users`/`slurm_build`/`munge`/`slurm_config` foundation, plus:

7. `slurm_client` → `slurmd` systemd unit, `cgroup.conf`, optional `gres.conf` if the host has `gres=` in inventory

After this finishes, nodes should appear in `sinfo`:

```bash
ssh deploy@login01 'sinfo -h'
# prod*    up    infinite    1   idle    cn-01
# prod*    up    infinite    1   idle    cn-02
```

---

## 7. Post-deploy validation

### Functional smoke test

```bash
ssh deploy@login01 'sudo -u <a_test_user> sbatch --wrap="hostname && date"'
ssh deploy@login01 'squeue'              # job appears, then disappears
ssh deploy@login01 'sacct -X --starttime=$(date -d "5 min ago" +%FT%T)'
```

### LDAP resolution via SSSD

```bash
ssh deploy@login01 'getent passwd <a_test_user>'   # returns the LDAP entry
ssh deploy@login01 'id <a_test_user>'              # shows correct groups
```

### Admin escalation for `cn=admin` members

Members of LDAP `cn=admin` must be able to `sudo` on the cluster — that's
how operators log in interactively. Verify:

```bash
ssh deploy@login01 'getent group admin'        # LDAP group present
ssh deploy@login01 'sudo -l -U <admin_user>'   # NOPASSWD: ALL
```

> **Pending dependency** — see [related plan](./README.md): the
> `/etc/sudoers.d/admin` file with `%admin NOPASSWD: ALL` is currently
> created by the Docker entrypoint
> ([slurm/slurmd/entrypoint.sh:44-46](../slurm/slurmd/entrypoint.sh)) but
> is NOT yet in the Ansible `slurm_client` role. If `sudo -l` shows the
> admin user has no rules, that's the gap — it must be added to
> `ansible_scaffold_data.py` key `roles/slurm_client/tasks/main.yml` for
> bare-metal parity. Re-run `scripts/regen_ansible_scaffold.py` after the
> edit, commit, push, rsync to the workstation, and re-run
> `slurm-client/playbook.yml --tags sudoers`.

### Cross-cluster federation check (multi-cluster only)

If this is a federated deploy where `slurmdbd` is shared:

```bash
ssh deploy@login01 'sacctmgr show cluster format=Cluster,Federation -P'
```

All federated clusters should appear here. ColdFront's `SlurmCluster.name`
must match exactly (case-sensitive).

---

## 8. Editing playbooks — round-trip workflow

If during validation you discover a missing task (e.g. the sudoers gap
above), the source of truth is **never** the workstation's
`ansible/` tree:

1. SSH into the ColdFront host
2. Edit `ansible/<role>/tasks/main.yml` directly to validate the change
3. Re-run the affected playbook to confirm the fix works on a target host
4. **Persist the change**: edit the corresponding string inside
   `coldfront/custom/plugins/arch_sync/ansible_scaffold_data.py`
   (`STATIC_FILES[<path>]`)
5. Run `python3 scripts/regen_ansible_scaffold.py` to verify the embedded
   content matches the on-disk file (idempotent — should report zero diff
   if step 4 was complete)
6. Commit + push
7. On the workstation: `rsync` the fresh tree (path A in section 2 above)

Skipping step 4 means the next operator who regenerates from
`ansible_scaffold_data.py` will see their changes overwritten — memory
`feedback_ansible_scaffold_refresh` documents this trap.

---

## 9. Cleanup / rollback

`slurm-verify` is read-only, so nothing to clean.

`slurm-server` and `slurm-client` install + enable systemd units; to
revert manually:

```bash
ssh deploy@prod-ctld 'sudo systemctl stop slurmctld slurmdbd slurmrestd'
ssh deploy@prod-ctld 'sudo systemctl disable slurmctld slurmdbd slurmrestd'
ssh deploy@login01   'sudo systemctl stop slurmd && sudo systemctl disable slurmd'
# Binaries stay in /usr/local/sbin (slurm_build does `make install`).
# If you need them gone: rm -rf /usr/local/sbin/slurm* /etc/slurm/* /var/spool/slurm*
```

There's no `--undo` playbook today; if rollback becomes a recurring need,
that's a follow-up scaffolding addition.

---

## Reference

- [slurm/ANSIBLE.md](../slurm/ANSIBLE.md) — canonical bare-metal deployment guide (layout, playbooks, end-to-end deploy, what's NOT covered)
- [coldfront/custom/plugins/arch_sync/ansible_scaffold_data.py](../coldfront/custom/plugins/arch_sync/ansible_scaffold_data.py) — source of truth (Python dict)
- [scripts/regen_ansible_scaffold.py](../scripts/regen_ansible_scaffold.py) — re-embed playbook edits back into the data module
- [start.sh](../start.sh) — `cmd_slurm ansible`, `cmd_slurm prod-sim` entry points (search for `ansible` and `prod-sim`)

## Out of scope

- **Container target** — this is for bare-metal only. For dev, use
  `./start.sh deploy dev -y` on the ColdFront host.
- **Initial OS install** — Rocky 9 must already be installed and SSH-reachable.
  Provisioning the OS itself (kickstart/PXE/cloud-init) is a separate workflow.
- **Networking, storage drivers, GPU drivers** — out of scope per
  `ansible/README.md`. Ansible only installs Slurm + dependencies.
- **Federation registration in ColdFront** — register the new cluster via
  Django Admin → SlurmCluster, OR use the new `coldfront
  import_slurm_cluster --cluster <name>` management command to snapshot
  partitions/QOS/nodes from the live cluster into ColdFront (memory
  `project_import_slurm_cluster`).
