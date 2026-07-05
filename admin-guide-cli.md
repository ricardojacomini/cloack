# CLOACK — Staff / Admin CLI & Troubleshooting Guide

Console commands to operate and troubleshoot the CLOACK stack (ColdFront +
OpenLDAP + Keycloak + Slurm + PostgreSQL). Everything here is run from the
**host** where the stack runs (dev laptop or `mgmt0x`), against the running
containers.

> **Golden rules**
> - **Preview before you destroy.** Anything that removes users/accounts/homes
>   has a `--dry-run` (or a UI preview). Always run it first.
> - **Files & DB via commands, never hand-patched.** Use management commands and
>   `start.sh` subcommands — no manual SQL / no editing volumes by hand.
> - **`coldfront <cmd>`, not `manage.py`.** Inside the container the wrapper is
>   `coldfront`.
> - **Runtime module path is `coldfront.plugins.arch_sync`** (the repo's
>   `coldfront/custom/plugins/arch_sync/` is deployed into site-packages at
>   container start) — **not** `coldfront.custom.plugins.arch_sync`.
> - **Never add `-M <cluster>`** to `sacct` / `sreport` / `squeue` / `scontrol`
>   / `sacctmgr` — the cluster is auto-resolved.

---

## 0. Conventions

Run a Django management command inside the ColdFront container:

```bash
docker exec coldfront coldfront <command> [args]
```

Open a Django shell (one-liner):

```bash
docker exec coldfront coldfront shell -c "from coldfront.plugins.arch_sync.models import SlurmCluster as C; print(list(C.objects.values_list('name', flat=True)))"
```

Tail a service log:

```bash
./start.sh log coldfront          # or: keycloak | openldap | qcluster | postgres | helpdesk
docker logs --tail 50 <container>
```

Health snapshot of everything:

```bash
./start.sh doctor                 # 14-check reconcile/health report
bash scripts/verify-deploy.sh     # authoritative post-deploy gate (exit code = red checks)
docker ps --format '{{.Names}}\t{{.Status}}'
```

---

## 1. User & Account Management

### Remove specific users (by name) — **preview first**

Preview (does **not** delete; shows PI-cascade risk):

```bash
docker exec coldfront coldfront delete_users \
  -u ssci-asap7772 ext-nargatoff ext-mwaugh --dry-run
```

Actually delete (four-step teardown: Slurm associations + home dir
`/home/<aff>/<user>` + LDAP entry + ColdFront `User`/`UserProfile`):

```bash
docker exec coldfront coldfront delete_users -u ext-nargatoff ext-mwaugh --yes
```

| Flag | Effect |
|------|--------|
| `-u`, `--username` | One or more usernames (repeatable). |
| `--dry-run` | Preview only, incl. PI-cascade risk. |
| `--yes` | Required to actually delete (unless `--dry-run`). |
| `--keep-home` | Do not remove home directories. |
| `--keep-slurm` | Do not remove Slurm associations. |
| `--force-pi` | Allow deleting a **PI** (cascade-deletes their projects/allocations). Refused otherwise. |

> Refuses system accounts (`admin`, `root`, `coldfront`). Deleting a PI without
> `--force-pi` is blocked because it would cascade-delete their projects and
> allocations. Shared per-PI `/scratch` is never touched.

### Remove a whole course (class) cohort

Every account named `<course>-*` (disposable class accounts). Preview → confirm:

```bash
docker exec coldfront coldfront delete_course_users --course en540 --dry-run
docker exec coldfront coldfront delete_course_users --course en540 --yes
```

Same flags as `delete_users` (`--keep-home`, `--keep-slurm`). The helper refuses
a blank/invalid course code, so a bare `-` can never match the whole table.

### Import users

New users from CSV (`full_name,email,uid`):

```bash
docker exec coldfront coldfront import_users_csv --csv-file /path/in/container/users.csv
# Disposable course accounts (<course>-<uid> + per-course email):
docker exec coldfront coldfront import_users_csv --csv-file users.csv --course en540
```

Or use the UI: **Import Users** page (`/billing/import-users/`) — **New Users**
(CSV) and **Migration** (LDIF slapcat dump) modes.

### Export users (directory → CSV)

