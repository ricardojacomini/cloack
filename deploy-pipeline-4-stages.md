# Deploy pipeline — 4 stages (GHCR image promotion)

> This doc lives in `docs/` (tracked) and is the canonical reference for the
> `publish` / `promote` / `start --from-ghcr` commands and the GHCR tag scheme.
> It was previously kept at `.claude/plans/deploy-pipeline-4-stages.md`, which is
> **gitignored** — so it never travelled with a fresh clone and broke the README
> link. Moved here so every stage host (Proxmox, edulogin, portal) gets it on
> `git pull`. The operator runbook with the day-to-day recipes is
> [`docs/roles/admin.md`](roles/admin.md).

## The four stages

The stack moves through 4 environments along a single pipeline. Code travels via
**git** (`origin/dev`); built images travel via **GHCR tags** under explicit
operator control. No stage past #2 ever builds a Dockerfile — it `docker pull`s.

| Stage | Host | Arch | Builds? | What it runs | Image flow |
|---|---|---|---|---|---|
| **1** Laptop coding | macOS arm64 | yes (local) | `./start.sh deploy dev -y --paid-accounts` | full dev stack + test data | `arch/<svc>` local |
| **2** dev VM build | Proxmox Linux amd64 | yes (local) | `./start.sh start dev` → `./start.sh publish` | prod-arch build, then push | `:dev` + `:sha-<short>` → GHCR |
| **3** edulogin (pre-prod) | bare-metal amd64 | no — pull | `start --from-ghcr dev` → verify → `promote` | validate `:dev`, bless to `:prod` | pull `:dev`, push `:prod` + `:rc-<UTC>` |
| **4** portal (prod) | bare-metal amd64 | no — pull | `start --from-ghcr prod` | run promoted images | pull `:prod` |

```
git origin/dev                          GHCR registry
       ↓                                    ↓
laptop ──push──> origin            Stage 2 ──push :dev, :sha-*─> GHCR
proxmox ──pull──> origin           Stage 3 ──pull :dev, push :prod, :rc-*─> GHCR
edulogin ──pull──> origin          Stage 4 ──pull :prod
portal ──pull──> origin            (portal never pushes)
```

## Tag scheme

| Tag | Created by | Mutable? | Purpose |
|---|---|---|---|
| `:sha-<git-short>` | Stage 2 `publish` | **Immutable** | Snapshot tied to a git commit. "What code is in this image?" |
| `:dev` | Stage 2 `publish` | Mutable head | Latest Stage 2 build. What edulogin pulls. |
| `:<ver>-dev`, `:<ver>-sha-<short>` | Stage 2 `publish` | `:<ver>-sha-*` immutable | **Version-pinned** dual-tags for images that carry a real upstream version (`mariadb:10.11`, `openldap:1.5.0`). Published *alongside* the bare tags so a `FROM` bump surfaces as a new tag instead of silently overwriting (gap H2). |
| `:rc-YYYY-MM-DD-HHMMSS` | Stage 3 `promote` | **Immutable** | Validated + promoted at this exact UTC second. Rollback target. Seconds granularity (M5) so two promotes in the same minute don't overwrite each other's RC tag. |
| `:prod` | Stage 3 `promote` | Mutable head | Latest validated build. What portal pulls. |

Rollback is a retag (`:prod` ← previous `:rc-*`), not a rebuild — same digest,
new label (`./start.sh promote --rollback rc-YYYY-MM-DD-HHMMSS`).

**Version-pin convention (H2/L8):** the bare channel tag (`:dev`/`:prod`) is the
deploy contract — `docker-compose-ghcr.yml` and the Slurm overlay rewrite
(`cluster_manager._GHCR_IMAGE_REWRITE_RE`) both pull it. The `:<ver>-*` tags are
an additive registry-side audit/anti-overwrite layer. `cmd_promote`'s
variant-prefix retag already maps `<ver>-<src>` → `<ver>-prod`/`<ver>-rc-*`
symmetrically if the overlay is ever switched to version-pins.

