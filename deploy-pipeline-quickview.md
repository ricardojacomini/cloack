# Deploy pipeline — quick view (before-session cheat sheet)

> One-screen condensation of [`deploy-pipeline-4-stages.md`](deploy-pipeline-4-stages.md).
> Code travels via **git** (`origin/dev`); built images travel via **GHCR tags**
> under operator control. No stage past #2 builds a Dockerfile — it `docker pull`s.
> Registry: `ghcr.io/jhu-arch/cloack/<image>:<tag>`.

## Stages at a glance

| # | Host | Arch | Builds? | Command | Image flow |
|---|---|---|---|---|---|
| **1** laptop | macOS arm64 | local | `./start.sh deploy dev -y --paid-accounts` | `arch/<svc>` local |
| **2** build VM | Linux **amd64** | local | `start dev` → `slurm start` → `publish` | push `:dev` + `:sha-<short>` |
| **3** edulogin | bare-metal amd64 | pull | `start --from-ghcr dev` → verify → `promote` | pull `:dev`, push `:prod` + `:rc-<UTC>` |
| **4** portal (mgmt02) | bare-metal amd64 | pull | `start --from-ghcr prod` | pull `:prod` |

⚠️ **Arch:** Stage 2 must be **amd64** (Proxmox VM *or* mgmt02 itself). A Mac (arm64) build won't run on mgmt02 → `exec format error`.

## Tags

| Tag | Made by | Mutable | Meaning |
|---|---|---|---|
| `:sha-<short>` | Stage 2 `publish` | no | Snapshot tied to a git commit |
| `:dev` | Stage 2 `publish` | head | Latest build — what edulogin pulls |
| `:rc-YYYY-MM-DD-HHMMSS` | Stage 3 `promote` | no | Validated build — rollback target |
| `:prod` | Stage 3 `promote` | head | Latest validated — what portal pulls |

PATs (fine-grained, scoped `jhu-arch`, SSO-authorized): Stage 2/3 = `GHCR_BUILDER_PAT` (write); Stage 4 = `GHCR_CONSUMER_PAT` (read).

## Commands per stage

```bash
# ── Stage 2 (amd64 builder) ────────────────────────────────────────
rm slurm/conf/*.bak*                                   # publish refuses a DIRTY tree on :dev
git pull origin dev
echo $GHCR_BUILDER_PAT | docker login ghcr.io -u <user> --password-stdin
./start.sh start dev                                   # build base arch/<svc>
./start.sh slurm start                                 # build Slurm images — OPT-IN! else no login01 in GHCR
./start.sh publish --check                             # MUST show ZERO "SKIP" lines
./start.sh publish                                     # push :dev + :sha-<short>

# ── Stage 3 (edulogin — validate + promote) ────────────────────────
export CLUSTER=skipjack
echo $GHCR_BUILDER_PAT | docker login ghcr.io -u <user> --password-stdin   # write (promote pushes :prod)
./start.sh start --from-ghcr dev                       # base: pull :dev, no build
./start.sh slurm create skipjack --from-ghcr dev       # first time only
./start.sh slurm start  --from-ghcr dev                # → skipjack-login01 (+cn)
bash scripts/verify-deploy.sh                          # MANDATORY green gate
./start.sh promote                                     # :dev → :prod + :rc-<UTC>

# ── Stage 4 (mgmt02 — run prod) ────────────────────────────────────
export CLUSTER=skipjack
echo $GHCR_CONSUMER_PAT | docker login ghcr.io -u <user> --password-stdin  # read-only
./start.sh start --from-ghcr prod
./start.sh slurm create skipjack --from-ghcr prod      # first time only
./start.sh slurm start  --from-ghcr prod
```

## Data safety — will I lose deployed data?

**No.** Image = code + migrations; data lives in host volumes (`${BASE_DATA}/postgresql`, LDAP, `/home`, `/scratch`, munge, slurm state) and is re-attached on recreate. `start --from-ghcr` even runs `pg_dumpall` + `slapcat` → `${BASE_DATA}/backup/` **before** the boot-time `migrate`.

| Safe (reuse volumes) | Destroys data — NOT in GHCR flow |
|---|---|
| `start --from-ghcr`, `slurm start --from-ghcr` | `clean-all`, `prune-volumes`, `prune-slurm` |
| `promote`, `publish`, `update` | `init --regen-secrets`, `deploy` |

`deploy --from-ghcr` is **hard-refused** — can't wipe by accident. The one real risk is a **bad migration** auto-applying on boot (image rollback ≠ DB rollback) → that's why Stage 3 validates before promote.

