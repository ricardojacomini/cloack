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
- Inbound-email follow-ups auto-classify as **Communication (e-mail) - user's reply** (`note_kind=user_reply`) — distinct from staff outbound comms (`customer_reply`, which carry FQR/LQR), and never Technical Note.

## Top gotchas

- **Slurm build is opt-in** — `start dev` does NOT build `arch/slurmd`. Run `slurm start` before `publish`, and confirm `publish --check` shows zero `SKIP` for Slurm images, or Stage 3 has no `login01` to pull.
- **Dirty tree blocks publish** on the `:dev` tag (so `:sha-*` can't lie). Commit or clean first.
- **Cluster name = `skipjack`** on Stage 3/4, set once at `slurm create`; keep only one `docker-compose-slurm-*.yml` on the host (glob picks alphabetically).
- **`start --from-ghcr` brings up the base stack only** — Slurm is a separate `slurm start`. "No login01" on a fresh host = you forgot `slurm start`.
- **`_autodetect_ghcr_mode`** keeps later `update`/`keycloak` commands on GHCR images (no fallback to `arch/<svc>`).