On the **Import Users** page, the **Export Users** card streams a CSV with
`full_name,email,uid,is_pi,members,member_of` (filterable by scope and
affiliation). `members` = a PI's LDAP group members; `member_of` = the PI(s) a
user belongs to. Direct URL:

```
/billing/import-users/?export=csv&scope=all&affiliation=schmidt
```

---

## 2. LDAP / Directory

ColdFront is the source of truth; LDAP is pushed from it.

```bash
# Reconcile ColdFront → LDAP (users, homes, PI groups, admin/helpdesk groups)
docker exec coldfront coldfront sync_ldap

# Repair drifted/duplicate GIDs
docker exec coldfront coldfront repair_ldap_gids

# Full LDAP reinit from ColdFront (CLI) — reimports LDIF, forces sync
./start.sh ldap-reinit dev

# Inspect LDAP directly
docker exec openldap slapcat -n 1 | less           # dump the data DB (LDIF)
# phpLDAPadmin UI is also available as a service
```

Backup LDAP before risky changes:

```bash
docker exec openldap slapcat -n 1 > migration/ldap-$(date +%F).cat
```

> **SSSD cache** on login nodes is flushed at the end of every `sync_ldap`
> (`sss_cache -E`); group changes (`admin`, `helpdesk-admin`) become visible to
> `id`/`sudo` within ~60 s. A shell already open keeps its group list until the
> user re-SSHes.

---

## 3. Keycloak

### Status & admin password

```bash
./start.sh keycloak status          # health + master-admin auth probe
./start.sh keycloak reset-admin     # cure master-admin password drift (hard reset, any KC version)
```

### Repoint the public host (`ARCH_PUBLIC_HOST` → OIDC/Keycloak URLs)

`ARCH_PUBLIC_HOST` in `.env` drives every browser-facing OIDC URL
(`KEYCLOAK_HOSTNAME`, `*_REDIRECT_URI`, `KEYCLOAK_ACCOUNT_BASE`, ColdFront's
`OIDC_OP_AUTHORIZATION_ENDPOINT`). A plain `detect_public_host.py` **never**
overwrites an existing value — operator-override wins, so it prints `[kept]` and
a stale/internal FQDN (e.g. one `socket.getfqdn()` baked in once) sticks. Force
a clean re-derivation with the wrapper:

```bash
./scripts/fix_public_host.sh mgmt02.mgmt.ai.cluster --dry-run   # review
./scripts/fix_public_host.sh mgmt02.mgmt.ai.cluster             # apply to files
# then, when ready: ./start.sh init prod && ./start.sh start prod  (reconcile)
```