## GHCR namespace + PATs

Registry path: `ghcr.io/jhu-arch/cloack/<image>:<tag>`. The repo lives in the
`cloack` GitHub org; packages live in the `jhu-arch` org. Both PATs are
fine-grained, scoped to `jhu-arch`, SSO-authorized for JHU Enterprise, rotated
≤ 1 year.

| Stage | PAT | Operations |
|---|---|---|
| 1 (laptop) | none | git push only |
| 2 (Proxmox) | `GHCR_BUILDER_PAT` (write) | `docker push :dev` + `:sha-*` |
| 3 (edulogin) | `GHCR_BUILDER_PAT` (write) | `docker pull :dev`, `docker push :prod` + `:rc-*` |
| 4 (portal) | `GHCR_CONSUMER_PAT` (read-only) | `docker pull :prod` only |

## Image inventory (14 runtime + 1 build-only)

| Track | Images | Promoted? |
|---|---|---|
| Cloack apps (8) | `coldfront` (also `qcluster`), `helpdesk`, `keycloak`, `keycloak-config`, `openldap`, `phpldapadmin`, `postgres`, `ai-engine` | Yes — from `docker-compose-ghcr.yml` |
| Slurm runtime (6) | `slurmctld`, `slurmdbd`, `slurmrestd`, `slurm-munge`, `mariadb`, `slurmd` | Yes — `promote` PASS 1b, **best-effort** |
| Build-only (1) | `cn-rockylinux` | No — `FROM` base, never a container |

`publish` pushes all 15 (14 runtime + `cn-rockylinux`). `promote` retags the 14
runtime images `:dev`→`:prod`+`:rc`.

> **Inventory drift guard (H1):** these lists live in four places that must
> agree — `cmd_publish` mapping, `cmd_promote` `slurm_svcs`,
> `docker-compose-ghcr.yml`, and every `image: arch/<svc>` reference.
> `scripts/lint_image_inventory.py` cross-checks them (PR gate
> `.github/workflows/lint-image-inventory.yml` + a soft check in
> `verify-deploy.sh`). Add a new `arch/<svc>` without adding it to `cmd_publish`
> and the lint fails before Stage 4 can 404 on `docker pull`.
>
> **`publish --skip-base` (L6):** omit `cn-rockylinux` (build-only base, never
> pulled by Stage 3/4) to save ~500 MB/publish of GHCR storage. Opt-in — Stage 3
> then can't rebuild from the base, which it isn't meant to do anyway.
>
> **`promote --require-slurm` / `--mode-a` (H3):** turn the best-effort Slurm
> skip below into a **fatal** error. Use when Stage 4 will run **Mode A** (Docker
> Slurm) so a missing `:source-tag` Slurm image fails at promote time instead of
> when the portal can't pull it. Omit for **Mode B** (bare-metal Slurm). `ai-engine` is a thin layer over the pinned
upstream `llama.cpp` digest (weights stay bind-mounted, never baked in); it is
**best-effort** like Slurm — a builder that never ran `./start.sh ai start` has no
local `arch/ai-engine` image, so `publish` warns + skips it. The 6 Slurm images are **best-effort** in
`promote`: a deploy that never built Slurm (compute workers bare-metal, slurm
build opt-out) simply has no `:dev` Slurm tag — `promote` warns + skips instead
of aborting the cloack promotion. `slurmd` is also the **login-node** image, so
it promotes even when compute workers are excluded.

> Current target topology (2026-06): core Slurm containerized (controller +
> login + `slurmdbd`/`slurmrestd`/`mariadb`); **compute workers excluded** from
> Stage 3/4 (added later, bare-metal or Docker). When login + `mariadb`
> eventually move to bare-metal/external, the runtime set drops to 11 and the
> `promote` best-effort silently skips the two missing images — no code change.

