# User — researcher running jobs on ARCH Portal

## Who you are + scope

You are a researcher with an ARCH Portal account: a JHU faculty / staff /
student authenticating with **JHED**, an **external collaborator** with
an `ext-*` username, or a **Schmidt Sciences** member with an `ssci-*`
username. Your day-to-day is submitting Slurm jobs, checking quotas
and bills, and managing the data in your own `/home/` and `/scratch/`.

If you also **manage allocations** for a project (request a new paid
allocation, set Weekly Caps, add other users), read [`pi.md`](pi.md) too.
If you are a member of the helpdesk team handling tickets, see
[`staff.md`](staff.md) instead.

## Quick start

### 1. First-time login

| Username pattern | Login URL | How you authenticate |
|---|---|---|
| JHED ID (e.g. `rdesouz4`) | the JHU portal card | Entra Federation → MFA from JHU (no separate password) |
| `ext-jsmith` | the JHU portal card | LDAP password (set during registration) + TOTP (Keycloak account console) |
| `ssci-jsmith` | the Schmidt portal card | LDAP password (set during registration) + TOTP (Keycloak account console) |

If you do not have an account yet:

- **JHED users** — no registration form; just sign in via JHU SSO and
  the account is created on first login.
- **`ext-*` users** — open `/user/jhu/register/` on the portal, fill in
  the form (the username **must** start with `ext-`), then wait for your
  PI to add you to their project / allocation.
- **`ssci-*` users** — same flow but on `/user/schmidt/register/`, and
  the username **must** start with `ssci-`.

After your first successful login a banner at the top of every page
prompts you to **enrol TOTP** in the Keycloak account console. Click
the banner link, scan the QR code with Google Authenticator / Authy /
Microsoft Authenticator, and you are done. The banner disappears
within 30 minutes once the periodic sync picks up the new credential.

### 2. SSH to a login node

```bash
ssh <username>@<login-node-host>
```

Authentication on login nodes uses the same identity as the portal:

- **JHED** — browser-based Device Flow opens automatically; approve
  in your browser, complete MFA in Entra, the SSH session unblocks.
- **`ext-*` / `ssci-*` / `cn=admin`** — terminal asks for your LDAP
  password and then a TOTP code (2 PAM prompts via `pam_kc_ssh.so`).

### 3. Submit your first job

```bash
sbatch --partition=med --account=<your-account> --time=00:30:00 \
  --wrap='hostname && date'
```

The Slurm CLI Lua filter prompts you to confirm the cost estimate
before the job is admitted. Press `y` to confirm or `n` to abort.

For longer jobs, use a script:

```bash
#!/bin/bash
#SBATCH --partition=high
#SBATCH --account=rdesouz4_proj1
#SBATCH --time=04:00:00
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --job-name=myrun
#SBATCH --output=/home/jhu/rdesouz4/logs/%x-%j.out

module load anaconda
python my_analysis.py
```

### 4. Check your usage and balance

```bash
squeue -u $USER          # currently queued or running jobs
sacct -u $USER --starttime=now-7days   # last week's accounting
sbalance                 # core-hours used vs the allocation cap
sbalance --week          # weekly $ cap remaining (paid allocations)
```

`sbalance` reads from ColdFront's allocation attributes — the same
numbers the portal's Allocation Detail page shows.

### 5. Manage data

| Path | Quota | Backup | Cost |
|---|---|---|---|
| `/home/<aff>/<username>` | Small, per-user | Yes (snapshot) | Included |
| `/home/<aff>/<username>/scratch_<pi>` | Symlink into PI's scratch — same quota as `/scratch` | No | Included |
| `/scratch/<aff>/<pi>` | Set per allocation (`storage_tb`) | Optional (`backup_tb`) | `$0.025 / TB / month` storage + `$0.015 / TB / month` backup |
| `/scratch/<aff>/<pi>/<project>` | Same quota as parent | Inherits parent | Same |

Storage and backup are billed per-month regardless of whether you run
jobs. Storage charges show up on your PI's billing report under
"Storage" / "Backup" lines.

## Reference matrices

### Slurm partitions

| Partition | Nodes | Max wall | Default? | When to use |
|---|---|---|---|---|
| `scavenger` | `cn-01` | 3 days | **Yes** | Free, preemptible. Default partition. Anything you can re-run if killed. |
| `low` | `cn-01` | 3 days | No | Paid, lowest CPU price. Single-node jobs that tolerate slower scheduling. |
| `med` | `cn-01`, `cn-02` | 3 days | No | Paid, mid-tier CPU price. 2-node jobs. |
| `high` | `cn-02`, `cn-03` | 3 days | No | Paid, highest CPU price. Fast scheduling, larger memory nodes. |
| `premium` | `cn-01`, `cn-02`, `cn-03` | 7 days | No | Paid, 7-day wall, all CPU nodes. No `interactive` QoS. |
| `a100` | `gpu-01` | 1 day | No | Paid GPU partition (NVIDIA A100). |
| `l40s`, `h100`, `h200` | varies | varies | No | Paid GPU partitions per hardware generation. |

### QoS (job class)