It backs up `.env`/`coldfront.env`/`helpdesk.env`, `--force`-re-derives all OIDC
vars + themes, and scans for leftover old-host references. Editing the files does
**not** touch the running Keycloak (realm `frontendUrl` + client redirectUris) —
the reconcile (`init`+`start`, or the script's `--apply <mode>`) re-applies it.

### Clear cache / pick up theme & realm changes

Theme cache is **off** in this stack (`KC_SPI_THEME_CACHE_*=false`), so staged
theme edits are picked up on restart:

```bash
# Restage themes from the repo → Keycloak bind mount, then restart
./start.sh doctor                   # reconcile restages themes (idempotent)
docker restart keycloak
```

Re-apply **realm config** (login theme, `registrationAllowed`, OIDC redirect
URIs) — clears the one-shot marker and re-runs the config container:

```bash
docker exec -u 0 keycloak rm -f /opt/keycloak/data/.config_done
docker compose -f docker-compose-dev.yml run --rm keycloak-config   # dev
# (prod: use the prod compose file; `./start.sh update` does this at step 6.5)
```

### Dev-only: unpoison HSTS (portal unreachable after visiting KC over https)

```
# In the browser: visit https://<host>:40443/ once (gets max-age=0)
# or chrome://net-internals/#hsts → delete the host entry
```

---

## 4. TOTP / Passwordless Auth

The TOTP-enrolment banner shows for Schmidt + `ext-*` users while
`UserProfile.totp_enrolled = False`.

```bash
# Refresh totp_enrolled from Keycloak for all in-scope users (clears the banner
# for anyone who has actually enrolled; forces enrolment for those who haven't)
docker exec coldfront coldfront sync_totp_status

# Single user
docker exec coldfront coldfront sync_totp_status --username ssci-huan

# Inspect only — do not persist changes
docker exec coldfront coldfront sync_totp_status --dry-run
```

> Runs on a 30-min schedule too. If a user's banner won't clear after they
> enrolled, run the single-user form to poll Keycloak immediately.

---

## 5. Slurm

### Cluster lifecycle (Docker-managed, dev)

```bash
./start.sh slurm status             # nodes + job queue across clusters
./start.sh slurm create [name]      # generate overlay + DB record + register Resource
./start.sh slurm start              # start cluster (auto-builds images if missing)
./start.sh slurm restart            # slurmdbd → slurmctld → nodes, correct order
./start.sh slurm stop
./start.sh slurm destroy            # stop + remove containers/volumes/overlay
./start.sh slurm test-jobs          # seed jobs per user×account×partition, then collect_sacct
```

### Sync & accounting

```bash
docker exec coldfront coldfront sync_slurm              # ColdFront → Slurm (accounts/QOS/caps)
docker exec coldfront coldfront collect_sacct           # import completed jobs (REST default)
docker exec coldfront coldfront collect_sacct --clear   # wipe + re-import
docker exec coldfront coldfront collect_sacct --current-week
docker exec coldfront coldfront export_qos_config --cluster <name>   # regen qos_config.lua
```

### Node drain / resume, reload config

```bash
docker exec <cluster>-slurmctld sinfo
docker exec <cluster>-slurmctld scontrol update NodeName=cn-01 State=DRAIN Reason="maint"
docker exec <cluster>-slurmctld scontrol update NodeName=cn-01 State=RESUME
docker exec <cluster>-slurmctld scontrol reconfigure     # reload slurm.conf
```

### Regenerate a cluster's `slurm.conf` from the DB

```bash
docker exec coldfront coldfront shell -c "from coldfront.plugins.arch_sync.cluster_manager import write_slurm_conf; from coldfront.plugins.arch_sync.models import SlurmCluster as C; write_slurm_conf(C.objects.get(name='skipjack'))"
docker restart skipjack-slurmctld
```

### Import an existing cluster's topology (nodes/partitions/QOS from slurmrestd)

```bash
docker exec coldfront coldfront import_slurm_cluster --cluster skipjack --check   # preview
docker exec coldfront coldfront import_slurm_cluster --cluster skipjack --update  # apply
```

### DB ↔ slurm.conf direction, and closing the conf→DB loop (bare-metal / `.arch`)

Two directions, two sources of truth:

| Flow | Command | Source of truth |
|------|---------|-----------------|
| **DB → conf** (dev/Docker) | `write_slurm_conf(cluster)` | the ColdFront **DB** |
| **conf → DB** (bare-metal `.arch`) | `import_slurm_cluster --update` (reads the **live** cluster via slurmrestd, not the file) | the **`slurm.conf` file** |

There is **no file→DB parser**. When you hand-edit the bare-metal
`/etc/slurm/slurm.conf` (i.e. `slurm.conf.template.arch`), close the loop so
billing/gauges/allocations stay accurate — this is the **post-deploy step**:

```bash
# 1. edit /etc/slurm/slurm.conf on the controller
# 2. apply to the live cluster
scontrol reconfigure
# 3. dry-run the DB diff
docker exec coldfront coldfront import_slurm_cluster --cluster skipjack --update --check
# 4. write the DB
docker exec coldfront coldfront import_slurm_cluster --cluster skipjack --update
```

`--update` upserts `SlurmPartition`/`SlurmQOS`/`SlurmNode` and overwrites
Slurm-owned fields (`max_time`, `allow_qos`, `priority`, `cpus`, `real_memory`),
but **never** ColdFront policy fields (`billing_rate_tier`, `is_free`,
`tres_billing_weights`, `kind`, `hardware`, `tier_code`). **Global settings**
(auth, accounting, `JobSubmitPlugins`, JWT, `SelectType`, `ClusterName`,
`SlurmUser`) are the **immutable zone** of `.arch` — not modelled in the DB and
never imported back. Step 3 (`--check`) is automated as **check 13** of
`scripts/verify-deploy.sh` (WARN-only, bare-metal clusters); keep step 4
(`--update`) an explicit operator action (it mutates the DB).

**Wiring `.arch` as `/etc/slurm/slurm.conf` (Ansible, Stage 4):** by default the
`slurm_config` role deploys the DB-generated per-cluster bundle. To deploy the
hand-maintained static file instead, set in `clusters/<name>/group_vars/all.yml`:

```yaml
slurm_conf_src: "{{ playbook_dir }}/../../slurm/conf/slurm.conf.template.arch"
```

The role deploys that path verbatim (owner `slurm:slurm`, mode 0644) and notifies
`reconfigure slurm`. Leave `slurm_conf_src` unset for the DB-generated default.

> **Pitfall — empty partition fatals slurmctld:** a partition with **no compute
> nodes** used to render `Nodes=` empty and kill slurmctld with
> `fatal: Invalid node names in partition <p>` (`sinfo` → "Unable to contact
> slurm controller"). `generate_slurm_conf` now **skips** empty-node partitions
> (WARN per skip). To bring up a control-plane-only cluster (bare-metal compute
> added later), leave the partitions defined and regenerate — they self-suppress
> until nodes are registered as `SlurmNode(node_type="compute")`.

### Bare-metal node bring-up — troubleshooting (Stage 4 / `.arch`)

Field notes from standing up bare-metal GPU nodes (e.g. skipjack `ga1[38-43]`,
8× A100 each). The order below is the order failures surface: **config parses →
node authenticates (munge) → node registers (GRES) → jobs run (storage).** Fix
them in that order — a node that can't authenticate never gets its GRES checked.

**1) `slurm.conf` — one directive = one physical line.**
```text
error: _parse_next_key: Parsing error at unrecognized key: RealMemory
fatal: Unable to process configuration file
```
A `NodeName=` / `PartitionName=` split across lines — via a trailing `\`
continuation **or** a bare newline — is not reliably rejoined by Slurm's parser
(`slurmd` **and** `pam_slurm_adopt` both die; the latter also closes SSH sessions
right after key-accept). The wrapped `RealMemory=…` / `OverSubscribe=…` is read as
a bogus top-level key. **Fix:** collapse every directive onto a single physical
line — never wrap. In `slurm.conf.template.arch` the commented bring-up blocks are
also one-line each, so uncomment by deleting the leading `# ` only.

**2) The node parses its LOCAL file — deploy the file `slurm_conf_src` points at.**
A bare-metal `slurmd` reads its own `/etc/slurm/slurm.conf` (no `--conf-server`).
`scontrol reconfigure` on the **container** controller reloads the *controller's*
config — it does **not** rewrite a node's local file. And if the playbook is run
with `-e slurm_conf_src=/…/clusters/<name>/slurm.conf`, **that** staged file is
deployed — editing the repo `.arch` template does nothing for that path. Fix the
file `slurm_conf_src` actually resolves to, then re-run **without** `--check`.
Verify the deployed copy before restarting:
```bash
# on the node: line-initial RealMemory or a trailing backslash = still wrapped
ssh <node> "grep -nE '^\\s*RealMemory=|\\\\$' /etc/slurm/slurm.conf" || echo clean
ssh <node> 'slurmd -C'          # prints node config if the file parses
```