## Rollback

```bash
./start.sh promote --rollback rc-YYYY-MM-DD-HHMMSS                       # :prod ← prior RC (retag, same digest)
docker exec -i postgres psql -U <admin> < ${BASE_DATA}/backup/postgres_<ts>.sql   # DB restore if migration corrupted data
```

## Email intake (arch-mta) — bring up + verify

Inbound email → Helpdesk tickets: MX → NPM `:25` stream → `arch-mta` (postfix,
edge profile) → `/maildrop/<queue>/new` → `get_email` cron → ticket.

```bash
# 1. MTA up (edge/prod; off in dev)
./start.sh mta create && ./start.sh mta start && ./start.sh mta status
#    Prod: pull the promoted image instead of building arch/mta locally
#    (tag defaults to the running coldfront tag, else 'dev'):
./start.sh mta create && ./start.sh mta start --from-ghcr prod && ./start.sh mta status

# 2. helpdesk must be RECREATED — `docker restart` is NOT enough: it skips the
#    /maildrop mount + overlay re-stage (migration 0050) + env_file reload:
./start.sh helpdesk stop && ./start.sh helpdesk start

# 3. queues must be LOCAL (not imap) — migration 0050:
docker exec helpdesk python manage.py shell -c \
 "from helpdesk.models import Queue; q=Queue.objects.get(slug='feedback'); \
  print(q.email_address, q.email_box_type, q.email_box_local_dir, q.allow_email_submission)"
# -> feedback@arch.jhu.edu local /maildrop/feedback/new True

# 4. end-to-end (proxy off for a direct swaks; prod sets MTA_PROXY_PROTOCOL=1):
docker exec arch-mta swaks --helo $(hostname) --server 127.0.0.1:25 \
  --to feedback@helpdesk.arch.jhu.edu --from tester@example.com \
  --header 'Subject: [feedback-3] Re: t' --body 'test'
docker exec helpdesk python manage.py get_email      # -> threads into ticket 3
```

Gotchas:
- **Stale `QUEUE_EMAIL_BOX_*` in the HOST `helpdesk.env` breaks intake** — it
  overrides EVERY queue to `imap` (settings-first) → `get_email` dies on a DNS
  error and never reads the Maildir. `ensure_host_paths` now auto-scrubs it; on an
  already-running host: `sed -i '/^QUEUE_EMAIL_BOX_/d' ${BASE_ETC}/helpdesk/helpdesk.env`
  then `helpdesk stop && start`.
- **`docker restart helpdesk` ≠ apply** — skips overlay re-stage (repo→BASE_DATA),
  container recreate (`/maildrop`), env_file reload. Always `helpdesk stop && start`.
- **`arch-mta` is edge-only** — off in dev; only the `get_email` side (empty Maildirs) runs, so dev stays green.
- **`MTA_PROXY_PROTOCOL=1`** in prod (mail via NPM stream); `0` only for a direct swaks test.
- **`mta start --from-ghcr` needs GHCR read auth** — the ghcr overlay resolves
  `arch-mta` → `ghcr.io/jhu-arch/cloack/mta:${tag}` (`build: !reset null`, so it can
  only pull). A private package fails with `[mta] GHCR pull failed (auth?...)`; log in
  with a **`read:packages`** token first: `echo "$PAT" | docker login ghcr.io -u <user> --password-stdin`.
  Within a `deploy --from-ghcr` (global `CLOACK_FROM_GHCR=1`) arch-mta already comes up
  no-build; the flag is only for standalone `mta start`. Verify: `docker inspect arch-mta --format '{{.Config.Image}}'`.
- Inbound-email follow-ups auto-classify as **Communication (e-mail) - user's reply** (`note_kind=user_reply`) — distinct from staff outbound comms (`customer_reply`, which carry FQR/LQR), and never Technical Note.

## Top gotchas