## Slurm cluster — naming + bring-up (Stage 3/4)

`start --from-ghcr` brings up the **base stack only** (coldfront, keycloak,
postgres, openldap, helpdesk, ai-engine). It **never** starts the Slurm cluster —
the controller, login node, and compute nodes are a separate per-cluster overlay
brought up by `slurm start`. Forgetting this is the #1 "no `login01`" symptom on a
fresh Stage 3 host (the base stack is healthy, but `slurm start` was never run).

### Cluster naming convention

| Stage | Cluster name | Set where |
|---|---|---|
| 1 / 2 (laptop + Proxmox build) | `arch-dev` | dev default ([`_default_cluster`](../start.sh)) |
| 3 / 4 (edulogin + portal) | `skipjack` | explicit `slurm create skipjack` |

The name is **not** a runtime flag — `slurm start` derives `_cluster_name` from
the overlay file it finds (`docker-compose-slurm-<name>.yml`, first match
alphabetically). You set the name **once**, at `slurm create`, and keep **only
one** Slurm overlay on the host so the glob can't pick the wrong cluster:

```bash
# Stage 3/4 — create the skipjack cluster (DB record + default nodes incl.
# login01 + overlay with GHCR refs), then bring it up.
./start.sh slurm create skipjack --from-ghcr dev   # first time only
./start.sh slurm start  --from-ghcr dev            # pulls slurmd → skipjack-login01 (+ cn)
```

Container names become `skipjack-slurmctld` / `skipjack-login01` /
`skipjack-cn-01`; compose project `skipjack-slurm`; conf at
`${BASE_DATA}/slurm/skipjack/slurm.conf`; Service Health shows "Cluster:
skipjack". The shared core (`slurmdbd`/`slurmrestd`/`slurm-db`/`munge`, project
`arch-slurm-shared`) is **not** per-cluster — skipjack reuses it
(federation-ready), and `sacctmgr show cluster` registers `skipjack`.

> **Optional — brand the core containers per-cluster (`prefix_core_containers`).**
> On the hybrid skipjack path the **site DNS** maps
> `skipjack-slurmctld` / `skipjack-slurmdbd` / `skipjack-slurmrestd` → the
> ColdFront host (e.g. `172.16.1.2  openldap skipjack-slurmctld skipjack-slurmdbd
> skipjack-slurmrestd`). By default the shared core is unprefixed (`slurmdbd` /
> `slurmrestd`), so `docker ps` and that DNS disagree. Setting
> `SlurmCluster.prefix_core_containers=True` renames the two DNS-facing daemons to
> `skipjack-slurmdbd` / `skipjack-slurmrestd` in `docker ps` **and** Service
> Health, so operators and the bare-metal Ansible tooling see one consistent name.
>
> - **Mechanics (low-risk):** only `container_name` changes. The compose **service
>   keys** stay `slurmdbd`/`slurmrestd`, so `AccountingStorageHost=slurmdbd`,
>   `DbdHost`, and `slurmrestd_url` keep resolving via the network alias — **no conf
>   change, no rebuild**. The MariaDB backend stays `slurm-db` (never prefixed;
>   whitelist `cluster_manager._PREFIXABLE_CORE_SERVICES`). Default OFF, migration `0110`.
> - **Apply (bind-mount, no publish/promote):** it's a `coldfront/custom/` change, so
>   `git pull` → `docker restart coldfront` (runs migrate) →
>   `SlurmCluster.objects.filter(name='skipjack').update(prefix_core_containers=True)`
>   (or the Django admin checkbox) → `./start.sh slurm start --from-ghcr prod`
>   regenerates the overlay and recreates the core. External volumes are reused → no
>   data loss.
> - **Verify:**
>   ```bash
>   docker ps --format '{{.Names}}\t{{.Status}}' | grep -E 'slurmdbd|slurmrestd|slurm-db'
>   docker exec skipjack-slurmctld getent hosts slurmdbd     # base name still resolves via the alias
>   ```
> - **Consumer:** the bare-metal compute nodes reach the containerized core by these
>   DNS names — see [Deploy ARCH Slurm from a remote workstation](./ansible-remote-deploy.md),
>   whose `docker exec skipjack-slurmdbd …` commands assume this flag is ON.