**3) GRES — `INVAL`, `gres/gpu count reported lower than configured (0 < 8)`.**
slurmd is not built `--with-nvml` (scaffold default `slurm_enable_nvml: false`),
so it cannot auto-detect GPUs — it needs an explicit `gres.conf`. Confirm the
hardware, then stage a per-cluster `gres.conf` (the `slurm_client` role copies
`clusters/<name>/gres.conf` → `/etc/slurm/gres.conf`):
```bash
ssh <node> 'nvidia-smi -L; ls -l /dev/nvidia[0-7]'   # expect 8 GPUs + 8 device files
cat > clusters/skipjack/gres.conf <<'EOF'
AutoDetect=off
NodeName=ga1[38-43] Name=gpu Type=a100 File=/dev/nvidia[0-7]
EOF
```
`Type=` (lowercase) must match slurm.conf `Gres=gpu:<type>:<n>`. After deploy:
`systemctl restart slurmd` on the node, then
`scontrol update NodeName=<n> State=RESUME`.

**4) Munge — `down*` (unreachable) vs `down` (reachable).**
In `sinfo`, the **asterisk** is the tell: `down*` = slurmctld can't reach/authenticate
the node (munge); plain `down` = the node registered fine and is just parked →
`scontrol update NodeName=<n> State=RESUME`.
```text
error: Munge decode failed: Invalid credential
auth/munge: _print_cred: ENCODED: Wed Dec 31 19:00:00 1969   # epoch-0 = key mismatch
Protocol authentication error
```
Cause: the node's `/etc/munge/munge.key` ≠ the controller's, **or** the key was
replaced but `munged` wasn't restarted (it caches the key at daemon start). A
`--limit <onenode>` run leaves the rest stale. Fix directly (don't rely on the
Ansible handler firing):
```bash
# compare every node + controller — all must match
for n in ga139 ga140 ga141 ga142 ga143; do echo -n "$n "; ssh $n md5sum /etc/munge/munge.key; done
docker exec <cluster>-slurmctld md5sum /etc/munge/munge.key
# push the good key + RESTART munged (mandatory) + slurmd
for n in …; do ssh $n 'chown munge:munge /etc/munge/munge.key && chmod 400 /etc/munge/munge.key && systemctl restart munge && systemctl restart slurmd'; done
# ground truth: credential minted on the node must decode in the controller
ssh <node> 'munge -n' | docker exec -i <cluster>-slurmctld unmunge | grep STATUS   # STATUS: Success (0)
```