- **Slurm build is opt-in** — `start dev` does NOT build `arch/slurmd`. Run `slurm start` before `publish`, and confirm `publish --check` shows zero `SKIP` for Slurm images, or Stage 3 has no `login01` to pull.
- **Dirty tree blocks publish** on the `:dev` tag (so `:sha-*` can't lie). Commit or clean first.
- **Cluster name = `skipjack`** on Stage 3/4, set once at `slurm create`; keep only one `docker-compose-slurm-*.yml` on the host (glob picks alphabetically).
- **`start --from-ghcr` brings up the base stack only** — Slurm is a separate `slurm start`. "No login01" on a fresh host = you forgot `slurm start`.
- **`_autodetect_ghcr_mode`** keeps later `update`/`keycloak` commands on GHCR images (no fallback to `arch/<svc>`).
- **Controller on host `:6817`** under `prefix_core_containers` — published straight on 6817 (NOT `6817+(pk-1)`), so bare-metal `slurmd`/`sackd --conf-server <host>:6817` can reach it. A pk-offset publish (e.g. 6818 for pk=2) → nodes stuck `Down` / configless "connect failure".
- **Per-cluster mount files must EXIST before `up -d`** — `slurm.conf`, `gres.conf`, `cgroup.conf`, `plugstack.conf` all as **files** in `${BASE_DATA}/slurm/<cluster>/`. A missing source → Docker creates a phantom empty **directory** and slurmctld fails reading it as a dir. Stage via `cp slurm/conf/<file> ${BASE_DATA}/slurm/<cluster>/`.
- **Configless node stuck `unk`** — brief `UNKNOWN` while configless `slurmd` re-registers after a controller recreate is a normal ~1 min blip (`sinfo -N` again). If the ctld log warns `different slurm.conf than the slurmctld` (CONF_HASH), that node cached a stale local conf: `systemctl restart slurmd` (re-fetch via `--conf-server`) → `scontrol update NodeName=<X> State=RESUME`. Don't paper over with `DebugFlags=NO_CONF_HASH` — config must actually converge.
- **`docker compose up -d` is idempotent** — overlay file + `.env` unchanged → no-op (only containers whose resolved image/env/volume/port diverged recreate; one-shot `munge-init` re-runs harmlessly). `.env` is in the config hash, so an `.env` edit WILL recreate. Preview with `up -d --dry-run`.
- **Never `docker compose build` / `up --build` on a pull host** (mgmt02) — rebuilds local `arch/*` and drifts from the tested GHCR artifacts. Builds happen on the Stage 1/2 build host → publish → mgmt02 pulls. ("In Use" in `docker image ls` is per image-ID; many tags share one ID, so removing redundant tags frees ~0 disk.)

## ⚠️ Re-running Stage 4 on a LIVE host — the 2026-07-06 saga

The Stage 4 recipe above is **first-time only**. Running it **again on an already-`:prod`
host** (mgmt02 / `skipjack`) caused an accounting outage. On a live host, **verify — don't
re-deploy**. Memory: `project_stage4_live_rerun_saga`.

**Chain of failure:**
1. Host was already on `:prod` — `docker buildx imagetools inspect ghcr.io/jhu-arch/cloack/<svc>:prod`
   digest **== local `arch/<svc>`** for **7/7** images. The re-deploy pulled identical bits — zero benefit.
2. `./start.sh slurm start --from-ghcr prod` logged `SLURM_BIN not set — switching to dev compose
   overlay` and deployed **`arch-dev`**, not skipjack → `cluster_manager._ensure_shared_stack`
   **recreated the shared core** (`slurm-db`, `munge-init`, `skipjack-slurmdbd`) and hit a name conflict
   on `slurmrestd` (compose service `slurmrestd` vs the running `skipjack-slurmrestd`), aborting half-done.
3. The recreated `skipjack-slurmdbd` got container **hostname `skipjack-slurmdbd`** (compose
   `hostname: {dbd_ctr}` under `prefix_core_containers`) but the staged `slurmdbd.conf` still had
   **`DbdHost=slurmdbd`** → `fatal: This host not configured to run SlurmDBD ((skipjack-slurmdbd) !=
   slurmdbd)` → crash loop → **accounting down** (`sacctmgr … Connection refused :6819`; login nodes
   `No route to host`).

**Why:** `start.sh` (~L1160-1181) renders `DbdHost` to the branded hostname, but
`cluster_manager._ensure_shared_stack` — the path a `--from-ghcr` deploy/recreate takes **inside
coldfront** — does **not**, so a recreate leaves `DbdHost=slurmdbd` while the container hostname is branded.

**Recovery (what fixed it):**
```bash
# align DbdHost to the branded container hostname, then restart JUST slurmdbd
CONF=$(docker inspect skipjack-slurmdbd --format '{{range .Mounts}}{{if eq .Destination "/etc/slurm/slurmdbd.conf"}}{{.Source}}{{end}}{{end}}')
sed -i -E 's/^[[:space:]]*DbdHost[[:space:]]*=.*/DbdHost=skipjack-slurmdbd/' "$CONF"
docker restart skipjack-slurmdbd                               # crash loop → clean start
docker exec skipjack-slurmctld sacctmgr show cluster -P        # skipjack row = accounting back
```

**Rules for a LIVE host (do NOT repeat this):**
- **Verify first, deploy never.** If `imagetools inspect …:prod` == local digests for all images,
  you're already on `:prod` — **do nothing**. `--from-ghcr` only churns a healthy stack.
- **`slurm start` / `slurm create` are first-time only.** `slurm create` re-seeds the DB
  (`seed_initial_data.py`); `slurm start` needs `SLURM_BIN` set (+ `ARCH_ENV=prod`) or it targets
  `arch-dev` and stomps the shared core.
- **After any shared-core recreate, check the DbdHost fatal:** `docker logs skipjack-slurmdbd`
  (`scontrol` won't even show slurmdbd while it crash-loops).

### Same session — three more fixes worth remembering

- **Admin 500 on ANY allocation** (`AttributeError: 'NoneType' … 'name'`) — the cluster **Resource was
  deleted+recreated**, cascade-wiping **all 36** `allocation↔resource` M2M links → upstream
  `Allocation.__str__` derefs `get_parent_resource.name` and 500s the change page (the changelist
  survives — it formats `None` as `-`). Fix: re-attach the Resource to every orphan
  (`a.resources.add(Resource.objects.get(name='SKIPJACK'))`). **Never delete a cluster Resource that
  still has allocations.**
- **Interactive `srun --pty` / `interact` hangs** (job reaches `RUNNING`, no prompt) — `SrunPortRange=0-0`.
  `interact` = `salloc … srun --pty bash`; batch works because it never opens a back-channel to the login
  node. Fix: pin `SrunPortRange=60001-63000` in `slurm.conf.arch` (byte-identical across nodes →
  `HASH_VAL=Match`) **and** open that TCP range **INBOUND on the login node**
  (`firewall-cmd --permanent --add-port=60001-63000/tcp`; compute nodes have no firewalld). srun reads
  the range from the **login node's local** `slurm.conf`.
- **Single-file bind stale-inode** — after `cp`-ing a new `slurm.conf`/`slurmdbd.conf` onto a container's
  single-file bind mount, the container keeps reading the OLD inode; `scontrol reconfigure` sees stale
  content. **Restart the container** to re-bind (`HASH_VAL=Match` + `scontrol show config` confirm).

**Durable fixes (identified, PENDING):** (1) render `DbdHost` in `_ensure_shared_stack`; (2) add
`SrunPortRange=60001-63000` to `slurm/conf/slurm.conf.template.arch` + the login firewall port to the
Ansible `slurm_config`/login role; (3) guard `slurm start` from routing to `arch-dev` when a prefixed
prod cluster is live (or require `SLURM_BIN`).

### Rebrand core containers `arch/*` → `ghcr:prod` on a LIVE host — safe recipe

To move the running core to the GHCR-tagged images **without** the saga (same bits — `arch/*` and
`ghcr:prod` share the image ID), regen the overlay and `up -d` it — **never** `slurm start --from-ghcr`
(that re-runs `write_slurm_conf` → clobbers the served `slurm.conf`: `SrunPortRange=0-0` / `DbdHost`).
Caveat: in **prod the repo is NOT rw-mounted** into `coldfront` (base compose mounts only `./ansible`
+ `./slurm`), so `write_compose_overlay` writes an **ephemeral container path** → `docker cp` it out.

```bash
# 1. regen overlay INSIDE coldfront — pass GHCR env via -e (NOT os.environ mid-python):
docker exec -e CLOACK_FROM_GHCR=1 -e CLOACK_IMAGE_TAG=prod coldfront python -c \
 "import django,os; os.environ.setdefault('DJANGO_SETTINGS_MODULE','coldfront.config.settings'); django.setup(); \
  from coldfront.plugins.arch_sync.models import SlurmCluster; \
  from coldfront.plugins.arch_sync.cluster_manager import write_compose_overlay; \
  print(write_compose_overlay(SlurmCluster.objects.get(name='$CLUSTER'), core_only=True))"
docker cp coldfront:/opt/arch/docker-compose-slurm-$CLUSTER.yml ./docker-compose-slurm-$CLUSTER.yml
docker compose -p $CLUSTER-slurm --env-file .env -f docker-compose-slurm-$CLUSTER.yml config -q  # validate
# guard-diff vs a backup of the RUNNING overlay: ONLY image lines (arch/*→ghcr:prod) + configless
# mounts (slurm.conf :ro, cgroup→<cluster>/, +plugstack) may change. Compute node / port / container_name
# change → STOP.
docker compose -p $CLUSTER-slurm --env-file .env -f docker-compose-slurm-$CLUSTER.yml up -d   # controller blip only; no write_slurm_conf → no saga
```