> If an `arch-dev` overlay is also present on the host, the glob picks it first
> (`arch-dev` < `skipjack` alphabetically). Run `./start.sh slurm destroy` (or
> `rm docker-compose-slurm-arch-dev.yml`) so only the skipjack overlay remains.

### Two knobs that key off the cluster name

These two used to be hardcoded to `dev-*`. They now **auto-derive the cluster
name from the slurm overlay** present on the host (`docker-compose-slurm-<name>.yml`),
falling back to `arch-dev` when none exists — so on a Stage 3/4 host with the
`skipjack` overlay they resolve to `skipjack-login01` / `skipjack` with no
manual step. Override only if the overlay isn't the cluster you mean:

| Knob | Auto-derives to | Override (optional) |
|---|---|---|
| `NPM_SSH_STREAM_TARGET` | `<overlay-cluster>-login01` ([start.sh](../start.sh) `edge_mode_setup`) | `cloack.env` |
| verify-deploy `CLUSTER` | `<overlay-cluster>` ([scripts/verify-deploy.sh](../scripts/verify-deploy.sh)) | invocation env |

```bash
# Only needed if you must override the overlay-derived default, e.g. a host
# that has no slurm overlay yet but you want the verify gate scoped to skipjack:
export CLUSTER=skipjack                       # promote's internal verify inherits it
echo 'NPM_SSH_STREAM_TARGET=skipjack-login01' >> cloack.env   # gitignored, chmod 600
```

## Commands

```bash
# Stage 2 — build (base + Slurm) then publish
./start.sh start dev               # builds base stack (arch/<svc>)
./start.sh slurm start             # builds Slurm images (arch/slurmd …) — opt-in build!
./start.sh publish --check         # dry-run: confirm 0 "SKIP arch/slurmd" lines
./start.sh publish                 # :dev + :sha-<short> (+ :<ver>-* for mariadb/openldap); refuses on dirty tree
./start.sh publish --skip-base     # same, minus cn-rockylinux (saves GHCR storage; L6)

# Stage 3 — validate then promote   (cluster = skipjack)
export CLUSTER=skipjack
./start.sh start --from-ghcr dev            # base stack: pull :dev, no build
./start.sh slurm create skipjack --from-ghcr dev   # first time only: DB record + overlay
./start.sh slurm start  --from-ghcr dev     # Slurm: skipjack-slurmctld + login01 (+ cn)
bash scripts/verify-deploy.sh               # mandatory pre-flight gate (uses $CLUSTER)
./start.sh promote                          # :dev -> :prod + :rc-<UTC>  (runs verify first)
./start.sh promote --require-slurm          # Mode A (Docker Slurm): missing Slurm image = FATAL (H3)
./start.sh promote --check                  # dry-run, no push
./start.sh promote --rollback rc-2026-06-09-143055

# Stage 4 — run prod images          (cluster = skipjack)
export CLUSTER=skipjack
./start.sh start --from-ghcr prod           # base stack: pull :prod
./start.sh slurm create skipjack --from-ghcr prod  # first time only
./start.sh slurm start  --from-ghcr prod    # Slurm cluster
./start.sh ldap-reinit                      # manual, only if LDAP needs re-seed
```

> **Why the Stage 2 Slurm build matters:** the Slurm image build is opt-in —
> `start dev` does **not** build `arch/slurmd`. If it was never built, `publish`
> silently `SKIP`s it (it only pushes images present locally), `slurmd:dev` never
> reaches GHCR, and Stage 3's `slurm start --from-ghcr` has no login/compute
> image to pull → no `login01`. Always run `slurm start` (or `slurm rebuild`)
> before `publish`, and confirm `publish --check` shows zero `SKIP` lines for the
> Slurm images.

