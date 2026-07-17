# DSAI → Skipjack slurm.conf merge — impact review

Review of the DSAI slurm.conf parameters proposed for addition to
`slurm/conf/slurm.conf.template.arch` (skipjack). Most of the list is fine
(many are Slurm defaults), but **four items interact directly with the CLOACK
billing & enforcement design** (paid partitions, weekly cap, scavenger
preemption).

---

## 🚨 1. Priority block — silently redefines the Weekly Cap

Our weekly spend cap is enforced as `GrpTRESMins=billing=` on the account
association, and Slurm counts usage against it based on the priority plugin's
decay/reset settings:

- **As proposed** (`PriorityDecayHalfLife=30-0` + `PriorityUsageResetPeriod=quarterly`):
  the cap becomes a **rolling 30-day budget reset quarterly**, not weekly.
- **Today** (no `PriorityType` → `priority/basic`): no decay at all — the cap
  is an all-time limit. Also wrong.
- The portal gauge reads true per-week data from slurmrestd → **UI and
  enforcement would silently diverge** (gauge says "under cap", Slurm blocks
  the job — or vice-versa).

✅ **Use instead:** `PriorityUsageResetPeriod=weekly` + `PriorityDecayHalfLife=0`.
The rest of the multifactor block is fine as-is.

## 🚨 2. PreemptType=preempt/partition_prio — breaks scavenger preemption

Our design runs scavenger (free, preemptible) **in the same partition** as
paid jobs (`a100`, `AllowQos=ALL`). `partition_prio` only preempts across
partitions with different `PriorityTier` — so paid jobs could never preempt
scavenger. "Free, preemptible" becomes free and unpreemptible.

✅ **Use instead:** `PreemptType=preempt/qos` + `PreemptMode=REQUEUE`, then on
the QOS layer:

```bash
sacctmgr modify qos normal,premium,ssci set Preempt=scavenger -i
```

(`SlurmQOS.preempt_mode` already exists in ColdFront for this.)

## ⛔ 3. AllowSpecResourcesUsage=yes — users can steal the Weka cores

`ga1[38-43]` reserve `CpuSpecList="2-9"` + `MemSpecLimit=40000` for the system
(including the Weka client). This flag lets any user override the reservation
with `--core-spec`/`--thread-spec` and starve Weka I/O **node-wide** (hurts
every job on the node).

✅ **Omit it** (Linux default = no). Keep `TaskPluginParam=SlurmdOffSpec` —
that one is the good companion (keeps slurmd itself off the reserved cores).

## ⚠️ 4. AccountingStoreFlags — keep job_comment, drop job_env

- `job_comment` is a **win**: our paid-submit cost gate travels in
  `comment=accept_cost`, so storing it gives an audit trail in sacct of who
  accepted charges.
- `job_env` stores every batch job's full environment in slurmdbd → secrets
  (API tokens, `SLURM_JWT`) + the biggest DB-bloat item.

✅ **Use:** `AccountingStoreFlags=job_comment,job_script`

## (soft) `defer` in SchedulerParameters — degrades interactive

`defer` + `batch_sched_delay=20` delays every scheduling decision — directly
against our `interactive` QOS (priority 1000, supposed to start fast) and
`srun --pty` sessions. Tuned for ~980 nodes; on the 6-node bring-up, omit
`defer` and re-add when queue depth justifies it. Not a correctness break,
just UX.

---

## Recommended final form (the 4 critical items)

```ini
# 1. Priority — weekly cap semantics (matches CLOACK Weekly Cap gauge)
PriorityType=priority/multifactor
PriorityFlags=MAX_TRES,CALCULATE_RUNNING
PriorityUsageResetPeriod=weekly      # NOT quarterly
PriorityDecayHalfLife=0              # NOT 30-0 (no decay; hard weekly reset)
PriorityMaxAge=7-0
PriorityFavorSmall=NO
PriorityWeightAge=5000
PriorityWeightFairshare=20000
PriorityWeightJobSize=5000

# 2. Preemption — QOS-based (scavenger shares partition with paid)
PreemptType=preempt/qos              # NOT partition_prio
PreemptMode=REQUEUE

# 3. Core specialization — protect Weka cores
TaskPluginParam=SlurmdOffSpec
# AllowSpecResourcesUsage — OMIT (do not add)

# 4. Accounting extras — accept_cost audit, no env capture
AccountingStoreFlags=job_comment,job_script   # NOT job_env
```

## How to apply

- Items **1 & 2 need a slurmctld restart** (`scontrol reconfigure` is not
  enough for `PriorityType`/`PreemptType`).
- Skipjack runs `manage_config=False`, so the live source of truth is the
  mirror `${BASE_DATA}/slurm/skipjack/slurm.conf` — edit it there, then mirror
  the same change into `slurm/conf/slurm.conf.template.arch` in git (the
  template is only the bootstrap copy).

## Everything else on the DSAI list

Slurm defaults / neutral to the ColdFront↔Slurm integration (accounting, JWT,
`job_submit.lua`, billing weights untouched):

| Param | Verdict |
|---|---|
| `ReturnToService=2` | 👍 Welcome — auto-resumes nodes after reboot (cures the manual `scontrol update state=resume` step). |
| `SwitchType=switch/none` | Default no-op; add a commented topology note for future IB. |
| `TreeWidth=65533` | Fine — direct slurmd→ctld comms suits the containerized controller. |
| `MpiDefault=none` | Default no-op. |
| `SlurmctldTimeout=120` / `SlurmdTimeout=300` | Defaults, no-op. |
| `RebootProgram=/sbin/reboot` | Fine — enables `scontrol reboot`; pairs well with `ReturnToService=2`. |
| `MaxJobCount=100000` / `MaxArraySize=10000` | Fine — just confirm the skipjack-slurmctld container memory limit accommodates it. |
| `Waittime=0`, `InactiveLimit=0`, `KillWait=30` | Defaults, no-op. |
| `MinJobAge=30` | Safe — pipeline reads completed jobs from slurmdbd (REST), never squeue. |
| `JobAcctGatherFrequency=30`, RAPL energy | Fine — energy TRES unused by billing; RAPL needs `msr` on bare-metal nodes. |
| `JobCompType=jobcomp/none` | Correct — slurmdbd (`collect_sacct`) is our job record pipeline; no jobcomp plugin needed. |
| `DebugFlags=Gres` | Fine during GPU bring-up; drop later (log volume). |
| `HealthCheckProgram` (NHC) | 👍 Good addition — pair with `HealthCheckInterval`/`HealthCheckNodeState`; keep checks conservative (spurious failures = bulk drains). |