**5) `srun --pty` hangs right after `[BILLING] loaded!`.**
The job launched (slurmstepd + the SPANK plugin ran — that's the BILLING line),
but `bash` blocks on startup because its home (`/weka/home/…`) isn't mounted on
the node. **Storage is outside the Slurm Ansible scope**, so a node with healthy
Slurm can still have no `/weka`. Discriminate:
```bash
srun -p a100 hostname                       # works instantly? → task launch + IO OK
srun -p a100 --chdir=/tmp --pty bash -c id  # works but plain --pty hangs? → it's home
ssh <node> 'mount | grep -i weka; timeout 5 stat /weka/home/jhu/<user> || echo HOME_UNREACHABLE'
```
`srun hostname` OK + `--chdir=/tmp` OK + Weka unmounted → mount WekaFS on the node.
If `srun hostname` **also** hangs, it's the srun stdio reverse-path (`SrunPortRange`
/ firewall between the node and the login host), not the home directory.

---

## 6. Billing

```bash
docker exec coldfront coldfront collect_billing            # compute charges
docker exec coldfront coldfront sync_job_usage             # write SlurmAccountUsagePeriod rows (the "Compute Billing" button)
docker exec coldfront coldfront generate_billing_report    # CSV/report artefacts
docker exec coldfront coldfront recalculate_billing        # recompute from scratch
docker exec coldfront coldfront shadow_compare_weekly_cap_usage --threshold 2.0   # sreport vs REST diff
```

> `SlurmAccountUsagePeriod` (the "Usage & Charges" rows) is written **only** by
> `sync_job_usage` (the Compute Billing button) — the scheduled sync and the
> `test-jobs` seeder never upsert billing periods.

---

## 7. Stack Lifecycle & Health

```bash
./start.sh init dev            # bootstrap dirs, generate .env
./start.sh start dev           # build + start all services
./start.sh update              # PROD day-to-day: git pull + migrate + restart (non-destructive)
./start.sh restart dev         # stop, rebuild images, start (keeps data)
./start.sh rebuild dev         # rebuild images --no-cache
./start.sh stop dev
./start.sh destroy dev         # remove containers (keeps volumes)
./start.sh deploy dev -y       # DESTRUCTIVE bootstrap/reset (wipes everything)
./start.sh clean-all           # full cleanup: stop + remove arch volumes/images only
```

> `update` = non-destructive (preserves DB/LDAP/volumes/jobs).
> `deploy` = bootstrap/reset (wipes) — use only on a fresh machine or an
> intentional nuke.

Post-deploy verification (silent stdout ≠ healthy):