`deploy --from-ghcr` is **hard-refused** ([start.sh `cmd_deploy`](../start.sh)):
Stages 3/4 must never populate test data (`populate` / `slurm test-jobs` /
billing seeds are Stage 1/2 only). Use `start --from-ghcr` + manual
`ldap-reinit` and stop.

### Order: `slurm create` before the first `ldap-reinit` — now self-healing

Keep the documented order (`slurm create` → `slurm start` → *then* `ldap-reinit`).
Two things break if `ldap-reinit` runs **before** a cluster exists:

1. **Default PI projects are skipped.** `ldap-reinit` runs `seed_initial_data.py`,
   whose `seed_personal_projects()` no-ops when no active `SlurmCluster` with a
   linked `Resource` is registered — so migrated PIs/admins (e.g. `rdesouz4`) get
   no default project, silently.
2. **cn=admin / ext-* miss the Mobile Authenticator (TOTP) prompt.** The JHU realm
   is `CONFIGURE_TOTP defaultAction=false`; the "Mobile Authenticator Setup" screen
   is attached **only** to the ROPC population (LDAP `cn=admin` + `ext-*`) by
   `enforce_ropc_totp_enrolment` ([configure-keycloak.sh](../keycloak/configure-keycloak.sh)),
   and only against users **already federated** into Keycloak. If `keycloak-config`
   ran while LDAP was still empty, the reconcile saw nobody.

Both are now self-healing in [start.sh](../start.sh) (no manual recovery needed if
you ever run out of order):

- **`slurm create`** re-runs the project seed at the end, so default projects are
  created regardless of whether `ldap-reinit` ran first.
- **`ldap-reinit`** now re-runs `sync-ldap-federation.sh` + restarts
  `keycloak-config` so freshly-imported `cn=admin`/`ext-*` users get the TOTP
  action, and **warns loudly** if it seeded with no active cluster.

Manual recovery on a host already in the bad state (cluster now exists):

```bash
docker exec -e SEED_ON_RESTART=true coldfront python /opt/coldfront/init/seed_initial_data.py
docker exec coldfront coldfront sync_slurm          # provision scavenger accounts
bash keycloak/sync-ldap-federation.sh               # federate users + cn=admin group
docker restart keycloak-config                      # re-run TOTP reconcile (cn=admin/ext-*)
```

### Stage 3/4 never falls back to `arch/<svc>` images

`$DC` only layers `docker-compose-ghcr.yml` when the **current** invocation
passed `--from-ghcr`. Maintenance commands that don't take that flag (`update`,
the `keycloak` recovery subcommand) used to resolve the base-compose
`arch/<svc>` images and fail with `pull access denied for arch/keycloak-config`
on a GHCR host. They now call `_autodetect_ghcr_mode` ([start.sh](../start.sh)),
which inspects the running `coldfront`/`keycloak` container and re-applies the
ghcr overlay when the stack is `ghcr.io/jhu-arch/cloack/*` (no-op on local build
hosts). The throwaway `sync-ldap-federation.sh` run likewise derives the
keycloak-config image from the live stack instead of hardcoding `arch/`.

### Rebrand the Slurm core to GHCR on a live host

The Slurm **core** (`{cluster}-slurmctld/dbd/restd` + `slurm-db`) is generated by
`cluster_manager`, not by `docker-compose-ghcr.yml`, so it does not auto-follow
the main stack onto GHCR. To rename it `arch/*` → `ghcr.io/jhu-arch/cloack/*:prod`
on a **running** host — they are the **same image ID** (`publish`/`promote`
retagged the same build), so this is a cosmetic tag swap with zero behavioral
change:

> **Never** `./start.sh slurm start --from-ghcr` on a live host — it re-runs
> `write_slurm_conf` and clobbers the served `slurm.conf` (`SrunPortRange=0-0`,
> `DbdHost` breakage — the 2026-07-06 saga). The runbook below calls **only**
> `write_compose_overlay`, so the mounted `slurm.conf` stays byte-identical.

> **Caveat — the prod repo is not rw-mounted into `coldfront`.** Base
> [docker-compose.yml](../docker-compose.yml) mounts only `./ansible` + `./slurm`
> under `/opt/arch` (the full `.:/opt/arch:rw` mount is dev-only). So
> `write_compose_overlay` writes the overlay to an **ephemeral container path** —
> the host file never changes. `docker cp` it out.

```bash
cd /opt/cloack
cp docker-compose-slurm-skipjack.yml docker-compose-slurm-skipjack.yml.bak    # rollback

# 1. regen via coldfront — GHCR env MUST come via `docker exec -e`
#    (os.environ set mid-python inside `coldfront shell -c` does NOT apply)
docker exec -e CLOACK_FROM_GHCR=1 -e CLOACK_IMAGE_TAG=prod coldfront python -c "
import django, os
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'coldfront.config.settings'); django.setup()
from coldfront.plugins.arch_sync.models import SlurmCluster
from coldfront.plugins.arch_sync.cluster_manager import write_compose_overlay
print(write_compose_overlay(SlurmCluster.objects.get(name='skipjack'), core_only=True))"

# 2. materialize on the host + validate the YAML BEFORE it touches containers
docker cp coldfront:/opt/arch/docker-compose-slurm-skipjack.yml ./docker-compose-slurm-skipjack.yml
docker compose -p skipjack-slurm --env-file .env -f docker-compose-slurm-skipjack.yml config -q

# 3. guard-diff: ONLY `image:` lines + configless mounts may change —
#    STOP if a compute node / port / container_name moves
diff docker-compose-slurm-skipjack.yml.bak docker-compose-slurm-skipjack.yml

# 4. recreate the core with ghcr (controller-recreate blip only; no saga)
docker compose -p skipjack-slurm --env-file .env -f docker-compose-slurm-skipjack.yml up -d
```

Under `prefix_core_containers` the controller is published on host port **6817**
(not the `6817+(pk-1)` offset, which would publish e.g. `6818` for `pk=2` — a port
the bare-metal `slurmd`/`sackd` fixed on `6817` can never reach → nodes stuck
Down / configless "connect failure"). `up -d` is idempotent: with the overlay file
**and** `.env` unchanged it is a no-op (unchanged containers stay `Running`; only
the one-shot `munge-init` re-runs); preview with `up -d --dry-run`.

## Code vs data — two separate channels

A container image carries **code, not data**. Pulling images never moves
datasets. Three distinct layers:

| Layer | Where it lives | How it reaches Stage 3/4 |
|---|---|---|
| App code + **migration files** (`*/migrations/*.py`) | inside the image | `docker pull` (built once on Stage 2) |
| **Schema** (applied migrations) | Postgres data dir | created/updated at container boot by `migrate` |
| **Data** (rows, LDAP, `/home`, `/scratch`) | host volumes (`${BASE_DATA}/postgresql`, …) | **never in the image** — persists on the host |

On every coldfront boot, [`entrypoint.sh`](../coldfront/entrypoint.sh) runs
`migrate --noinput` (+ `--database=slurm`) against the **existing persistent
DB**, applying only the pending migrations. Schema evolves incrementally; data
is preserved. Real prod data arrives via the LDIF migration import
(`ldap-reinit` / Import Users) and live usage — never via the dev seeders.

## Data safety — `pg_dump` before `migrate` (Stage 3/4)