| QoS | Priority | Max wall | Notes |
|---|---|---|---|
| `scavenger` | low | 3 days | Free. Job is preempted if a paid job needs the resource. |
| `interactive` | 1000 | 4 hours | Higher priority, capped 4 h. Excluded from `premium`. |
| `normal` | normal | per partition | Default for paid allocations. |
| `premium` | high | per partition | Higher priority on `premium` partition. |
| `ssci` | normal | per partition | Free QoS for `ssci-*` accounts on paid partitions (Schmidt Sciences funded). |

### Cost (per the live `rates.lua` shipped in the cluster)

Cost is partition-specific, not global. The numbers below match the
Service Center 2026-2034 rate schedule loaded on the dev cluster (see
[`slurm/conf/rates.lua`](../../slurm/conf/rates.lua) for the source of
truth; it is auto-generated by `coldfront export_rates_lua` from the
`BillingRate` table).

| Partition | CPU rate (per SU = core-hour) | GPU rate (per GPU-hour) |
|---|---|---|
| `scavenger` | **free** ($0) | n/a |
| `low` | **$0.0056** | n/a |
| `med` | **$0.0062** | n/a |
| `high` | **$0.0107** | n/a |
| `premium` | **$0.0107** | n/a |
| `a100` | $0 (GPU billing only) | **$0.5550** |
| `l40s`, `h100`, `h200` | not yet listed in `rates.lua` — check with staff before submitting | varies |

Storage and backup are flat, per-TB-per-month:

| Item | Rate |
|---|---|
| Storage | **$0.025 / TB / month** |
| Backup | **$0.015 / TB / month** |

Storage and backup are billed every month regardless of whether you
run any jobs.

**GPU-only billing rule** — when a job uses any GPU, only its GPU
charges go into the bill total; the CPU hours of that job are reported
for transparency but **excluded** from the amount due. Pure-CPU jobs
charge CPU as expected.

**Practical sense check** — a typical 8-core, 24 h job on `med`
costs `8 cores * 24 h * $0.0062 = $1.19`. A 24 h `a100` job (1 GPU)
costs `24 * $0.5550 = $13.32`. Numbers are intentionally small in
`Service Center` mode — your PI's Weekly Cap protects you from
runaways well before the bill bites.

### CLI cheat-sheet

| Command | Use |
|---|---|
| `sinfo` | Show partition / node status. |
| `squeue` | List queued + running jobs (use `-u $USER` to filter). |
| `sbatch <script>` | Submit a batch job. |
| `srun --pty bash` | Interactive shell on a compute node. |
| `scancel <jobid>` | Cancel one of your jobs. |
| `sacct --starttime=...` | Past job accounting (state, exit code, hours). |
| `sbalance` | ColdFront-side usage vs allocation cap. |
| `sbalance --week` | Weekly $ cap remaining for paid allocations. |
| `seff <jobid>` | Efficiency report after a job finishes. |
| `stree --full` | Visualise the Slurm account hierarchy you sit under. |
| `sqos`, `sassoc`, `sfeatures` | ARCH-specific helpers (QoS overview, your associations, partition features). |

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Permission denied (publickey)` on SSH | Public-key auth disabled by default; you must complete the OTP flow | Re-SSH and follow the terminal prompts (password + TOTP, or browser Device Flow) — see [`docs/break-glass-admin.md`](../break-glass-admin.md) for the model |
| Banner says "Activate TOTP" but I already enrolled | Cache up to 30 min — the `sync_totp_status` task polls Keycloak periodically | Wait 30 min, or hit `/user/user-profile/` to trigger a refresh; if it persists open a helpdesk ticket |
| `srun: error: AssocMaxCpuMinutesPerJobLimit` | Your weekly $ cap or core-hour cap is exhausted on that allocation | `sbalance --week` to confirm; switch to `scavenger` (free), or ask your PI to raise the Weekly Cap |
| `sbatch: error: Invalid account or account/partition combination` | The account you passed (e.g. `rdesouz4_proj1`) has no QoS that admits jobs on that partition | Use `sassoc` to see which `account × partition` pairs you can submit to |
| Job stuck in PD with reason `QOSGrpBillingMinutes` | Your account's Weekly Cap on `billing` TRES was reached | `sbalance --week` to confirm; switch partitions / wait for the weekly window to reset / ask the PI to raise the cap |
| `cli_filter` aborted my job with "accept_cost not set" | A non-CLI submission path (e.g. OnDemand custom app, slurmrestd direct) did not inject `comment=accept_cost` | Re-submit via `sbatch` / `srun` from the login node, or set `--comment=accept_cost` explicitly |
| Storage charge appears on bill with 0 hours run | Storage and backup are charged monthly regardless of compute activity | This is by design; see the "Storage / Backup billing without jobs" note above |
| GPU job's CPU hours appear in the report but not in the total | GPU-only billing rule (this page's Cost table) | This is by design — GPU jobs charge GPU only; CPU hours are shown for transparency |
| `Permission denied` editing `/scratch/jhu/myproject` | The directory belongs to the PI, not you | Ask your PI to add you to the project / allocation in ColdFront |

## Where to go next

- `/user/help/` — in-app help with live updates.
- [`slurm/README.md`](../../slurm/README.md) — full Slurm reference
  (cost enforcement, hierarchy, weekly cap details, daemons in depth).
- [`pi.md`](pi.md) — if you also manage a project.
- [`README.md`](README.md) — the role index.