```bash
bash scripts/verify-deploy.sh   # container status, logs, sacct State, node health,
                                # stuck PD jobs, CLI-fallback, django-q audit, OIDC secret
```

### Migrations & schedules

```bash
docker exec coldfront coldfront migrate
docker exec coldfront coldfront setup_schedules      # reconcile django-q cron dict
docker restart coldfront                             # redeploy custom/ overlay + reload code
```

---

## 8. Troubleshooting Quick Reference

| Symptom | Likely cause / fix |
|---------|--------------------|
| `slurm_load_partitions: Unable to contact slurm controller` | slurmctld died — check `docker logs <cluster>-slurmctld`. Common: `fatal: Invalid node names in partition` (empty-node partition) → regenerate `slurm.conf` (§5). |
| `getaddrinfo(slurmdbd:6819) failed` | slurmctld can't resolve `slurmdbd` — shared core services down or not on the same docker network. |
| Login node shows empty `/home` (users missing) | Container mounts the **dev named volume** instead of the real WekaFS. Prod must bind-mount `${HOME_DATA}:/home` / `${SCRATCH_DATA}:/scratch`. Check `docker inspect <node> --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}'`. |
| Keycloak login still shows "Register" after disabling it | Old theme/realm still live — restage themes + `docker restart keycloak`, and re-run `keycloak-config` to apply `registrationAllowed=false` (§3). |
| Portal unreachable (`Up (unhealthy)`, curl 000) after visiting KC | HSTS poisoning (dev http portal). Unpoison the browser (§3). |
| Bind-mounted file looks truncated in a container | virtiofs cache — `docker restart <container>`; if not fixed, full recreate. |
| `ModuleNotFoundError: No module named 'coldfront.custom'` | Wrong import path in `shell -c` — use `coldfront.plugins.arch_sync`, not `coldfront.custom.plugins.arch_sync`. |
| TOTP banner won't clear after enrolment | `coldfront sync_totp_status --username <u>` to poll Keycloak now. |
| `sacctmgr modify` silently fails | Missing `where`: `modify account where name=X set parent=Y -i`. |
| `Nodes go down` after container recreate | slurmd auto-resumes on reconnect; if stuck: `scontrol update NodeName=<n> State=RESUME`. |
| `_parse_next_key: unrecognized key: RealMemory` → `fatal: Unable to process configuration file` (slurmd/`pam_slurm_adopt`) | A `NodeName=`/`PartitionName=` wrapped across lines (`\` or bare newline). Collapse to one physical line; fix the file `slurm_conf_src` points at, not the repo template (§5, bring-up #1–2). |
| Bare-metal node still broken after `scontrol reconfigure` | Reconfigure on the container controller doesn't rewrite a node's **local** `/etc/slurm/slurm.conf`; re-deploy the file + `systemctl restart slurmd` on the node (§5, bring-up #2). |
| Node `INVAL` / `gres/gpu count reported lower than configured (0 < N)` | No `--with-nvml` build → needs explicit `/etc/slurm/gres.conf` (`Name=gpu Type=<t> File=/dev/nvidia[0-N]`); verify `nvidia-smi -L`, then RESUME (§5, bring-up #3). |
| `sinfo` shows `down*` (asterisk) / `Munge decode failed: Invalid credential` / epoch-0 `ENCODED` timestamp | munge key mismatch **or** `munged` not restarted after a key change. Match `md5sum /etc/munge/munge.key` node↔controller, push good key, `systemctl restart munge && slurmd` (§5, bring-up #4). `down` without `*` = reachable, just `State=RESUME`. |
| `srun --pty` hangs right after `[BILLING] loaded!` | Job launched but `bash` blocks — home (`/weka/home/…`) not mounted on the node (storage is outside Slurm's Ansible scope). `srun … hostname` + `--chdir=/tmp` isolate it (§5, bring-up #5). |
| Nightly paid Projects disappeared | Guarded now (a337776), but check `seed_initial_data` marker on the `${BASE_DATA}` bind-mount. |

---

*Generated for CLOACK staff/admins. Commands reflect the management commands in
`coldfront/custom/plugins/arch_sync/management/commands/` and the `start.sh`
subcommands. Keep in sync when commands change.*