**The risk.** Because `migrate` runs automatically on boot, a bad migration baked
into an image **auto-applies to the prod DB** the moment portal pulls it. Image
rollback ≠ DB rollback: retagging `:prod` ← a previous `:rc-*` restores the **old
code**, but the schema is already migrated forward and Django has no automatic
downgrade for data migrations (`migrate <app> <prev>` only reverses if reverse
ops are defined, and dropped data is gone). The `:rc-*` tag protects **code**
lineage, not the **database**.

**Automatic snapshot (implemented).** `cmd_start` calls
[`_backup_before_update()`](../start.sh) — `pg_dumpall` + `slapcat` →
`${BASE_DATA}/backup/` — **before** `compose_up_scoped` recreates coldfront,
whenever `CLOACK_FROM_GHCR=1` (i.e. on every `start --from-ghcr`, so both
edulogin and portal). It runs after Postgres is confirmed up but before the new
image's boot-time `migrate`, giving a point-in-time snapshot to restore from if a
migration corrupts data. Best-effort: on a truly fresh start (no running
Postgres) the helper warns and skips — nothing to lose. dev/Proxmox local builds
never set `CLOACK_FROM_GHCR`, so they are unaffected (the laptop's own
`./start.sh update` path still backs up independently).

Stage 3 (edulogin) remains the primary gate — it migrates pre-prod data first
and `verify-deploy.sh` must pass before `promote` — but it is only meaningful if
edulogin's DB resembles prod's. Restore from a snapshot with
`docker exec -i postgres psql -U <admin> < ${BASE_DATA}/backup/postgres_<ts>.sql`.

## Stage 4 first-run vs reset semantics — `init` / `--regen-secrets` / `clean-all`

`init` writes `.env`. Its 14 secrets (DB passwords, LDAP binds, Keycloak admin,
Django keys) are **preserved across reruns** by default (gap M4 fix): `write_env`
resolves each from the `.env.<UTC>` backup `init` makes before rewriting, so
re-running `init` on a host with **populated DB volumes** does **not** rotate the
passwords those volumes were initialised with. Rotating them blindly used to
break auth — `slurmdbd` "1045 Access denied", Keycloak "did not become ready",
Postgres/OpenLDAP bind failures — because the new random `.env` no longer matched
the values baked into the volumes on first boot.

| Operator intent | Command | Effect on secrets | Effect on data |
|---|---|---|---|
| First install | `init` | Generate fresh (no prior `.env`) | Fresh volumes |
| Re-run / pick up new keys | `init` | **Preserve** existing | Data intact |
| Deliberate secret rotation | `init --regen-secrets` then `clean-all` | **Rotate all** | **Volumes wiped** (must, or they reject new secrets) |
| Full reset | `clean-all` then `init` | Fresh (no `.env` after wipe) | Everything wiped |

`init` is therefore idempotent and safe to re-run in prod by default (honors
[feedback_no_direct_db_patches] — a fresh deploy needs no manual steps).
`--regen-secrets` is the explicit "I want new secrets" escape hatch and **warns
it is destructive**; always pair it with `clean-all` so volumes re-init against
the rotated secrets. The dev `deploy` runs `clean --all` then `init` (no
`--regen-secrets`), so fresh volumes simply adopt the preserved `.env` secrets —
the invariant *(.env secrets == volume secrets)* always holds.

## Config promotion gap (open item)

Images are promoted via **immutable tags**; git **configs/templates are not** —
every stage does `git pull origin dev`, so a half-finished config change on `dev`
can reach portal on the next pull even with the image pinned at `:prod`. A
separate pre-prod **repo** is overkill (mirror/drift burden). The aligned fix is
a **branch**: advance `main` in the `promote` step (`git push origin <sha>:main`)
and have portal `git pull origin main` instead of `dev`, so config promotion
mirrors image promotion. Not yet implemented.

## Overlay change runbook — `settings.py` / templates / CSS (per stage)

**When this applies.** A change to a **runtime overlay** file — anything under
`helpdesk/custom/` (e.g. `config/settings.py`, templates, CSS) or
`coldfront/custom/` — that is **bind-mounted** into the container at startup, not
baked into the read path of the image. These travel on the **git channel** (the
"Config promotion gap" above), never the GHCR image channel. The bind-mount
(`${BASE_DATA}/helpdesk/custom`, staged from the host repo by
`stage_helpdesk_overlays`) **shadows** whatever the image baked, so the image tag
is irrelevant to the change — no rebuild is ever required.

> This is **not** for image-baked changes (Python deps, `django-helpdesk`
> version, Dockerfile, base code). Those go through Stage 2 `build` + `publish`
> and Stage 3/4 `--from-ghcr` (pull) like any other image change.

**What to do, per stage.** Restart belongs on the stages that *serve* the app
(3 and 4), per host, via its own `git pull`. Stage 2 only relays the **image**
via GHCR — it does **not** relay overlays, so a `restart` there accomplishes
nothing for an overlay change.

```bash
# Stage 3 (edulogin) — the running pre-prod host
git pull && ./start.sh helpdesk restart

# Stage 4 (portal) — repeat on the prod host itself
git pull && ./start.sh helpdesk restart

# Stage 2 (dev VM) — NOTHING, unless you also want the GHCR image to bake the fix:
git pull && ./start.sh start dev && ./start.sh publish   # build + publish, NEVER restart
```

`./start.sh helpdesk restart` runs `stage_helpdesk_overlays` (rsync repo →
`${BASE_DATA}/helpdesk/custom`) then `docker compose restart` — **no build, no
pull**, and it keeps whatever image the container already runs. Use
`./start.sh helpdesk start --from-ghcr` instead only when you also need to
(re)pull the GHCR image (e.g. the container drifted onto a local
`arch/helpdesk:latest` build and must be returned to `ghcr.io/.../helpdesk:dev`);
it recreates on the pulled image and re-stages the overlay in the same step.

## Phase 2 — multi-FQDN / per-realm public hostnames

A later phase of this design covers per-realm public hostnames and subpath
mounts (so JHU / Schmidt / helpdesk can each present a distinct FQDN behind a
reverse proxy). That work is implemented — not planned — and the source is the
authority:

- `reconcile_public_host()` in [start.sh](../start.sh) runs on every
  `init/start/deploy/restart/rebuild`; `./start.sh doctor` validates the result.
- [scripts/detect_public_host.py](../scripts/detect_public_host.py) — multi-FQDN detection.
- [keycloak/configure-keycloak.sh](../keycloak/configure-keycloak.sh) — multi-FQDN reverse-proxy + per-realm hostname config (multi-URI safety list on the realm).
- Helpdesk subpath mount via `HELPDESK_SCRIPT_NAME` ([helpdesk/custom/config/settings.py](../helpdesk/custom/config/settings.py)).

### Repoint a stale public host

When `ARCH_PUBLIC_HOST` is pinned to the wrong value (e.g. an internal
`socket.getfqdn()` FQDN that `detect_public_host.py` keeps returning as
`[kept]`), force-re-derive every OIDC var + theme with the wrapper, then
reconcile so the live Keycloak (realm `frontendUrl` + client redirectUris)
picks it up:

```bash
./scripts/fix_public_host.sh mgmt02.mgmt.ai.cluster --dry-run   # review
./scripts/fix_public_host.sh mgmt02.mgmt.ai.cluster             # apply to files
# then, when ready: ./start.sh init prod && ./start.sh start prod  (reconcile)
```

`fix_public_host.sh` wraps `detect_public_host.py --force --default-host <host>`
(the only way to overwrite an operator-pinned value), backs up the touched env
files, and scans for leftover old-host references. `--apply <mode>` runs the
reconcile for you.

See the memory note `project_reconcile_public_host` for the drift-detection
(`.host-applied`) and 14-check `doctor` details.
